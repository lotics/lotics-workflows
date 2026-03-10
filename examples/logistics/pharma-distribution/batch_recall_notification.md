# Batch Recall Notification and Quarantine

## The problem

When a pharmaceutical manufacturer issues a batch recall, the distributor must identify every customer who received product from that batch, quarantine remaining stock, and notify all affected parties — within hours, not days. GDP (Good Distribution Practice) regulations require full traceability and documented notification. A distributor handling 5,000+ SKUs across 200+ customers cannot manually trace batch distribution: it takes 2-3 days to compile the affected customer list from shipping records, by which time recalled product may already have been dispensed to patients.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Recalls | recall_id, product_sku, batch_number, reason, severity, manufacturer_notice, status | Recall events |
| Inventory Batches | sku, batch_number, qty_on_hand, location, status | Current stock |
| Distribution History | batch_number, sku, customer_id, qty_shipped, ship_date, invoice_number | Who received what |
| Customers | name, email, pharmacy_license, type | Pharmacies, hospitals, clinics |

## Example prompts

- "When a recall record is created, quarantine all matching inventory, find every customer who received that batch, and send recall notices."
- "Automatically execute the recall protocol — quarantine stock, trace distribution, and notify affected customers."

## Workflow

**Trigger:** New recall record created in the Recalls table.

```json
[
  {
    "id": "get_recall",
    "type": "tool_call",
    "description": "Fetch the recall record details",
    "tool_name": "get_record",
    "input": {
      "table_id": "recalls",
      "record_id": "{{trigger.record_updated.record_id}}"
    }
  },
  {
    "id": "find_affected_inventory",
    "type": "tool_call",
    "description": "Find all inventory of the recalled batch still in stock",
    "tool_name": "query_records",
    "input": {
      "table_id": "inventory_batches",
      "filter": {
        "and": [
          { "field": "sku", "operator": "equals", "value": "{{get_recall.output.data.product_sku}}" },
          { "field": "batch_number", "operator": "equals", "value": "{{get_recall.output.data.batch_number}}" },
          { "field": "qty_on_hand", "operator": "greater_than", "value": 0 }
        ]
      }
    }
  },
  {
    "id": "quarantine_inventory",
    "type": "foreach",
    "description": "Quarantine all affected inventory batches",
    "items": "{{find_affected_inventory.output.data}}",
    "steps": [
      {
        "id": "quarantine_batch",
        "type": "tool_call",
        "description": "Set batch status to quarantined and lock the record",
        "tool_name": "update_record",
        "input": {
          "table_id": "inventory_batches",
          "record_id": "{{quarantine_inventory.item.id}}",
          "data": {
            "status": "quarantined"
          }
        }
      },
      {
        "id": "lock_batch",
        "type": "tool_call",
        "description": "Lock the quarantined batch to prevent any movement",
        "tool_name": "lock_record",
        "input": {
          "table_id": "inventory_batches",
          "record_id": "{{quarantine_inventory.item.id}}",
          "reason": "Recall {{get_recall.output.data.recall_id}} — batch quarantined"
        }
      }
    ]
  },
  {
    "id": "trace_distribution",
    "type": "tool_call",
    "description": "Find all customers who received product from the recalled batch",
    "tool_name": "query_records",
    "input": {
      "table_id": "distribution_history",
      "filter": {
        "and": [
          { "field": "sku", "operator": "equals", "value": "{{get_recall.output.data.product_sku}}" },
          { "field": "batch_number", "operator": "equals", "value": "{{get_recall.output.data.batch_number}}" }
        ]
      }
    }
  },
  {
    "id": "notify_customers",
    "type": "foreach",
    "description": "Send recall notification to each affected customer",
    "items": "{{trace_distribution.output.data}}",
    "steps": [
      {
        "id": "get_customer",
        "type": "tool_call",
        "description": "Fetch customer contact details",
        "tool_name": "get_record",
        "input": {
          "table_id": "customers",
          "record_id": "{{notify_customers.item.customer_id}}"
        }
      },
      {
        "id": "send_recall_notice",
        "type": "tool_call",
        "description": "Email the formal recall notice to the customer",
        "tool_name": "gmail_send_email",
        "input": {
          "to": "{{get_customer.output.data.email}}",
          "subject": "URGENT RECALL: {{get_recall.output.data.product_sku}} Batch {{get_recall.output.data.batch_number}}",
          "body": "Dear {{get_customer.output.data.name}},\n\nThis is an urgent product recall notification.\n\nProduct: {{get_recall.output.data.product_sku}}\nBatch: {{get_recall.output.data.batch_number}}\nReason: {{get_recall.output.data.reason}}\nSeverity: {{get_recall.output.data.severity}}\n\nOur records show you received {{notify_customers.item.qty_shipped}} units on {{notify_customers.item.ship_date}} (Invoice: {{notify_customers.item.invoice_number}}).\n\nPlease immediately quarantine any remaining stock from this batch and do not dispense. Contact us to arrange return collection.\n\nThis notification is issued in compliance with GDP requirements."
        }
      },
      {
        "id": "log_notification",
        "type": "tool_call",
        "description": "Record that this customer was notified for audit trail",
        "tool_name": "create_record_comments",
        "input": {
          "table_id": "recalls",
          "record_id": "{{get_recall.output.data.id}}",
          "comment": "Recall notice sent to {{get_customer.output.data.name}} ({{get_customer.output.data.email}}). Qty affected: {{notify_customers.item.qty_shipped}}. Ship date: {{notify_customers.item.ship_date}}."
        }
      }
    ]
  },
  {
    "id": "update_recall_status",
    "type": "tool_call",
    "description": "Update the recall record with tracing completion details",
    "tool_name": "update_record",
    "input": {
      "table_id": "recalls",
      "record_id": "{{get_recall.output.data.id}}",
      "data": {
        "status": "notices_sent",
        "customers_notified": "{{trace_distribution.output.data.length}}",
        "qty_quarantined": "{{find_affected_inventory.output.data.length}}"
      }
    }
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| Time to complete recall trace | 2-3 days | Under 30 minutes |
| Customer notification time | 3-5 days | Under 1 hour |
| GDP compliance audit findings | 2-4 per audit | 0 |

→ [Set up this workflow on Lotics](https://lotics.ai)

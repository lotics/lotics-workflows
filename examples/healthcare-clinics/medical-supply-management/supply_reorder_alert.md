# Low Stock Reorder Alert

## The problem

Medical clinics consume 200-400 supply SKUs daily — gloves, syringes, gauze, test kits. When stock dips below safe levels, staff discover shortages mid-procedure. Manual weekly inventory counts miss fast-moving items, and emergency orders cost 30-50% more than standard procurement. A single stockout of PPE or blood draw supplies can delay patient care for hours.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Supplies | item_name, sku, category, current_quantity, reorder_threshold, reorder_quantity, unit_cost, supplier, last_ordered | Inventory of all medical supplies |
| Purchase Orders | supplier, items, total_cost, status, order_date, expected_delivery | Tracks orders placed with suppliers |
| Suppliers | name, email, lead_time_days, payment_terms | Supplier contact and terms |

## Example prompts

- "When a supply item's quantity drops below its reorder threshold, automatically create a purchase order for that supplier and email them the order details."
- "Set up an alert that checks supply levels every time inventory is updated. If anything is running low, generate a PO and notify the office manager."

## Workflow

**Trigger:** `record_updated` — Fires when a supply record's current_quantity changes.

```json
[
  {
    "id": "check_threshold",
    "type": "if",
    "description": "Check if the updated quantity has dropped below the reorder threshold",
    "condition": "{{trigger.record_updated.next_data.current_quantity < trigger.record_updated.next_data.reorder_threshold}}",
    "then": [
      {
        "id": "get_supplier",
        "type": "tool_call",
        "description": "Look up the supplier details for this item",
        "tool_name": "query_records",
        "input": {
          "table_name": "Suppliers",
          "filters": {
            "name": "{{trigger.record_updated.next_data.supplier}}"
          }
        }
      },
      {
        "id": "create_po",
        "type": "tool_call",
        "description": "Create a purchase order for the reorder quantity",
        "tool_name": "create_record",
        "input": {
          "table_name": "Purchase Orders",
          "data": {
            "supplier": "{{trigger.record_updated.next_data.supplier}}",
            "items": "{{trigger.record_updated.next_data.item_name}} (SKU: {{trigger.record_updated.next_data.sku}}) x {{trigger.record_updated.next_data.reorder_quantity}}",
            "total_cost": "{{trigger.record_updated.next_data.unit_cost * trigger.record_updated.next_data.reorder_quantity}}",
            "status": "Pending",
            "order_date": "{{new Date().toISOString().split('T')[0]}}",
            "expected_delivery": "{{new Date(Date.now() + get_supplier.output.records[0].lead_time_days * 86400000).toISOString().split('T')[0]}}"
          }
        }
      },
      {
        "id": "email_supplier",
        "type": "tool_call",
        "description": "Send the purchase order to the supplier via email",
        "tool_name": "gmail_send_email",
        "input": {
          "to": "{{get_supplier.output.records[0].email}}",
          "subject": "Purchase Order - {{trigger.record_updated.next_data.item_name}}",
          "body": "Hello,\n\nWe would like to place the following order:\n\nItem: {{trigger.record_updated.next_data.item_name}}\nSKU: {{trigger.record_updated.next_data.sku}}\nQuantity: {{trigger.record_updated.next_data.reorder_quantity}}\nUnit Cost: ${{trigger.record_updated.next_data.unit_cost}}\nTotal: ${{trigger.record_updated.next_data.unit_cost * trigger.record_updated.next_data.reorder_quantity}}\n\nPlease confirm availability and expected ship date.\n\nThank you."
        }
      },
      {
        "id": "update_last_ordered",
        "type": "tool_call",
        "description": "Update the supply record with today as the last ordered date",
        "tool_name": "update_record",
        "input": {
          "table_name": "Supplies",
          "record_id": "{{trigger.record_updated.record_id}}",
          "data": {
            "last_ordered": "{{new Date().toISOString().split('T')[0]}}"
          }
        }
      },
      {
        "id": "notify_manager",
        "type": "tool_call",
        "description": "Alert the office manager about the automatic reorder",
        "tool_name": "send_notification",
        "input": {
          "message": "Auto-reorder triggered: {{trigger.record_updated.next_data.item_name}} dropped to {{trigger.record_updated.next_data.current_quantity}} units (threshold: {{trigger.record_updated.next_data.reorder_threshold}}). PO created for {{trigger.record_updated.next_data.reorder_quantity}} units from {{trigger.record_updated.next_data.supplier}}.",
          "channel": "office-manager"
        }
      }
    ],
    "else": []
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| Stockout incidents per month | 6-8 | 0-1 |
| Emergency order premium spend | $1,200/month | $150/month |
| Staff time on manual inventory checks | 4 hrs/week | 0.5 hrs/week |
| Procedure delays from missing supplies | 3-4/month | 0 |

-> [Set up this workflow on Lotics](https://lotics.ai)

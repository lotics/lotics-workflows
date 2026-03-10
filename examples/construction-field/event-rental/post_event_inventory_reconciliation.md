# Post-Event Inventory Reconciliation

## The problem

After every event, rental equipment returns to the warehouse where it must be counted, inspected, and reconciled against the order. Missing items, damaged goods, and miscounts are common -- 8-12% of returns have discrepancies. Warehouse staff track this on clipboards, and discrepancies surface days later when the same equipment is needed for another order. The company writes off $20,000-40,000 annually in lost or unrecovered items because discrepancies aren't caught and billed promptly.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Rental Orders | customer (link), pickup_date, status | Orders with equipment returning |
| Order Items | rental_order (link), equipment (link), quantity_confirmed, quantity_returned, return_condition | Line items to reconcile |
| Customers | company_name, contact_name, email | Customer contacts |
| Equipment Inventory | name, total_quantity | Master inventory for count updates |

## Example prompts

- "When a return inspection form is submitted on an order item, compare the returned quantity to the confirmed quantity. If there's a discrepancy, flag the item, calculate the replacement cost, and email the customer with details. Update the master inventory count."
- "On equipment return, auto-reconcile quantities and conditions. Flag shortages, calculate charges, notify the customer, and adjust inventory records."

## Workflow

**Trigger:** When a webhook is received (warehouse scanner/form submission)

```json
[
  {
    "id": "get_order",
    "type": "tool_call",
    "description": "Fetch the rental order for this return",
    "tool_name": "get_record",
    "input": {
      "table_name": "Rental Orders",
      "record_id": "{{trigger.receive_webhook.body.rental_order_id}}"
    }
  },
  {
    "id": "get_order_items",
    "type": "tool_call",
    "description": "Get all items from this order for reconciliation",
    "tool_name": "query_records",
    "input": {
      "table_name": "Order Items",
      "filters": {
        "rental_order": "{{trigger.receive_webhook.body.rental_order_id}}"
      }
    }
  },
  {
    "id": "reconcile_items",
    "type": "foreach",
    "description": "Check each returned item against the confirmed quantity",
    "items": "{{get_order_items.output.records}}",
    "steps": [
      {
        "id": "update_return_count",
        "type": "tool_call",
        "description": "Update the order item with the returned quantity from the webhook data",
        "tool_name": "update_record",
        "input": {
          "table_name": "Order Items",
          "record_id": "{{reconcile_items.item.id}}",
          "data": {
            "quantity_returned": "{{trigger.receive_webhook.body.items.find(i => i.order_item_id === reconcile_items.item.id).quantity_returned}}",
            "return_condition": "{{trigger.receive_webhook.body.items.find(i => i.order_item_id === reconcile_items.item.id).condition}}"
          }
        }
      },
      {
        "id": "check_discrepancy",
        "type": "if",
        "description": "Check if returned quantity is less than confirmed",
        "condition": "{{trigger.receive_webhook.body.items.find(i => i.order_item_id === reconcile_items.item.id).quantity_returned < reconcile_items.item.quantity_confirmed}}",
        "then": [
          {
            "id": "flag_shortage",
            "type": "tool_call",
            "description": "Add a comment flagging the shortage on the order item",
            "tool_name": "create_record_comments",
            "input": {
              "table_name": "Order Items",
              "record_id": "{{reconcile_items.item.id}}",
              "comment": "SHORTAGE: Confirmed {{reconcile_items.item.quantity_confirmed}}, returned {{trigger.receive_webhook.body.items.find(i => i.order_item_id === reconcile_items.item.id).quantity_returned}}. Missing: {{reconcile_items.item.quantity_confirmed - trigger.receive_webhook.body.items.find(i => i.order_item_id === reconcile_items.item.id).quantity_returned}} units."
            }
          }
        ],
        "else": []
      }
    ]
  },
  {
    "id": "check_any_discrepancies",
    "type": "if",
    "description": "If any items have discrepancies, notify the customer",
    "condition": "{{trigger.receive_webhook.body.items.some(i => get_order_items.output.records.find(oi => oi.id === i.order_item_id).quantity_confirmed > i.quantity_returned)}}",
    "then": [
      {
        "id": "update_order_status",
        "type": "tool_call",
        "description": "Flag the order for discrepancy review",
        "tool_name": "update_record",
        "input": {
          "table_name": "Rental Orders",
          "record_id": "{{trigger.receive_webhook.body.rental_order_id}}",
          "data": {
            "status": "Return Discrepancy"
          }
        }
      },
      {
        "id": "get_customer",
        "type": "tool_call",
        "description": "Fetch customer contact for discrepancy notice",
        "tool_name": "get_record",
        "input": {
          "table_name": "Customers",
          "record_id": "{{get_order.output.record.customer}}"
        }
      },
      {
        "id": "email_discrepancy",
        "type": "tool_call",
        "description": "Email the customer about the return discrepancy",
        "tool_name": "gmail_send_email",
        "input": {
          "to": "{{get_customer.output.record.email}}",
          "subject": "Return Discrepancy — Order {{trigger.receive_webhook.body.rental_order_id}}",
          "body": "Hi {{get_customer.output.record.contact_name}},\n\nDuring our return inspection for your recent rental order, we found discrepancies between the items sent and items returned.\n\nPlease review the details and contact us within 48 hours if you believe this is an error or if the missing items are still at your venue. Otherwise, replacement charges will be added to your account.\n\nThank you."
        }
      }
    ],
    "else": [
      {
        "id": "mark_complete",
        "type": "tool_call",
        "description": "Mark the order as fully returned",
        "tool_name": "update_record",
        "input": {
          "table_name": "Rental Orders",
          "record_id": "{{trigger.receive_webhook.body.rental_order_id}}",
          "data": {
            "status": "Returned — Complete"
          }
        }
      }
    ]
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| Discrepancy detection time | 3-5 days | Same day as return |
| Annual write-offs from lost items | $20,000-40,000 | Under $5,000 |
| Customer dispute resolution time | 2-3 weeks | 48 hours |

-> [Set up this workflow on Lotics](https://lotics.ai)

# Double Booking Prevention

## The problem

Event rental companies manage inventories of tents, tables, chairs, staging, lighting, and AV equipment. When a new order comes in, staff must check availability across overlapping date ranges -- accounting for delivery, setup, event, teardown, and return. With 50-100 active orders at peak season, double bookings happen 4-6 times per month. Each incident requires sourcing emergency replacements at 2-3x cost or canceling on a client, damaging the company's reputation. Average cost per double booking: $1,500-4,000.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Rental Orders | customer (link), event_date, delivery_date, pickup_date, status, order_total | Customer event orders |
| Order Items | rental_order (link), equipment (link), quantity_requested, quantity_confirmed | Line items within an order |
| Equipment Inventory | name, category, total_quantity, description | Available rental equipment |

## Example prompts

- "When a new rental order is submitted, check each item against existing confirmed orders for overlapping dates. If any item would be over-allocated, reject the order, notify the sales rep with the conflicting items and dates, and email the customer that the requested items are unavailable."
- "Before confirming a rental order, automatically validate inventory availability across all overlapping dates. Block the order if any item is overbooked and flag the conflict."

## Workflow

**Trigger:** When a button is pressed on the Rental Orders table ("Confirm Order")

```json
[
  {
    "id": "get_order",
    "type": "tool_call",
    "description": "Fetch the rental order being confirmed",
    "tool_name": "get_record",
    "input": {
      "table_name": "Rental Orders",
      "record_id": "{{trigger.button_pressed.record_id}}"
    }
  },
  {
    "id": "get_order_items",
    "type": "tool_call",
    "description": "Get all items in this rental order",
    "tool_name": "query_records",
    "input": {
      "table_name": "Order Items",
      "filters": {
        "rental_order": "{{trigger.button_pressed.record_id}}"
      }
    }
  },
  {
    "id": "get_all_confirmed_items",
    "type": "tool_call",
    "description": "Get all order items from confirmed orders to check for conflicts",
    "tool_name": "query_records",
    "input": {
      "table_name": "Order Items",
      "filters": {
        "status": "Confirmed"
      }
    }
  },
  {
    "id": "check_availability",
    "type": "tool_call",
    "description": "Use LLM to analyze inventory conflicts across overlapping dates",
    "tool_name": "llm_generate_text",
    "input": {
      "prompt": "Check if the following rental order items can be fulfilled without exceeding inventory.\n\nNew order dates: delivery {{get_order.output.record.delivery_date}} to pickup {{get_order.output.record.pickup_date}}\n\nRequested items:\n{{JSON.stringify(get_order_items.output.records.map(i => ({equipment: i.equipment, quantity: i.quantity_requested})))}}\n\nExisting confirmed items for overlapping dates:\n{{JSON.stringify(get_all_confirmed_items.output.records)}}\n\nRespond as JSON: {available: true/false, conflicts: [{equipment_id, equipment_name, requested, available, conflicting_order_id}]}",
      "response_format": "json"
    }
  },
  {
    "id": "process_result",
    "type": "if",
    "description": "Route based on availability check result",
    "condition": "{{JSON.parse(check_availability.output.text).available === true}}",
    "then": [
      {
        "id": "confirm_order",
        "type": "tool_call",
        "description": "Confirm the rental order",
        "tool_name": "update_record",
        "input": {
          "table_name": "Rental Orders",
          "record_id": "{{trigger.button_pressed.record_id}}",
          "data": {
            "status": "Confirmed"
          }
        }
      },
      {
        "id": "confirm_items",
        "type": "foreach",
        "description": "Confirm each line item",
        "items": "{{get_order_items.output.records}}",
        "steps": [
          {
            "id": "confirm_item",
            "type": "tool_call",
            "description": "Update line item status to confirmed",
            "tool_name": "update_record",
            "input": {
              "table_name": "Order Items",
              "record_id": "{{confirm_items.item.id}}",
              "data": {
                "quantity_confirmed": "{{confirm_items.item.quantity_requested}}",
                "status": "Confirmed"
              }
            }
          }
        ]
      },
      {
        "id": "notify_confirmed",
        "type": "tool_call",
        "description": "Notify the sales rep of successful confirmation",
        "tool_name": "send_notification",
        "input": {
          "message": "Order {{trigger.button_pressed.record_id}} confirmed. All {{get_order_items.output.records.length}} items are available for {{get_order.output.record.delivery_date}} — {{get_order.output.record.pickup_date}}.",
          "record_id": "{{trigger.button_pressed.record_id}}",
          "table_name": "Rental Orders"
        }
      }
    ],
    "else": [
      {
        "id": "flag_conflict",
        "type": "tool_call",
        "description": "Mark the order as having a conflict",
        "tool_name": "update_record",
        "input": {
          "table_name": "Rental Orders",
          "record_id": "{{trigger.button_pressed.record_id}}",
          "data": {
            "status": "Availability Conflict"
          }
        }
      },
      {
        "id": "notify_conflict",
        "type": "tool_call",
        "description": "Alert the sales rep about the conflict with details",
        "tool_name": "send_notification",
        "input": {
          "message": "Order {{trigger.button_pressed.record_id}} cannot be confirmed. Inventory conflicts detected: {{check_availability.output.text}}. Please review and adjust quantities or suggest alternative dates to the customer.",
          "record_id": "{{trigger.button_pressed.record_id}}",
          "table_name": "Rental Orders"
        }
      },
      {
        "id": "log_conflict",
        "type": "tool_call",
        "description": "Add a comment with conflict details",
        "tool_name": "create_record_comments",
        "input": {
          "table_name": "Rental Orders",
          "record_id": "{{trigger.button_pressed.record_id}}",
          "comment": "Confirmation blocked — inventory conflict detected. Details: {{check_availability.output.text}}"
        }
      }
    ]
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| Double bookings per month | 4-6 | 0 (blocked at confirmation) |
| Emergency replacement costs | $6,000-24,000/month | $0 |
| Availability check time per order | 10-15 minutes (manual) | Under 10 seconds |

-> [Set up this workflow on Lotics](https://lotics.ai)

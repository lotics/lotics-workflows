# Order Production Progress Tracker

## The problem

Sourcing agents manage 50-200 active orders across dozens of factories, each at a different production stage — material procurement, cutting, assembly, finishing, packing. Factories send updates sporadically via email and WeChat, and agents lose hours each week manually updating order status boards and fielding buyer status inquiries. When a factory falls behind schedule, the agent often does not find out until the ship date passes, leaving no time to expedite or rebook the vessel.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Orders | order_ref, buyer_id, supplier_id, product, qty, production_stage, expected_completion, ship_date, status | Order master |
| Production Updates | order_id, stage, completion_pct, notes, reported_at, reported_by | Factory progress updates |
| Buyers | name, email | Buyer contacts |

## Example prompts

- "Every Wednesday and Friday, check which orders have not received a production update in the past 5 days and email the factory asking for a status report."
- "Twice-weekly, flag stale orders with no recent factory updates and request progress reports."

## Workflow

**Trigger:** Recurring schedule every Wednesday and Friday at 09:00 UTC.

```json
[
  {
    "id": "get_active_orders",
    "type": "tool_call",
    "description": "Query all orders in production status",
    "tool_name": "query_records",
    "input": {
      "table_id": "orders",
      "filter": {
        "field": "status",
        "operator": "equals",
        "value": "in_production"
      }
    }
  },
  {
    "id": "check_each_order",
    "type": "foreach",
    "description": "Check each order for recent production updates",
    "items": "{{get_active_orders.output.data}}",
    "steps": [
      {
        "id": "get_latest_update",
        "type": "tool_call",
        "description": "Query the most recent production update for this order",
        "tool_name": "query_records",
        "input": {
          "table_id": "production_updates",
          "filter": {
            "field": "order_id",
            "operator": "equals",
            "value": "{{check_each_order.item.id}}"
          },
          "sort": { "field": "reported_at", "direction": "desc" },
          "limit": 1
        }
      },
      {
        "id": "check_stale",
        "type": "if",
        "description": "If no update in the last 5 days, chase the factory",
        "condition": "{{get_latest_update.output.data.length === 0 || get_latest_update.output.data[0].reported_at < DATE_ADD(NOW(), -5, 'days')}}",
        "then": [
          {
            "id": "get_supplier",
            "type": "tool_call",
            "description": "Fetch factory contact details",
            "tool_name": "get_record",
            "input": {
              "table_id": "suppliers",
              "record_id": "{{check_each_order.item.supplier_id}}"
            }
          },
          {
            "id": "chase_factory",
            "type": "tool_call",
            "description": "Email the factory requesting a production update",
            "tool_name": "gmail_send_email",
            "input": {
              "to": "{{get_supplier.output.data.factory_contact_email}}",
              "subject": "Production Update Required — Order {{check_each_order.item.order_ref}}",
              "body": "Dear {{get_supplier.output.data.name}},\n\nWe have not received a production update for order {{check_each_order.item.order_ref}} ({{check_each_order.item.product}}, qty: {{check_each_order.item.qty}}) in the past 5 days.\n\nCurrent stage on record: {{check_each_order.item.production_stage}}\nShip date: {{check_each_order.item.ship_date}}\n\nPlease provide:\n1. Current production stage and completion percentage\n2. Whether you are on track for the ship date\n3. Any issues or delays\n\nThank you."
            }
          },
          {
            "id": "flag_stale",
            "type": "tool_call",
            "description": "Add a comment to the order flagging the stale update",
            "tool_name": "create_record_comments",
            "input": {
              "table_id": "orders",
              "record_id": "{{check_each_order.item.id}}",
              "comment": "No production update received in 5+ days. Factory chased via email."
            }
          }
        ],
        "else": []
      }
    ]
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| Orders with stale updates (5+ days) | 30-40% at any time | Under 10% |
| Late shipments discovered after ship date | 15-20% | Under 5% |
| Hours spent chasing factory updates | 8-10 hrs/week | Under 2 hrs/week |

→ [Set up this workflow on Lotics](https://lotics.ai)

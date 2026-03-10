# Order Fulfillment Pick List Generation

## The problem

Food distributors receive orders from restaurants, grocery stores, and institutions throughout the day for next-morning delivery. By the evening cut-off, the warehouse team needs consolidated pick lists organized by zone and delivery route to start picking. Currently, a supervisor manually compiles orders, groups items by cold storage zone (frozen, chilled, dry), and sequences them by route. This takes 1-2 hours every evening and delays the start of picking, pushing back loading and departure times.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Orders | order_number, customer_id, delivery_route, delivery_date, status | Customer orders |
| Order Lines | order_id, sku, product_name, qty, unit, storage_zone, lot_preference | Items per order |
| Pick Lists | delivery_date, route, storage_zone, status, created_at | Grouped pick lists |
| Pick List Lines | pick_list_id, order_line_id, sku, product_name, qty, pick_location, lot_number | Individual pick instructions |

## Example prompts

- "At 6 PM every day, generate pick lists for tomorrow's deliveries grouped by storage zone and delivery route."
- "Automatically create consolidated pick lists from pending orders each evening."

## Workflow

**Trigger:** Recurring daily schedule at 18:00 UTC (evening cut-off).

```json
[
  {
    "id": "get_tomorrows_orders",
    "type": "tool_call",
    "description": "Query all confirmed orders for tomorrow's delivery",
    "tool_name": "query_records",
    "input": {
      "table_id": "orders",
      "filter": {
        "and": [
          { "field": "delivery_date", "operator": "equals", "value": "{{DATE_ADD(NOW(), 1, 'days')}}" },
          { "field": "status", "operator": "equals", "value": "confirmed" }
        ]
      }
    }
  },
  {
    "id": "get_all_order_lines",
    "type": "tool_call",
    "description": "Fetch all line items for tomorrow's orders",
    "tool_name": "query_records",
    "input": {
      "table_id": "order_lines",
      "filter": {
        "field": "order_id",
        "operator": "in",
        "value": "{{get_tomorrows_orders.output.data.map(function(o) { return o.id })}}"
      }
    }
  },
  {
    "id": "group_pick_lists",
    "type": "tool_call",
    "description": "Group order lines into pick lists by route and storage zone",
    "tool_name": "llm_generate_text",
    "input": {
      "prompt": "Group these order lines into pick lists by delivery_route and storage_zone. Orders: {{JSON.stringify(get_tomorrows_orders.output.data.map(function(o) { return { id: o.id, route: o.delivery_route } }))}}. Order lines: {{JSON.stringify(get_all_order_lines.output.data)}}. Return JSON: { \"pick_lists\": [{ \"route\": string, \"storage_zone\": string, \"lines\": [{ \"order_line_id\": string, \"sku\": string, \"product_name\": string, \"qty\": number, \"unit\": string }] }] }. Sort lines within each list by SKU for efficient picking. Only return JSON."
    }
  },
  {
    "id": "create_pick_lists",
    "type": "foreach",
    "description": "Create a pick list record for each route/zone combination",
    "items": "{{JSON.parse(group_pick_lists.output.text).pick_lists}}",
    "steps": [
      {
        "id": "create_pick_list",
        "type": "tool_call",
        "description": "Create the pick list header record",
        "tool_name": "create_record",
        "input": {
          "table_id": "pick_lists",
          "data": {
            "delivery_date": "{{DATE_ADD(NOW(), 1, 'days')}}",
            "route": "{{create_pick_lists.item.route}}",
            "storage_zone": "{{create_pick_lists.item.storage_zone}}",
            "status": "pending",
            "created_at": "{{NOW()}}"
          }
        }
      },
      {
        "id": "create_pick_lines",
        "type": "foreach",
        "description": "Create individual pick list line items",
        "items": "{{create_pick_lists.item.lines}}",
        "steps": [
          {
            "id": "create_pick_line",
            "type": "tool_call",
            "description": "Create a pick list line record",
            "tool_name": "create_record",
            "input": {
              "table_id": "pick_list_lines",
              "data": {
                "pick_list_id": "{{create_pick_list.output.data.id}}",
                "order_line_id": "{{create_pick_lines.item.order_line_id}}",
                "sku": "{{create_pick_lines.item.sku}}",
                "product_name": "{{create_pick_lines.item.product_name}}",
                "qty": "{{create_pick_lines.item.qty}}"
              }
            }
          }
        ]
      }
    ]
  },
  {
    "id": "notify_warehouse",
    "type": "tool_call",
    "description": "Notify the warehouse team that pick lists are ready",
    "tool_name": "send_notification",
    "input": {
      "message": "Pick lists generated for {{DATE_ADD(NOW(), 1, 'days')}} deliveries. {{get_tomorrows_orders.output.data.length}} orders across {{JSON.parse(group_pick_lists.output.text).pick_lists.length}} pick lists. Picking can begin."
    }
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| Pick list preparation time | 1-2 hours/evening | Under 5 minutes |
| Picking start time | 8:30 PM | 6:15 PM |
| Loading departure delay | 30-60 min late | On time |

→ [Set up this workflow on Lotics](https://lotics.ai)

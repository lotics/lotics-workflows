# Low Stock Replenishment Alert

## The problem

Warehouses fulfilling 500+ orders per day need to keep pick-face locations stocked from bulk storage. When a pick location runs dry, pickers walk to the back of the warehouse to find stock or — worse — skip the order, causing a backlog. Supervisors currently rely on pickers shouting for replenishment or running manual stock reports every few hours, leading to 15-30 minute delays per stockout event and 20-40 missed SLA windows per week.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Pick Locations | location_code, sku, current_qty, min_qty, max_qty, zone | Pick face inventory levels |
| Bulk Inventory | sku, location_code, qty_available | Bulk storage positions |
| Replenishment Tasks | pick_location_id, sku, qty_to_replenish, status, assigned_to, created_at | Replenishment work orders |

## Example prompts

- "Every 2 hours, check all pick locations where stock is below the minimum level and create replenishment tasks from bulk inventory."
- "Automatically generate replenishment tasks whenever pick face stock drops below the reorder point."

## Workflow

**Trigger:** Recurring schedule every 2 hours during warehouse operating hours.

```json
[
  {
    "id": "find_low_stock",
    "type": "tool_call",
    "description": "Find pick locations where current quantity is at or below minimum",
    "tool_name": "query_records",
    "input": {
      "table_id": "pick_locations",
      "filter": {
        "field": "current_qty",
        "operator": "less_than_or_equals",
        "value_field": "min_qty"
      }
    }
  },
  {
    "id": "create_replenishments",
    "type": "foreach",
    "description": "For each low-stock location, check bulk and create a replenishment task",
    "items": "{{find_low_stock.output.data}}",
    "steps": [
      {
        "id": "check_bulk",
        "type": "tool_call",
        "description": "Check if bulk inventory has stock for this SKU",
        "tool_name": "query_records",
        "input": {
          "table_id": "bulk_inventory",
          "filter": {
            "and": [
              { "field": "sku", "operator": "equals", "value": "{{create_replenishments.item.sku}}" },
              { "field": "qty_available", "operator": "greater_than", "value": 0 }
            ]
          }
        }
      },
      {
        "id": "has_bulk_stock",
        "type": "if",
        "description": "Only create task if bulk stock exists",
        "condition": "{{check_bulk.output.data.length > 0}}",
        "then": [
          {
            "id": "create_task",
            "type": "tool_call",
            "description": "Create a replenishment task to refill the pick location",
            "tool_name": "create_record",
            "input": {
              "table_id": "replenishment_tasks",
              "data": {
                "pick_location_id": "{{create_replenishments.item.id}}",
                "sku": "{{create_replenishments.item.sku}}",
                "qty_to_replenish": "{{create_replenishments.item.max_qty - create_replenishments.item.current_qty}}",
                "bulk_location": "{{check_bulk.output.data[0].location_code}}",
                "status": "pending",
                "created_at": "{{NOW()}}"
              }
            }
          }
        ],
        "else": [
          {
            "id": "alert_no_bulk",
            "type": "tool_call",
            "description": "Alert that SKU has no bulk stock available for replenishment",
            "tool_name": "send_notification",
            "input": {
              "message": "SKU {{create_replenishments.item.sku}} at pick location {{create_replenishments.item.location_code}} is below minimum but no bulk stock is available. Procurement action needed.",
              "record_id": "{{create_replenishments.item.id}}"
            }
          }
        ]
      }
    ]
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| Pick location stockouts per day | 15-25 | 2-4 |
| Missed SLA windows per week | 20-40 | Under 5 |
| Picker idle time from stockouts | 45 min/day avg | Under 10 min/day |

→ [Set up this workflow on Lotics](https://lotics.ai)

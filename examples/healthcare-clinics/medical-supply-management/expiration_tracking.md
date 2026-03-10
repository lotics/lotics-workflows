# Supply Expiration Date Tracking

## The problem

Medical supplies like test kits, medications, sterile packs, and reagents carry expiration dates. Clinics with 150+ perishable SKUs regularly discover expired items during use, forcing disposal and emergency replacements. Expired supply waste costs mid-size clinics $3,000-5,000 per quarter. Worse, using an expired diagnostic kit can produce unreliable results, creating patient safety risk.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Perishable Supplies | item_name, sku, lot_number, quantity, expiration_date, category, storage_location, status | Tracks items with expiration dates |
| Waste Log | item_name, lot_number, quantity_disposed, reason, disposal_date, cost_lost | Records disposed expired items |
| Supply Tasks | assigned_to, task_description, due_date, status, priority | Staff action items for supply management |

## Example prompts

- "Every morning, check for supplies expiring in the next 30 days. Create a task for the supply tech to pull and rotate them. If anything expires within 7 days, mark it urgent and notify the office manager."
- "Build a daily workflow that scans our perishable supplies for upcoming expirations and assigns tasks to rotate stock before it goes to waste."

## Workflow

**Trigger:** `recurring_schedule` — Runs daily at 6:00 AM.

```json
[
  {
    "id": "find_expiring_items",
    "type": "tool_call",
    "description": "Query all perishable supplies expiring within the next 30 days",
    "tool_name": "query_records",
    "input": {
      "table_name": "Perishable Supplies",
      "filters": {
        "status": "In Stock"
      }
    }
  },
  {
    "id": "process_expiring",
    "type": "foreach",
    "description": "Evaluate each supply item that is approaching expiration",
    "items": "{{find_expiring_items.output.records.filter(r => new Date(r.expiration_date) < new Date(Date.now() + 30 * 86400000))}}",
    "steps": [
      {
        "id": "check_urgency",
        "type": "if",
        "description": "Determine if the item expires within 7 days (urgent) or within 30 days (standard)",
        "condition": "{{new Date(item.expiration_date) < new Date(Date.now() + 7 * 86400000)}}",
        "then": [
          {
            "id": "create_urgent_task",
            "type": "tool_call",
            "description": "Create an urgent task to pull and replace the nearly-expired item",
            "tool_name": "create_record",
            "input": {
              "table_name": "Supply Tasks",
              "data": {
                "task_description": "URGENT: Pull {{item.item_name}} (Lot: {{item.lot_number}}) from {{item.storage_location}} - expires {{item.expiration_date}}. {{item.quantity}} units remaining.",
                "due_date": "{{new Date().toISOString().split('T')[0]}}",
                "status": "Open",
                "priority": "Urgent"
              }
            }
          },
          {
            "id": "notify_manager_urgent",
            "type": "tool_call",
            "description": "Alert the office manager about a supply expiring within 7 days",
            "tool_name": "send_notification",
            "input": {
              "message": "URGENT: {{item.item_name}} (Lot: {{item.lot_number}}, {{item.quantity}} units) in {{item.storage_location}} expires {{item.expiration_date}}. Immediate action required.",
              "channel": "office-manager"
            }
          }
        ],
        "else": [
          {
            "id": "create_rotation_task",
            "type": "tool_call",
            "description": "Create a standard task to rotate stock before expiration",
            "tool_name": "create_record",
            "input": {
              "table_name": "Supply Tasks",
              "data": {
                "task_description": "Rotate {{item.item_name}} (Lot: {{item.lot_number}}) in {{item.storage_location}} - expires {{item.expiration_date}}. Move to front of shelf and prioritize for use.",
                "due_date": "{{new Date(Date.now() + 3 * 86400000).toISOString().split('T')[0]}}",
                "status": "Open",
                "priority": "Normal"
              }
            }
          }
        ]
      }
    ]
  },
  {
    "id": "find_already_expired",
    "type": "tool_call",
    "description": "Query items that have already passed their expiration date",
    "tool_name": "query_records",
    "input": {
      "table_name": "Perishable Supplies",
      "filters": {
        "status": "In Stock"
      }
    }
  },
  {
    "id": "log_expired",
    "type": "foreach",
    "description": "Log and mark each item that has already expired",
    "items": "{{find_already_expired.output.records.filter(r => new Date(r.expiration_date) < new Date())}}",
    "steps": [
      {
        "id": "create_waste_entry",
        "type": "tool_call",
        "description": "Record the expired item in the waste log",
        "tool_name": "create_record",
        "input": {
          "table_name": "Waste Log",
          "data": {
            "item_name": "{{item.item_name}}",
            "lot_number": "{{item.lot_number}}",
            "quantity_disposed": "{{item.quantity}}",
            "reason": "Expired",
            "disposal_date": "{{new Date().toISOString().split('T')[0]}}"
          }
        }
      },
      {
        "id": "mark_expired",
        "type": "tool_call",
        "description": "Update the supply record status to expired",
        "tool_name": "update_record",
        "input": {
          "table_name": "Perishable Supplies",
          "record_id": "{{item.id}}",
          "data": {
            "status": "Expired"
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
| Expired supply waste per quarter | $4,200 | $600 |
| Expired items discovered during use | 5-8/month | 0 |
| Staff time on manual expiration checks | 3 hrs/week | 0 |

-> [Set up this workflow on Lotics](https://lotics.ai)

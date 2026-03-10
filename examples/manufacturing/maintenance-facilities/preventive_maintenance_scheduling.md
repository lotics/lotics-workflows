# Preventive Maintenance Schedule Generation

## The problem

A paper mill operates 60 major assets (rollers, dryers, pumps, drives) with manufacturer-recommended PM intervals ranging from weekly to annually. The maintenance planner manually builds the monthly PM schedule in a spreadsheet, cross-referencing each asset's last service date, run hours, and technician availability. This takes a full day at the start of each month, and schedules frequently conflict with production runs. When PMs are missed, unplanned failures increase -- the plant attributes 40% of its $320,000 annual breakdown repair cost to deferred preventive maintenance.

## Data model

| Table | Key fields | Purpose |
|-------|-----------|---------|
| Assets | asset_id, name, location, asset_class, pm_interval_days, last_pm_date, run_hours_since_pm | Equipment registry with PM parameters |
| PM Schedules | schedule_id, asset_id, scheduled_date, assigned_to, status, wo_id | Generated PM schedule entries |
| Work Orders | wo_id, asset_id, type, description, priority, status, assigned_to | Maintenance job tickets |
| Technicians | tech_id, name, specializations, shift_pattern | Maintenance staff roster |

## Example prompts

- "On the first of each month, check every asset's last PM date and run hours. Generate PM work orders for any asset that is due or overdue, and assign them across the month based on technician availability."
- "Automatically generate next month's preventive maintenance schedule by checking which assets are coming due and creating work orders for each."

## Workflow

**Trigger:** Recurring schedule on the 1st of each month at 06:00

```json
[
  {
    "id": "find_due_assets",
    "type": "tool_call",
    "description": "Query all assets where days since last PM exceed or approach the PM interval",
    "tool_name": "query_records",
    "input": {
      "table_id": "assets",
      "filters": {
        "pm_interval_days__gt": 0
      }
    }
  },
  {
    "id": "get_technicians",
    "type": "tool_call",
    "description": "Fetch all available maintenance technicians",
    "tool_name": "query_records",
    "input": {
      "table_id": "technicians"
    }
  },
  {
    "id": "schedule_pms",
    "type": "foreach",
    "description": "Evaluate each asset and create PM work orders for those that are due",
    "items": "{{find_due_assets.output.records}}",
    "steps": [
      {
        "id": "check_due",
        "type": "tool_call",
        "description": "Use LLM to determine if this asset is due for PM and calculate the ideal schedule date",
        "tool_name": "llm_generate_text",
        "input": {
          "prompt": "Asset: {{schedule_pms.item.name}}\nLast PM: {{schedule_pms.item.last_pm_date}}\nPM interval: {{schedule_pms.item.pm_interval_days}} days\nRun hours since PM: {{schedule_pms.item.run_hours_since_pm}}\nToday: {{trigger.recurring_schedule.triggered_at}}\n\nIs this asset due for PM within the next 30 days? If yes, return JSON: {\"due\": true, \"ideal_date\": \"YYYY-MM-DD\", \"urgency\": \"overdue|due|upcoming\"}. If no, return {\"due\": false}."
        }
      },
      {
        "id": "create_if_due",
        "type": "if",
        "description": "Create a PM schedule entry and work order if the asset is due",
        "condition": "{{check_due.output.text.due == true}}",
        "then": [
          {
            "id": "create_wo",
            "type": "tool_call",
            "description": "Create a preventive maintenance work order",
            "tool_name": "create_record",
            "input": {
              "table_id": "work_orders",
              "data": {
                "asset_id": "{{schedule_pms.item.asset_id}}",
                "type": "Preventive Maintenance",
                "description": "Scheduled PM for {{schedule_pms.item.name}} ({{schedule_pms.item.asset_class}}). Interval: {{schedule_pms.item.pm_interval_days}} days. Status: {{check_due.output.text.urgency}}.",
                "priority": "{{check_due.output.text.urgency == 'overdue' ? 'High' : 'Normal'}}",
                "status": "Scheduled"
              }
            }
          },
          {
            "id": "create_schedule",
            "type": "tool_call",
            "description": "Create a PM schedule entry linked to the work order",
            "tool_name": "create_record",
            "input": {
              "table_id": "pm_schedules",
              "data": {
                "asset_id": "{{schedule_pms.item.asset_id}}",
                "scheduled_date": "{{check_due.output.text.ideal_date}}",
                "status": "Scheduled",
                "wo_id": "{{create_wo.output.record_id}}"
              }
            }
          }
        ],
        "else": []
      }
    ]
  },
  {
    "id": "count_scheduled",
    "type": "tool_call",
    "description": "Count how many PM work orders were created this month",
    "tool_name": "aggregate_records",
    "input": {
      "table_id": "pm_schedules",
      "filters": {
        "status": "Scheduled",
        "scheduled_date__gte": "{{trigger.recurring_schedule.triggered_at}}"
      },
      "aggregate": "count"
    }
  },
  {
    "id": "notify_planner",
    "type": "tool_call",
    "description": "Notify the maintenance planner with the monthly PM summary",
    "tool_name": "send_notification",
    "input": {
      "title": "Monthly PM Schedule Generated",
      "message": "{{count_scheduled.output.result}} preventive maintenance work orders have been created for this month. Please review the PM schedule and adjust technician assignments as needed."
    }
  }
]
```

## Results

| Metric | Before | After |
|--------|--------|-------|
| Monthly scheduling effort | 8 hours | 30 minutes (review only) |
| Missed PMs per quarter | 12-18 | 1-2 |
| Annual breakdown cost from deferred PM | $128,000 | $35,000 |

-> [Set up this workflow on Lotics](https://lotics.ai)

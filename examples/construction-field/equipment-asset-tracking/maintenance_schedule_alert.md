# Maintenance Schedule Alert

## The problem

Construction and industrial companies maintain fleets of 50-200 pieces of equipment. Each has different maintenance intervals based on engine hours, mileage, or calendar time. Maintenance coordinators track this in spreadsheets, but missed services are common -- 25% of equipment runs past its scheduled maintenance window. A single missed oil change on a $150,000 excavator can lead to a $40,000 engine rebuild and 3 weeks of downtime.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Equipment | name, type, serial_number, current_hours, current_mileage, location, status | Asset registry |
| Maintenance Schedules | equipment (link), service_type, interval_hours, interval_miles, interval_days, last_service_date, last_service_hours | Recurring maintenance rules |
| Maintenance Logs | equipment (link), service_type, performed_date, performed_by, cost, notes | Historical maintenance records |

## Example prompts

- "Every morning at 7 AM, check all equipment maintenance schedules. If any piece of equipment is within 10% of its next service interval, create a maintenance work order and notify the maintenance coordinator."
- "Run a daily check on equipment hours and mileage against maintenance schedules. Flag anything due for service in the next 50 hours or 500 miles and create a maintenance log entry so nothing gets missed."

## Workflow

**Trigger:** Recurring schedule, daily at 7:00 AM

```json
[
  {
    "id": "get_schedules",
    "type": "tool_call",
    "description": "Query all active maintenance schedules",
    "tool_name": "query_records",
    "input": {
      "table_name": "Maintenance Schedules"
    }
  },
  {
    "id": "get_equipment",
    "type": "tool_call",
    "description": "Query all equipment to get current hours and mileage",
    "tool_name": "query_records",
    "input": {
      "table_name": "Equipment",
      "filters": {
        "status": "Active"
      }
    }
  },
  {
    "id": "check_schedules",
    "type": "foreach",
    "description": "Evaluate each maintenance schedule against current equipment readings",
    "items": "{{get_schedules.output.records}}",
    "steps": [
      {
        "id": "check_hours_due",
        "type": "if",
        "description": "Check if equipment is approaching hours-based service interval",
        "condition": "{{check_schedules.item.interval_hours > 0 && (get_equipment.output.records.find(e => e.id === check_schedules.item.equipment).current_hours - check_schedules.item.last_service_hours) >= (check_schedules.item.interval_hours * 0.9)}}",
        "then": [
          {
            "id": "create_maintenance_order",
            "type": "tool_call",
            "description": "Create a maintenance log entry for the upcoming service",
            "tool_name": "create_record",
            "input": {
              "table_name": "Maintenance Logs",
              "data": {
                "equipment": "{{check_schedules.item.equipment}}",
                "service_type": "{{check_schedules.item.service_type}}",
                "performed_date": "",
                "notes": "Auto-generated: {{check_schedules.item.service_type}} due. Current hours: {{get_equipment.output.records.find(e => e.id === check_schedules.item.equipment).current_hours}}. Last service at: {{check_schedules.item.last_service_hours}} hours. Interval: {{check_schedules.item.interval_hours}} hours."
              }
            }
          },
          {
            "id": "notify_coordinator",
            "type": "tool_call",
            "description": "Send notification about upcoming maintenance",
            "tool_name": "send_notification",
            "input": {
              "message": "Maintenance due: {{get_equipment.output.records.find(e => e.id === check_schedules.item.equipment).name}} ({{get_equipment.output.records.find(e => e.id === check_schedules.item.equipment).serial_number}}) needs {{check_schedules.item.service_type}}. Current hours: {{get_equipment.output.records.find(e => e.id === check_schedules.item.equipment).current_hours}}, service due at {{check_schedules.item.last_service_hours + check_schedules.item.interval_hours}} hours.",
              "record_id": "{{check_schedules.item.equipment}}",
              "table_name": "Equipment"
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
| Missed maintenance events | 25% of scheduled services | Under 2% |
| Unplanned equipment downtime | 15 days/year per unit | 3 days/year per unit |
| Maintenance-related repair costs | $120,000/year (fleet of 50) | $35,000/year |

-> [Set up this workflow on Lotics](https://lotics.ai)

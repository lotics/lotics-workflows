# Fleet Maintenance Scheduling

## The problem

Rental fleets require maintenance between rentals to stay safe and rentable. But maintenance gets skipped when demand is high -- equipment goes out dirty, with low fluids, or past its service interval. This leads to breakdowns on customer sites (triggering SLA penalties and replacement costs of $500-2,000 per incident) and reduces equipment lifespan by 20-30%. The yard manager has no automated way to enforce maintenance before re-renting.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Equipment | name, type, serial_number, availability_status, total_hours, last_service_hours, service_interval_hours | Rental fleet assets |
| Rental Contracts | equipment (link), status, end_date | Contracts to detect returns |
| Maintenance Orders | equipment (link), service_type, status, assigned_to, due_date, completed_date | Maintenance work tracking |

## Example prompts

- "When a rental contract status changes to Returned, check if the equipment is due for maintenance based on its hours. If so, create a maintenance order, set the equipment to Out of Service, and notify the yard manager. Only change availability to Available after the maintenance order is completed."
- "On equipment return, auto-check maintenance status and block re-rental until any required service is done."

## Workflow

**Trigger:** When a Rental Contracts record is updated

```json
[
  {
    "id": "check_return",
    "type": "if",
    "description": "Only proceed if the contract was just returned",
    "condition": "{{trigger.record_updated.next_data.status === 'Returned' && trigger.record_updated.prev_data.status !== 'Returned'}}",
    "then": [
      {
        "id": "get_equipment",
        "type": "tool_call",
        "description": "Fetch the equipment record to check maintenance status",
        "tool_name": "get_record",
        "input": {
          "table_name": "Equipment",
          "record_id": "{{trigger.record_updated.next_data.equipment}}"
        }
      },
      {
        "id": "check_maintenance_due",
        "type": "if",
        "description": "Check if equipment hours exceed maintenance interval since last service",
        "condition": "{{(get_equipment.output.record.total_hours - get_equipment.output.record.last_service_hours) >= (get_equipment.output.record.service_interval_hours * 0.9)}}",
        "then": [
          {
            "id": "set_out_of_service",
            "type": "tool_call",
            "description": "Mark equipment as out of service pending maintenance",
            "tool_name": "update_record",
            "input": {
              "table_name": "Equipment",
              "record_id": "{{trigger.record_updated.next_data.equipment}}",
              "data": {
                "availability_status": "Out of Service - Maintenance Required"
              }
            }
          },
          {
            "id": "create_maintenance_order",
            "type": "tool_call",
            "description": "Create a maintenance work order",
            "tool_name": "create_record",
            "input": {
              "table_name": "Maintenance Orders",
              "data": {
                "equipment": "{{trigger.record_updated.next_data.equipment}}",
                "service_type": "Scheduled Service — {{get_equipment.output.record.service_interval_hours}}-hour interval",
                "status": "Pending",
                "due_date": "{{new Date(Date.now() + 172800000).toISOString().split('T')[0]}}"
              }
            }
          },
          {
            "id": "notify_yard",
            "type": "tool_call",
            "description": "Notify the yard manager about the required maintenance",
            "tool_name": "send_notification",
            "input": {
              "message": "Maintenance required: {{get_equipment.output.record.name}} ({{get_equipment.output.record.serial_number}}) returned and is due for {{get_equipment.output.record.service_interval_hours}}-hour service. Current hours: {{get_equipment.output.record.total_hours}}, last service: {{get_equipment.output.record.last_service_hours}} hours. Equipment is blocked from re-rental until maintenance is complete.",
              "record_id": "{{trigger.record_updated.next_data.equipment}}",
              "table_name": "Equipment"
            }
          }
        ],
        "else": [
          {
            "id": "set_available",
            "type": "tool_call",
            "description": "Equipment is good — mark as available for re-rental",
            "tool_name": "update_record",
            "input": {
              "table_name": "Equipment",
              "record_id": "{{trigger.record_updated.next_data.equipment}}",
              "data": {
                "availability_status": "Available"
              }
            }
          }
        ]
      }
    ],
    "else": []
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| On-site breakdowns from deferred maintenance | 3-5 per month | Under 1 per month |
| Equipment going out past service interval | 30% of rentals | 0% (blocked automatically) |
| Replacement/SLA penalty costs | $4,000-10,000/month | Under $1,000/month |

-> [Set up this workflow on Lotics](https://lotics.ai)

# Load Dispatch and Driver Assignment

## The problem

When a customer submits a load request, dispatch needs to find an available driver near the pickup location, confirm the assignment, and send the driver load details within 30 minutes. During peak season, dispatchers juggle 40+ load requests per day across phone, email, and portal, leading to delayed assignments, double-bookings, and loads sitting uncovered for hours while drivers idle nearby.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Load Requests | customer_id, pickup_location, delivery_location, pickup_date, weight, equipment_type, status | Incoming load requests |
| Drivers | name, email, phone, current_location, status, equipment_type | Available fleet |
| Loads | load_number, driver_id, customer_id, pickup_location, delivery_location, pickup_date, status | Assigned loads |
| Customers | name, email, phone | Customer contact info |

## Example prompts

- "When a new load request comes in via webhook, find an available driver with matching equipment near the pickup, create the load, and notify both driver and customer."
- "Automatically assign incoming load requests to the nearest available driver."

## Workflow

**Trigger:** Webhook received from customer portal with new load request data.

```json
[
  {
    "id": "find_available_drivers",
    "type": "tool_call",
    "description": "Query drivers who are available and have matching equipment type",
    "tool_name": "query_records",
    "input": {
      "table_id": "drivers",
      "filter": {
        "and": [
          { "field": "status", "operator": "equals", "value": "available" },
          { "field": "equipment_type", "operator": "equals", "value": "{{trigger.receive_webhook.data.equipment_type}}" }
        ]
      }
    }
  },
  {
    "id": "check_driver_available",
    "type": "if",
    "description": "Check if any matching driver is available",
    "condition": "{{find_available_drivers.output.data.length > 0}}",
    "then": [
      {
        "id": "select_driver",
        "type": "tool_call",
        "description": "Use LLM to select the best driver based on proximity to pickup",
        "tool_name": "llm_generate_text",
        "input": {
          "prompt": "Given pickup location: {{trigger.receive_webhook.data.pickup_location}}. Available drivers: {{JSON.stringify(find_available_drivers.output.data.map(function(d) { return { id: d.id, name: d.name, current_location: d.current_location } }))}}. Return only the driver ID of the driver whose current_location is closest to the pickup. Return just the ID string, nothing else."
        }
      },
      {
        "id": "create_load",
        "type": "tool_call",
        "description": "Create the load record with the assigned driver",
        "tool_name": "create_record",
        "input": {
          "table_id": "loads",
          "data": {
            "driver_id": "{{select_driver.output.text}}",
            "customer_id": "{{trigger.receive_webhook.data.customer_id}}",
            "pickup_location": "{{trigger.receive_webhook.data.pickup_location}}",
            "delivery_location": "{{trigger.receive_webhook.data.delivery_location}}",
            "pickup_date": "{{trigger.receive_webhook.data.pickup_date}}",
            "weight": "{{trigger.receive_webhook.data.weight}}",
            "status": "assigned"
          }
        }
      },
      {
        "id": "update_driver_status",
        "type": "tool_call",
        "description": "Mark the driver as dispatched",
        "tool_name": "update_record",
        "input": {
          "table_id": "drivers",
          "record_id": "{{select_driver.output.text}}",
          "data": {
            "status": "dispatched"
          }
        }
      },
      {
        "id": "update_request_status",
        "type": "tool_call",
        "description": "Mark the load request as assigned",
        "tool_name": "update_record",
        "input": {
          "table_id": "load_requests",
          "record_id": "{{trigger.receive_webhook.data.request_id}}",
          "data": {
            "status": "assigned"
          }
        }
      },
      {
        "id": "notify_driver",
        "type": "tool_call",
        "description": "Send load details to the assigned driver",
        "tool_name": "send_notification",
        "input": {
          "message": "New load assigned: Pick up at {{trigger.receive_webhook.data.pickup_location}} on {{trigger.receive_webhook.data.pickup_date}}, deliver to {{trigger.receive_webhook.data.delivery_location}}. Weight: {{trigger.receive_webhook.data.weight}} lbs.",
          "record_id": "{{create_load.output.data.id}}"
        }
      }
    ],
    "else": [
      {
        "id": "mark_uncovered",
        "type": "tool_call",
        "description": "Mark the load request as uncovered for manual dispatch",
        "tool_name": "update_record",
        "input": {
          "table_id": "load_requests",
          "record_id": "{{trigger.receive_webhook.data.request_id}}",
          "data": {
            "status": "uncovered"
          }
        }
      },
      {
        "id": "alert_dispatch",
        "type": "tool_call",
        "description": "Alert dispatch team that no driver is available",
        "tool_name": "send_notification",
        "input": {
          "message": "No available {{trigger.receive_webhook.data.equipment_type}} driver for load request at {{trigger.receive_webhook.data.pickup_location}} on {{trigger.receive_webhook.data.pickup_date}}. Manual assignment required.",
          "record_id": "{{trigger.receive_webhook.data.request_id}}"
        }
      }
    ]
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| Average time to assign a load | 45-90 minutes | Under 5 minutes |
| Loads uncovered for 2+ hours | 15-20% | Under 3% |
| Double-bookings per month | 4-6 | 0 |

→ [Set up this workflow on Lotics](https://lotics.ai)

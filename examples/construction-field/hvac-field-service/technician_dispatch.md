# Technician Dispatch

## The problem

HVAC companies receive emergency service calls throughout the day -- a restaurant's walk-in cooler is down, an office building has no heat in January. Dispatchers must find a certified tech with the right EPA credentials, check their current schedule, estimate travel time, and call the customer back. This takes 15-25 minutes per call. During peak season (summer/winter), dispatchers handle 40+ calls per day and response times stretch to 3-4 hours, causing customer churn of 15-20% annually.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Service Tickets | customer (link), issue_type, urgency, description, status, assigned_tech (link), scheduled_time, site_address | Inbound service requests |
| Technicians | name, phone, email, certifications, service_zone, current_status, current_location | Field tech roster with availability |
| Customers | company_name, contact_name, phone, email, address, contract_tier | Customer directory |

## Example prompts

- "When a new service ticket is submitted, find the nearest available technician with the right certifications for the issue type, assign them, send the customer a confirmation with the tech's name and ETA, and text the tech the job details."
- "Auto-dispatch incoming HVAC service tickets: match by certification and zone, assign the tech, confirm with the customer, and notify the tech -- all within seconds of the ticket being created."

## Workflow

**Trigger:** When a record submit action fires on the Service Tickets table

```json
[
  {
    "id": "get_customer",
    "type": "tool_call",
    "description": "Fetch customer details for the ticket",
    "tool_name": "get_record",
    "input": {
      "table_name": "Customers",
      "record_id": "{{trigger.record_submit.data.customer}}"
    }
  },
  {
    "id": "find_techs",
    "type": "tool_call",
    "description": "Find available technicians in the customer's service zone",
    "tool_name": "query_records",
    "input": {
      "table_name": "Technicians",
      "filters": {
        "current_status": "Available",
        "service_zone": "{{trigger.record_submit.data.site_address}}"
      }
    }
  },
  {
    "id": "select_and_assign",
    "type": "if",
    "description": "Check if any techs are available",
    "condition": "{{find_techs.output.records.length > 0}}",
    "then": [
      {
        "id": "assign_tech",
        "type": "tool_call",
        "description": "Assign the first available qualified tech to the ticket",
        "tool_name": "update_record",
        "input": {
          "table_name": "Service Tickets",
          "record_id": "{{trigger.record_submit.record_id}}",
          "data": {
            "assigned_tech": "{{find_techs.output.records[0].id}}",
            "status": "Dispatched",
            "scheduled_time": "{{new Date(Date.now() + 3600000).toISOString()}}"
          }
        }
      },
      {
        "id": "update_tech_status",
        "type": "tool_call",
        "description": "Mark the technician as dispatched",
        "tool_name": "update_record",
        "input": {
          "table_name": "Technicians",
          "record_id": "{{find_techs.output.records[0].id}}",
          "data": {
            "current_status": "Dispatched"
          }
        }
      },
      {
        "id": "notify_customer",
        "type": "tool_call",
        "description": "Email the customer with dispatch confirmation",
        "tool_name": "gmail_send_email",
        "input": {
          "to": "{{get_customer.output.record.email}}",
          "subject": "HVAC Service Dispatched — {{trigger.record_submit.data.issue_type}}",
          "body": "Hi {{get_customer.output.record.contact_name}},\n\nWe've received your service request for {{trigger.record_submit.data.issue_type}} at {{trigger.record_submit.data.site_address}}.\n\nYour technician {{find_techs.output.records[0].name}} has been dispatched and will arrive within approximately 1 hour.\n\nTech contact: {{find_techs.output.records[0].phone}}\n\nIf you need to reach us before then, please call our dispatch line.\n\nThank you."
        }
      },
      {
        "id": "notify_tech",
        "type": "tool_call",
        "description": "Send the tech a notification with job details",
        "tool_name": "send_notification",
        "input": {
          "message": "New dispatch: {{trigger.record_submit.data.issue_type}} ({{trigger.record_submit.data.urgency}}) at {{trigger.record_submit.data.site_address}}. Customer: {{get_customer.output.record.company_name}} ({{get_customer.output.record.contact_name}}, {{get_customer.output.record.phone}}). Description: {{trigger.record_submit.data.description}}",
          "record_id": "{{trigger.record_submit.record_id}}",
          "table_name": "Service Tickets"
        }
      }
    ],
    "else": [
      {
        "id": "mark_unassigned",
        "type": "tool_call",
        "description": "Mark ticket as pending assignment",
        "tool_name": "update_record",
        "input": {
          "table_name": "Service Tickets",
          "record_id": "{{trigger.record_submit.record_id}}",
          "data": {
            "status": "Pending Assignment"
          }
        }
      },
      {
        "id": "alert_dispatch",
        "type": "tool_call",
        "description": "Alert dispatch that no techs are available",
        "tool_name": "send_notification",
        "input": {
          "message": "No available technicians for {{trigger.record_submit.data.issue_type}} ({{trigger.record_submit.data.urgency}}) at {{trigger.record_submit.data.site_address}}. Manual assignment required.",
          "record_id": "{{trigger.record_submit.record_id}}",
          "table_name": "Service Tickets"
        }
      }
    ]
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| Dispatch time per call | 15-25 minutes | Under 30 seconds |
| Average customer response time | 3-4 hours (peak season) | Under 1 hour |
| Annual customer churn from slow response | 15-20% | Under 5% |

-> [Set up this workflow on Lotics](https://lotics.ai)

# Work Order Dispatch

## The problem

Field service companies receive 30-50 work orders daily via email from property managers and facility contacts. A dispatcher manually reads each email, creates a work order, looks up technician availability, and assigns the job. This takes 8-12 minutes per work order and is error-prone -- double bookings happen 3-4 times per week, and 15% of work orders sit unassigned for more than 4 hours.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Work Orders | title, description, priority, status, customer_email, site_address, assigned_tech (link), scheduled_date | Service requests |
| Technicians | name, phone, email, skills, current_status, service_area | Field tech roster |
| Customers | name, email, phone, address, contract_type | Customer directory |

## Example prompts

- "When we receive an email requesting service, use AI to extract the job details, create a work order, find an available technician with matching skills in the same area, assign them, and reply to the customer with the tech's name and ETA."
- "Automatically turn incoming service request emails into work orders. Match the sender to a customer, pull out the issue description, assign the closest available tech, and confirm back to the customer by email."

## Workflow

**Trigger:** When a Gmail email is received matching the service request filter

```json
[
  {
    "id": "parse_email",
    "type": "tool_call",
    "description": "Use LLM to extract structured work order details from the email",
    "tool_name": "llm_generate_text",
    "input": {
      "prompt": "Extract the following fields from this service request email as JSON: site_address, issue_description, urgency (low/medium/high), requested_date, skill_needed (one of: plumbing, electrical, HVAC, general). If any field is missing, use null.\n\nFrom: {{trigger.receive_gmail_email.from}}\nSubject: {{trigger.receive_gmail_email.subject}}\nBody: {{trigger.receive_gmail_email.body}}",
      "response_format": "json"
    }
  },
  {
    "id": "find_customer",
    "type": "tool_call",
    "description": "Look up the customer by sender email",
    "tool_name": "query_records",
    "input": {
      "table_name": "Customers",
      "filters": {
        "email": "{{trigger.receive_gmail_email.from}}"
      }
    }
  },
  {
    "id": "find_available_tech",
    "type": "tool_call",
    "description": "Find an available technician with the right skill set",
    "tool_name": "query_records",
    "input": {
      "table_name": "Technicians",
      "filters": {
        "current_status": "Available"
      }
    }
  },
  {
    "id": "create_work_order",
    "type": "tool_call",
    "description": "Create the work order record",
    "tool_name": "create_record",
    "input": {
      "table_name": "Work Orders",
      "data": {
        "title": "{{trigger.receive_gmail_email.subject}}",
        "description": "{{JSON.parse(parse_email.output.text).issue_description}}",
        "priority": "{{JSON.parse(parse_email.output.text).urgency}}",
        "status": "Assigned",
        "customer_email": "{{trigger.receive_gmail_email.from}}",
        "site_address": "{{JSON.parse(parse_email.output.text).site_address}}",
        "assigned_tech": "{{find_available_tech.output.records[0].id}}",
        "scheduled_date": "{{JSON.parse(parse_email.output.text).requested_date}}"
      }
    }
  },
  {
    "id": "update_tech_status",
    "type": "tool_call",
    "description": "Mark the technician as assigned",
    "tool_name": "update_record",
    "input": {
      "table_name": "Technicians",
      "record_id": "{{find_available_tech.output.records[0].id}}",
      "data": {
        "current_status": "Assigned"
      }
    }
  },
  {
    "id": "reply_customer",
    "type": "tool_call",
    "description": "Reply to the customer email with assignment confirmation",
    "tool_name": "gmail_reply_email",
    "input": {
      "message_id": "{{trigger.receive_gmail_email.message_id}}",
      "body": "Hi,\n\nWe've received your service request and created work order #{{create_work_order.output.record_id}}.\n\nYour assigned technician is {{find_available_tech.output.records[0].name}} and they are scheduled for {{JSON.parse(parse_email.output.text).requested_date}}.\n\nIf you need to reschedule or have questions, reply to this email.\n\nThank you."
    }
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| Time to create and assign work order | 8-12 minutes | Under 30 seconds |
| Unassigned work orders after 4 hours | 15% | 0% |
| Double bookings per week | 3-4 | 0 |

-> [Set up this workflow on Lotics](https://lotics.ai)

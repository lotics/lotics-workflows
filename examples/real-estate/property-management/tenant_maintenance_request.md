# Tenant Maintenance Request Routing

## The problem

Property managers handling 200+ units receive 40-60 maintenance requests per month via email. Each request must be triaged by urgency (water leak vs. cosmetic fix), assigned to the right vendor, and tracked to completion. Without automation, requests sit in inboxes for 24-48 hours before anyone reads them, tenants send frustrated follow-ups, and urgent issues like plumbing leaks cause thousands in preventable damage.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Maintenance Requests | request_id, unit_number, tenant_name, description, urgency, status, assigned_vendor, submitted_at | Track every maintenance ticket from submission to resolution |
| Units | unit_id, building, floor, tenant_id, lease_start, lease_end | Property unit registry |
| Vendors | vendor_id, name, trade, phone, email, availability | Licensed contractors and service providers |

## Example prompts

- "When a tenant submits a maintenance request form, classify the urgency using AI, assign it to an available vendor based on the trade needed, and notify both the tenant and vendor by email."
- "Set up a workflow so that every new maintenance request gets triaged automatically and the right contractor is emailed with the unit details and problem description."

## Workflow

**Trigger:** A tenant submits a maintenance request through a shared form.

```json
[
  {
    "id": "classify_urgency",
    "type": "tool_call",
    "description": "Use AI to classify the request urgency and identify the trade required",
    "tool_name": "llm_generate_text",
    "input": {
      "prompt": "Classify this maintenance request. Return a JSON object with two fields: \"urgency\" (one of: emergency, high, medium, low) and \"trade\" (one of: plumbing, electrical, hvac, general, appliance). Request: {{trigger.form_submitted.data.description}}"
    }
  },
  {
    "id": "update_request",
    "type": "tool_call",
    "description": "Update the maintenance request record with classified urgency",
    "tool_name": "update_record",
    "input": {
      "table_name": "Maintenance Requests",
      "record_id": "{{trigger.form_submitted.record_id}}",
      "data": {
        "urgency": "{{classify_urgency.output.urgency}}",
        "status": "Triaged"
      }
    }
  },
  {
    "id": "find_vendor",
    "type": "tool_call",
    "description": "Find an available vendor matching the required trade",
    "tool_name": "query_records",
    "input": {
      "table_name": "Vendors",
      "filters": {
        "trade": "{{classify_urgency.output.trade}}",
        "availability": "Available"
      },
      "limit": 1
    }
  },
  {
    "id": "assign_vendor",
    "type": "tool_call",
    "description": "Assign the matched vendor to the request",
    "tool_name": "update_record",
    "input": {
      "table_name": "Maintenance Requests",
      "record_id": "{{trigger.form_submitted.record_id}}",
      "data": {
        "assigned_vendor": "{{find_vendor.output.records[0].vendor_id}}",
        "status": "Assigned"
      }
    }
  },
  {
    "id": "notify_vendor",
    "type": "tool_call",
    "description": "Email the assigned vendor with job details",
    "tool_name": "gmail_send_email",
    "input": {
      "to": "{{find_vendor.output.records[0].email}}",
      "subject": "New Maintenance Request - Unit {{trigger.form_submitted.data.unit_number}} [{{classify_urgency.output.urgency}}]",
      "body": "You have been assigned a new maintenance request.\n\nUnit: {{trigger.form_submitted.data.unit_number}}\nUrgency: {{classify_urgency.output.urgency}}\nDescription: {{trigger.form_submitted.data.description}}\n\nPlease respond with your estimated arrival time."
    }
  },
  {
    "id": "notify_tenant",
    "type": "tool_call",
    "description": "Send the tenant a confirmation with vendor details",
    "tool_name": "send_notification",
    "input": {
      "recipient_id": "{{trigger.form_submitted.data.tenant_id}}",
      "message": "Your maintenance request for Unit {{trigger.form_submitted.data.unit_number}} has been received and assigned to {{find_vendor.output.records[0].name}}. Urgency: {{classify_urgency.output.urgency}}."
    }
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| Average triage time | 24-48 hours | Under 2 minutes |
| Emergency response time | 6+ hours | Under 30 minutes |
| Tenant follow-up complaints per month | 15-20 | 2-3 |
| Requests lost or forgotten | 5-8 per month | 0 |

-> [Set up this workflow on Lotics](https://lotics.ai)

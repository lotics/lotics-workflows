# Import Permit Expiry Alert

## The problem

Import businesses hold dozens of permits and licenses — import permits, fumigation certificates, food safety approvals, quota allocations — each with different expiry dates. When a permit expires unnoticed, incoming shipments get held at the border, incurring $1,000-5,000/day in demurrage and storage. Most teams track expiry dates in spreadsheets that nobody checks consistently.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Permits | permit_number, permit_type, issuing_authority, expiry_date, status, responsible_person_id | Active permits and licenses |
| Members | name, email | Team member contact info |

## Example prompts

- "Every Monday, check for permits expiring in the next 30 days and send an alert to the responsible person."
- "Set up a weekly scan for soon-to-expire import permits and notify the team."

## Workflow

**Trigger:** Recurring weekly schedule every Monday at 09:00 UTC.

```json
[
  {
    "id": "find_expiring_permits",
    "type": "tool_call",
    "description": "Query permits expiring within the next 30 days",
    "tool_name": "query_records",
    "input": {
      "table_id": "permits",
      "filter": {
        "and": [
          { "field": "status", "operator": "equals", "value": "active" },
          { "field": "expiry_date", "operator": "less_than_or_equals", "value": "{{DATE_ADD(NOW(), 30, 'days')}}" },
          { "field": "expiry_date", "operator": "greater_than", "value": "{{NOW()}}" }
        ]
      }
    }
  },
  {
    "id": "check_any_expiring",
    "type": "if",
    "description": "Only proceed if there are permits expiring soon",
    "condition": "{{find_expiring_permits.output.data.length > 0}}",
    "then": [
      {
        "id": "notify_each_permit",
        "type": "foreach",
        "description": "Send an alert for each expiring permit to the responsible person",
        "items": "{{find_expiring_permits.output.data}}",
        "steps": [
          {
            "id": "get_responsible",
            "type": "tool_call",
            "description": "Look up the responsible team member",
            "tool_name": "query_members",
            "input": {
              "member_id": "{{notify_each_permit.item.responsible_person_id}}"
            }
          },
          {
            "id": "send_alert",
            "type": "tool_call",
            "description": "Send an expiry alert notification",
            "tool_name": "send_notification",
            "input": {
              "message": "Permit {{notify_each_permit.item.permit_number}} ({{notify_each_permit.item.permit_type}}) expires on {{notify_each_permit.item.expiry_date}}. Please initiate renewal with {{notify_each_permit.item.issuing_authority}}.",
              "record_id": "{{notify_each_permit.item.id}}"
            }
          },
          {
            "id": "email_alert",
            "type": "tool_call",
            "description": "Email the responsible person directly",
            "tool_name": "gmail_send_email",
            "input": {
              "to": "{{get_responsible.output.data.email}}",
              "subject": "Action Required: Import permit {{notify_each_permit.item.permit_number}} expires {{notify_each_permit.item.expiry_date}}",
              "body": "Hi {{get_responsible.output.data.name}},\n\nThis is a reminder that the following permit is expiring soon:\n\n- Permit: {{notify_each_permit.item.permit_number}}\n- Type: {{notify_each_permit.item.permit_type}}\n- Issuing Authority: {{notify_each_permit.item.issuing_authority}}\n- Expiry Date: {{notify_each_permit.item.expiry_date}}\n\nPlease initiate the renewal process as soon as possible to avoid shipment delays.\n\nRegards,\nLotics Workflow"
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
| Permits that lapsed unnoticed | 3-5/year | 0 |
| Demurrage from expired permits | $8,000-20,000/year | $0 |
| Time spent manually tracking expiry | 2 hrs/week | 0 |

→ [Set up this workflow on Lotics](https://lotics.ai)

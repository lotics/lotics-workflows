# Hazmat Training and Certification Tracker

## The problem

Every employee who handles, packs, loads, or documents dangerous goods must hold current hazmat training certification — IATA DGR for air, IMDG for sea, DOT/49 CFR for ground. Certifications expire every 2-3 years depending on jurisdiction and must be renewed before expiry or the employee cannot perform DG functions. Companies with 20+ DG-qualified staff managing certifications in spreadsheets routinely discover expired certs during audits or — worse — after an incident, exposing the company to $50,000+ fines and criminal liability.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Employees | name, email, department, dg_role | DG-handling staff |
| Certifications | employee_id, cert_type, issuing_body, issue_date, expiry_date, cert_number, status | Training certifications |
| Training Requests | employee_id, cert_type, requested_date, training_provider, status | Renewal scheduling |

## Example prompts

- "On the first of every month, find DG certifications expiring in the next 90 days and create training renewal requests for each employee."
- "Monthly hazmat certification scan — flag expiring certs and schedule renewals automatically."

## Workflow

**Trigger:** Recurring monthly schedule on the 1st at 08:00 UTC.

```json
[
  {
    "id": "find_expiring_certs",
    "type": "tool_call",
    "description": "Query DG certifications expiring within 90 days",
    "tool_name": "query_records",
    "input": {
      "table_id": "certifications",
      "filter": {
        "and": [
          { "field": "status", "operator": "equals", "value": "active" },
          { "field": "expiry_date", "operator": "less_than_or_equals", "value": "{{DATE_ADD(NOW(), 90, 'days')}}" },
          { "field": "expiry_date", "operator": "greater_than", "value": "{{NOW()}}" }
        ]
      }
    }
  },
  {
    "id": "find_already_expired",
    "type": "tool_call",
    "description": "Find certifications that have already expired but are still marked active",
    "tool_name": "query_records",
    "input": {
      "table_id": "certifications",
      "filter": {
        "and": [
          { "field": "status", "operator": "equals", "value": "active" },
          { "field": "expiry_date", "operator": "less_than_or_equals", "value": "{{NOW()}}" }
        ]
      }
    }
  },
  {
    "id": "deactivate_expired",
    "type": "foreach",
    "description": "Mark already-expired certifications as expired",
    "items": "{{find_already_expired.output.data}}",
    "steps": [
      {
        "id": "mark_expired",
        "type": "tool_call",
        "description": "Update the certification status to expired",
        "tool_name": "update_record",
        "input": {
          "table_id": "certifications",
          "record_id": "{{deactivate_expired.item.id}}",
          "data": {
            "status": "expired"
          }
        }
      },
      {
        "id": "get_employee_expired",
        "type": "tool_call",
        "description": "Fetch the employee whose cert expired",
        "tool_name": "get_record",
        "input": {
          "table_id": "employees",
          "record_id": "{{deactivate_expired.item.employee_id}}"
        }
      },
      {
        "id": "urgent_alert",
        "type": "tool_call",
        "description": "Send urgent notification about expired certification",
        "tool_name": "send_notification",
        "input": {
          "message": "EXPIRED: {{get_employee_expired.output.data.name}}'s {{deactivate_expired.item.cert_type}} certification ({{deactivate_expired.item.cert_number}}) expired on {{deactivate_expired.item.expiry_date}}. This employee MUST NOT perform DG functions until recertified.",
          "record_id": "{{deactivate_expired.item.id}}"
        }
      }
    ]
  },
  {
    "id": "process_expiring",
    "type": "foreach",
    "description": "Create training renewal requests for expiring certifications",
    "items": "{{find_expiring_certs.output.data}}",
    "steps": [
      {
        "id": "create_training_request",
        "type": "tool_call",
        "description": "Create a training renewal request",
        "tool_name": "create_record",
        "input": {
          "table_id": "training_requests",
          "data": {
            "employee_id": "{{process_expiring.item.employee_id}}",
            "cert_type": "{{process_expiring.item.cert_type}}",
            "requested_date": "{{NOW()}}",
            "status": "pending"
          }
        }
      },
      {
        "id": "get_employee",
        "type": "tool_call",
        "description": "Fetch employee contact details",
        "tool_name": "get_record",
        "input": {
          "table_id": "employees",
          "record_id": "{{process_expiring.item.employee_id}}"
        }
      },
      {
        "id": "email_employee",
        "type": "tool_call",
        "description": "Notify the employee about upcoming certification expiry",
        "tool_name": "gmail_send_email",
        "input": {
          "to": "{{get_employee.output.data.email}}",
          "subject": "Hazmat Certification Renewal Required — {{process_expiring.item.cert_type}} expires {{process_expiring.item.expiry_date}}",
          "body": "Hi {{get_employee.output.data.name}},\n\nYour {{process_expiring.item.cert_type}} certification ({{process_expiring.item.cert_number}}) expires on {{process_expiring.item.expiry_date}}.\n\nA training renewal request has been created. Please coordinate with your manager to schedule recertification training before the expiry date.\n\nYou will not be able to perform DG handling, packing, or documentation duties after your certification expires.\n\nThank you."
        }
      }
    ]
  },
  {
    "id": "send_summary",
    "type": "if",
    "description": "Send a monthly summary if any certs need attention",
    "condition": "{{find_expiring_certs.output.data.length > 0 || find_already_expired.output.data.length > 0}}",
    "then": [
      {
        "id": "summary_notification",
        "type": "tool_call",
        "description": "Send monthly DG certification summary to management",
        "tool_name": "send_notification",
        "input": {
          "message": "Monthly DG Certification Report: {{find_already_expired.output.data.length}} expired cert(s) deactivated, {{find_expiring_certs.output.data.length}} cert(s) expiring within 90 days with renewal requests created."
        }
      }
    ],
    "else": []
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| Staff operating with expired DG certs | 2-5 at any time | 0 |
| Audit non-conformances for training | 3-6 per audit | 0 |
| Time spent tracking certifications | 4 hrs/month | 0 |

→ [Set up this workflow on Lotics](https://lotics.ai)

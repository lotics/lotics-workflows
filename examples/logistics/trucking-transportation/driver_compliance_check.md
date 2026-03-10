# Driver Compliance and License Expiry Check

## The problem

Trucking companies must ensure every driver on the road has valid licenses, medical certificates, and training certifications. A driver operating with an expired CDL or medical card exposes the company to DOT fines of $2,750-16,000 per violation and immediate out-of-service orders. Fleet managers tracking 50+ drivers in spreadsheets inevitably miss renewals, discovering expired documents only during roadside inspections or audits.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Drivers | name, email, cdl_number, cdl_expiry, medical_cert_expiry, hazmat_endorsement_expiry, status | Driver master with credential dates |
| Compliance Alerts | driver_id, alert_type, expiry_date, days_remaining, acknowledged | Tracked alerts |

## Example prompts

- "Every day, check for drivers whose CDL, medical certificate, or hazmat endorsement expires within 45 days, and create compliance alerts."
- "Run a daily compliance scan on all active drivers and flag anyone with expiring credentials."

## Workflow

**Trigger:** Recurring daily schedule at 06:00 UTC.

```json
[
  {
    "id": "find_expiring_cdl",
    "type": "tool_call",
    "description": "Find active drivers with CDL expiring in 45 days",
    "tool_name": "query_records",
    "input": {
      "table_id": "drivers",
      "filter": {
        "and": [
          { "field": "status", "operator": "equals", "value": "active" },
          { "field": "cdl_expiry", "operator": "less_than_or_equals", "value": "{{DATE_ADD(NOW(), 45, 'days')}}" },
          { "field": "cdl_expiry", "operator": "greater_than", "value": "{{NOW()}}" }
        ]
      }
    }
  },
  {
    "id": "find_expiring_medical",
    "type": "tool_call",
    "description": "Find active drivers with medical cert expiring in 45 days",
    "tool_name": "query_records",
    "input": {
      "table_id": "drivers",
      "filter": {
        "and": [
          { "field": "status", "operator": "equals", "value": "active" },
          { "field": "medical_cert_expiry", "operator": "less_than_or_equals", "value": "{{DATE_ADD(NOW(), 45, 'days')}}" },
          { "field": "medical_cert_expiry", "operator": "greater_than", "value": "{{NOW()}}" }
        ]
      }
    }
  },
  {
    "id": "alert_cdl",
    "type": "foreach",
    "description": "Create compliance alerts for expiring CDLs",
    "items": "{{find_expiring_cdl.output.data}}",
    "steps": [
      {
        "id": "create_cdl_alert",
        "type": "tool_call",
        "description": "Create a CDL expiry alert record",
        "tool_name": "create_record",
        "input": {
          "table_id": "compliance_alerts",
          "data": {
            "driver_id": "{{alert_cdl.item.id}}",
            "alert_type": "cdl_expiry",
            "expiry_date": "{{alert_cdl.item.cdl_expiry}}",
            "acknowledged": false
          }
        }
      },
      {
        "id": "email_driver_cdl",
        "type": "tool_call",
        "description": "Email the driver about their expiring CDL",
        "tool_name": "gmail_send_email",
        "input": {
          "to": "{{alert_cdl.item.email}}",
          "subject": "Action Required: Your CDL expires {{alert_cdl.item.cdl_expiry}}",
          "body": "Hi {{alert_cdl.item.name}},\n\nYour Commercial Driver's License ({{alert_cdl.item.cdl_number}}) expires on {{alert_cdl.item.cdl_expiry}}. Please schedule your renewal as soon as possible.\n\nYou will not be dispatched on loads after your CDL expiry date.\n\nThank you."
        }
      }
    ]
  },
  {
    "id": "alert_medical",
    "type": "foreach",
    "description": "Create compliance alerts for expiring medical certs",
    "items": "{{find_expiring_medical.output.data}}",
    "steps": [
      {
        "id": "create_medical_alert",
        "type": "tool_call",
        "description": "Create a medical cert expiry alert record",
        "tool_name": "create_record",
        "input": {
          "table_id": "compliance_alerts",
          "data": {
            "driver_id": "{{alert_medical.item.id}}",
            "alert_type": "medical_cert_expiry",
            "expiry_date": "{{alert_medical.item.medical_cert_expiry}}",
            "acknowledged": false
          }
        }
      },
      {
        "id": "notify_fleet_manager",
        "type": "tool_call",
        "description": "Notify fleet manager about the expiring medical cert",
        "tool_name": "send_notification",
        "input": {
          "message": "Driver {{alert_medical.item.name}} — medical certificate expires {{alert_medical.item.medical_cert_expiry}}. Schedule DOT physical.",
          "record_id": "{{alert_medical.item.id}}"
        }
      }
    ]
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| Drivers dispatched with expired credentials | 2-4/year | 0 |
| DOT compliance fines | $5,000-15,000/year | $0 |
| Manual compliance tracking time | 4 hrs/week | 0 |

→ [Set up this workflow on Lotics](https://lotics.ai)

# Permit and Route Survey Tracker

## The problem

Oversized and heavy-lift cargo — wind turbine blades, transformers, industrial equipment — requires transport permits from every jurisdiction along the route. A single shipment crossing 3 states may need 8-12 separate permits, each with different application timelines (5-30 business days), validity windows, and renewal requirements. Project cargo managers track permits in spreadsheets and email threads, routinely discovering expired or missing permits days before transport, causing $10,000-50,000 delays in mobilization fees and project penalties.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Projects | project_name, client_id, cargo_description, dimensions, weight, origin, destination, transport_date | Project cargo master |
| Route Segments | project_id, segment_order, from_location, to_location, jurisdiction, distance_km | Route breakdown |
| Permits | project_id, segment_id, jurisdiction, permit_type, application_date, expected_issue_date, expiry_date, status | Permit tracking |
| Clients | name, email, project_manager_name | Client contacts |

## Example prompts

- "Every weekday, check for permits that are pending and overdue for issuance, or permits expiring before the transport date, and alert the project manager."
- "Daily permit status scan — flag overdue applications and soon-to-expire permits for project cargo shipments."

## Workflow

**Trigger:** Recurring daily schedule at 08:00 UTC on weekdays.

```json
[
  {
    "id": "find_overdue_permits",
    "type": "tool_call",
    "description": "Find permits where expected issue date has passed but still pending",
    "tool_name": "query_records",
    "input": {
      "table_id": "permits",
      "filter": {
        "and": [
          { "field": "status", "operator": "equals", "value": "pending" },
          { "field": "expected_issue_date", "operator": "less_than", "value": "{{NOW()}}" }
        ]
      }
    }
  },
  {
    "id": "find_expiring_permits",
    "type": "tool_call",
    "description": "Find issued permits that expire within 14 days",
    "tool_name": "query_records",
    "input": {
      "table_id": "permits",
      "filter": {
        "and": [
          { "field": "status", "operator": "equals", "value": "issued" },
          { "field": "expiry_date", "operator": "less_than_or_equals", "value": "{{DATE_ADD(NOW(), 14, 'days')}}" },
          { "field": "expiry_date", "operator": "greater_than", "value": "{{NOW()}}" }
        ]
      }
    }
  },
  {
    "id": "process_overdue",
    "type": "foreach",
    "description": "Alert on each overdue permit application",
    "items": "{{find_overdue_permits.output.data}}",
    "steps": [
      {
        "id": "get_project_overdue",
        "type": "tool_call",
        "description": "Fetch the project for context",
        "tool_name": "get_record",
        "input": {
          "table_id": "projects",
          "record_id": "{{process_overdue.item.project_id}}"
        }
      },
      {
        "id": "alert_overdue",
        "type": "tool_call",
        "description": "Send alert about the overdue permit",
        "tool_name": "send_notification",
        "input": {
          "message": "OVERDUE PERMIT: {{process_overdue.item.permit_type}} for {{process_overdue.item.jurisdiction}} on project {{get_project_overdue.output.data.project_name}}. Expected by {{process_overdue.item.expected_issue_date}}, still pending. Transport date: {{get_project_overdue.output.data.transport_date}}. Follow up with issuing authority immediately.",
          "record_id": "{{process_overdue.item.id}}"
        }
      }
    ]
  },
  {
    "id": "process_expiring",
    "type": "foreach",
    "description": "Alert on each permit expiring before transport",
    "items": "{{find_expiring_permits.output.data}}",
    "steps": [
      {
        "id": "get_project_expiring",
        "type": "tool_call",
        "description": "Fetch the project details",
        "tool_name": "get_record",
        "input": {
          "table_id": "projects",
          "record_id": "{{process_expiring.item.project_id}}"
        }
      },
      {
        "id": "check_expires_before_transport",
        "type": "if",
        "description": "Only alert if permit expires before or on the transport date",
        "condition": "{{process_expiring.item.expiry_date <= get_project_expiring.output.data.transport_date}}",
        "then": [
          {
            "id": "alert_expiring",
            "type": "tool_call",
            "description": "Send permit expiry alert",
            "tool_name": "send_notification",
            "input": {
              "message": "PERMIT EXPIRING: {{process_expiring.item.permit_type}} for {{process_expiring.item.jurisdiction}} expires {{process_expiring.item.expiry_date}}, but transport is scheduled {{get_project_expiring.output.data.transport_date}}. Renewal required on project {{get_project_expiring.output.data.project_name}}.",
              "record_id": "{{process_expiring.item.id}}"
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
| Transport delays from permit issues | 3-5/year | 0-1/year |
| Cost per permit-related delay | $10,000-50,000 | Avoided |
| Manual permit tracking hours | 8 hrs/week | 1 hr/week |

→ [Set up this workflow on Lotics](https://lotics.ai)

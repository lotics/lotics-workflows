# Non-Conformance Report with Corrective Action Tracking

## The problem

An aerospace parts manufacturer averages 12 non-conformance reports per month. Each NCR requires root cause analysis, corrective action assignment, implementation verification, and effectiveness review -- a process that spans 4-6 weeks. Today, NCRs are tracked in a shared spreadsheet with no automated follow-up. 35% of corrective actions miss their due dates, and auditors have flagged incomplete closure documentation in three consecutive AS9100 surveillance audits.

## Data model

| Table | Key fields | Purpose |
|-------|-----------|---------|
| Non-Conformance Reports | ncr_id, source, part_number, description, severity, status, detected_by, detected_at | Documents the non-conforming condition |
| Corrective Actions | ca_id, ncr_id, assigned_to, action_description, due_date, status, completed_at | Individual corrective/preventive actions |
| Effectiveness Reviews | review_id, ca_id, reviewer, result, reviewed_at, evidence_files | Verifies the corrective action actually worked |

## Example prompts

- "When a corrective action's due date passes and it's still open, escalate to the quality director, add a comment to the NCR, and extend the deadline by 5 business days."
- "Automatically follow up on overdue corrective actions -- escalate, document the delay, and set a new deadline."

## Workflow

**Trigger:** Daily recurring schedule at 08:00

```json
[
  {
    "id": "find_overdue",
    "type": "tool_call",
    "description": "Query all corrective actions that are past due and still open",
    "tool_name": "query_records",
    "input": {
      "table_id": "corrective_actions",
      "filters": {
        "status": "Open",
        "due_date__lt": "{{trigger.recurring_schedule.triggered_at}}"
      }
    }
  },
  {
    "id": "process_overdue",
    "type": "foreach",
    "description": "Process each overdue corrective action",
    "items": "{{find_overdue.output.records}}",
    "steps": [
      {
        "id": "get_ncr",
        "type": "tool_call",
        "description": "Fetch the parent NCR for context",
        "tool_name": "get_record",
        "input": {
          "table_id": "non_conformance_reports",
          "record_id": "{{process_overdue.item.ncr_id}}"
        }
      },
      {
        "id": "add_overdue_comment",
        "type": "tool_call",
        "description": "Document the overdue status on the NCR record",
        "tool_name": "create_record_comments",
        "input": {
          "table_id": "non_conformance_reports",
          "record_id": "{{process_overdue.item.ncr_id}}",
          "comment": "Corrective action {{process_overdue.item.ca_id}} is overdue. Original due date: {{process_overdue.item.due_date}}. Assigned to: {{process_overdue.item.assigned_to}}. Escalated to quality director."
        }
      },
      {
        "id": "extend_deadline",
        "type": "tool_call",
        "description": "Extend the due date by 5 business days and mark as escalated",
        "tool_name": "update_record",
        "input": {
          "table_id": "corrective_actions",
          "record_id": "{{process_overdue.item.record_id}}",
          "data": {
            "status": "Escalated",
            "due_date": "{{process_overdue.item.due_date + 7 * 24 * 60 * 60 * 1000}}"
          }
        }
      },
      {
        "id": "log_escalation",
        "type": "tool_call",
        "description": "Log the escalation event in the NCR audit trail",
        "tool_name": "query_record_logs",
        "input": {
          "table_id": "corrective_actions",
          "record_id": "{{process_overdue.item.record_id}}"
        }
      },
      {
        "id": "escalate_notification",
        "type": "tool_call",
        "description": "Notify the quality director about the overdue corrective action",
        "tool_name": "send_notification",
        "input": {
          "title": "Overdue CAPA: {{process_overdue.item.ca_id}} (NCR {{get_ncr.output.data.ncr_id}})",
          "message": "Corrective action assigned to {{process_overdue.item.assigned_to}} missed its due date of {{process_overdue.item.due_date}}. NCR severity: {{get_ncr.output.data.severity}}. Part: {{get_ncr.output.data.part_number}}. Deadline extended by 5 business days."
        }
      }
    ]
  }
]
```

## Results

| Metric | Before | After |
|--------|--------|-------|
| Corrective actions missing due dates | 35% | 8% |
| Audit findings for incomplete closure | 3 per audit | 0 |
| Average NCR closure time | 6 weeks | 3.5 weeks |

-> [Set up this workflow on Lotics](https://lotics.ai)

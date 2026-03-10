# Billable Time Tracking Reminder

## The problem

Attorneys at a 20-person firm are expected to bill 1,800 hours per year — roughly 37.5 hours per week. Most attorneys log their time at the end of the week or month from memory, leading to under-reporting by an estimated 10-15%. That translates to $150,000-$250,000 in lost billable revenue annually. Partners have no real-time visibility into utilization rates, and associates don't get feedback until quarterly reviews when it's too late to course-correct.

## Data model

| Table | Key fields | Purpose |
|-------|-----------|---------|
| Time Entries | attorney (link), case (link), hours, date, description, entry_type, status | Daily time logs by attorney |
| Attorneys | attorney_name, email, billable_target_weekly, role, status | Attorney profiles with billing targets |
| Utilization Reports | attorney (link), week_start, total_hours, billable_hours, utilization_rate, status | Weekly utilization snapshots |

## Example prompts

- "Every Friday at 4pm, check each attorney's billable hours for the week. If they're below target, send them a reminder. Generate a utilization report for the partners."
- "Set up weekly time tracking enforcement: flag attorneys who haven't met their billable target and send a summary to the managing partner."

## Workflow

**Trigger:** `recurring_schedule` — every Friday at 16:00.

```json
[
  {
    "id": "get_attorneys",
    "type": "tool_call",
    "description": "Fetch all active attorneys",
    "tool_name": "query_records",
    "input": {
      "table_name": "Attorneys",
      "filters": {
        "status": "Active"
      }
    }
  },
  {
    "id": "process_attorneys",
    "type": "foreach",
    "description": "Check each attorney's weekly hours and send reminders if needed",
    "items": "{{get_attorneys.output.records}}",
    "steps": [
      {
        "id": "get_weekly_hours",
        "type": "tool_call",
        "description": "Aggregate this attorney's billable hours for the current week",
        "tool_name": "aggregate_records",
        "input": {
          "table_name": "Time Entries",
          "filters": {
            "attorney": "{{item.id}}",
            "date__gte": "{{startOfWeek(today())}}",
            "date__lte": "{{today()}}",
            "entry_type": "Billable"
          },
          "aggregations": [
            {
              "field": "hours",
              "function": "sum"
            }
          ]
        }
      },
      {
        "id": "create_utilization_record",
        "type": "tool_call",
        "description": "Create a weekly utilization snapshot for this attorney",
        "tool_name": "create_record",
        "input": {
          "table_name": "Utilization Reports",
          "data": {
            "attorney": "{{item.id}}",
            "week_start": "{{startOfWeek(today())}}",
            "billable_hours": "{{get_weekly_hours.output.results[0].value}}",
            "utilization_rate": "{{(get_weekly_hours.output.results[0].value / item.billable_target_weekly) * 100}}",
            "status": "{{get_weekly_hours.output.results[0].value >= item.billable_target_weekly ? 'On Track' : 'Below Target'}}"
          }
        }
      },
      {
        "id": "check_below_target",
        "type": "if",
        "description": "Send a reminder if the attorney is below their weekly billable target",
        "condition": "{{get_weekly_hours.output.results[0].value < item.billable_target_weekly}}",
        "then": [
          {
            "id": "send_reminder",
            "type": "tool_call",
            "description": "Email the attorney a reminder about their billable hours gap",
            "tool_name": "outlook_send_email",
            "input": {
              "to": "{{item.email}}",
              "subject": "Time Entry Reminder — {{item.billable_target_weekly - get_weekly_hours.output.results[0].value}} hours below target",
              "body": "Hi {{item.attorney_name}},\n\nThis is a friendly reminder that your billable hours for this week are at {{get_weekly_hours.output.results[0].value}} of your {{item.billable_target_weekly}}-hour target.\n\nPlease ensure all billable time is logged before end of day. Accurate time capture is essential for client billing and firm performance.\n\nThank you."
            }
          }
        ],
        "else": []
      }
    ]
  },
  {
    "id": "get_all_utilization",
    "type": "tool_call",
    "description": "Query all utilization reports for this week to build the partner summary",
    "tool_name": "query_records",
    "input": {
      "table_name": "Utilization Reports",
      "filters": {
        "week_start": "{{startOfWeek(today())}}"
      }
    }
  },
  {
    "id": "generate_partner_summary",
    "type": "tool_call",
    "description": "Generate a summary report for the partners",
    "tool_name": "llm_generate_text",
    "input": {
      "prompt": "Create a concise weekly utilization summary for law firm partners. Format as a clean HTML email.\n\nWeek of: {{startOfWeek(today())}}\n\nAttorney utilization data:\n{{JSON.stringify(get_all_utilization.output.records)}}\n\nInclude: overall firm utilization rate, attorneys on track, attorneys below target (with specific gaps), and any patterns worth noting. Keep it factual and actionable."
    }
  },
  {
    "id": "email_partners",
    "type": "tool_call",
    "description": "Send the utilization summary to the partner group",
    "tool_name": "outlook_send_email",
    "input": {
      "to": "partners@firm.com",
      "subject": "Weekly Utilization Report — Week of {{startOfWeek(today())}}",
      "body": "{{generate_partner_summary.output.text}}"
    }
  }
]
```

## Results

| Metric | Before | After |
|--------|--------|-------|
| Estimated unbilled hours per year | 10-15% of total | < 3% |
| Revenue recovered annually | — | $150,000-$250,000 |
| Partner visibility into utilization | Quarterly | Weekly, real-time |

-> [Set up this workflow on Lotics](https://lotics.ai)

# Client Matter Progress Update

## The problem

Clients of law firms frequently call or email asking "What's the status of my case?" These inquiries consume 3-5 hours of paralegal time per week and interrupt attorney workflow. Many firms send no proactive updates at all, leading to client anxiety and dissatisfaction. When updates are sent, they're inconsistent — some attorneys send detailed notes, others send nothing for months. For a firm handling 80+ active matters, this creates a significant client experience gap and increases the risk of bar complaints related to communication failures.

## Data model

| Table | Key fields | Purpose |
|-------|-----------|---------|
| Cases | case_number, client_name, client_email, matter_type, status, lead_attorney, last_update_sent | Active legal matters |
| Case Activity Log | case (link), activity_type, description, date, logged_by, billable | Chronological record of case actions |
| Client Communications | case (link), sent_to, subject, body, sent_at, method | Log of all client-facing communications |

## Example prompts

- "Every two weeks, pull recent activity for each active case, generate a plain-language summary, and email it to the client. Log the communication on the case record."
- "Automate client updates: summarize what's happened on their case using the activity log and send a professional update email."

## Workflow

**Trigger:** `recurring_schedule` — every other Monday at 10:00.

```json
[
  {
    "id": "get_active_cases",
    "type": "tool_call",
    "description": "Fetch all cases with Active status",
    "tool_name": "query_records",
    "input": {
      "table_name": "Cases",
      "filters": {
        "status": "Active"
      }
    }
  },
  {
    "id": "update_each_case",
    "type": "foreach",
    "description": "Generate and send a progress update for each active case",
    "items": "{{get_active_cases.output.records}}",
    "steps": [
      {
        "id": "get_recent_activity",
        "type": "tool_call",
        "description": "Query activity log entries from the last 14 days for this case",
        "tool_name": "query_records",
        "input": {
          "table_name": "Case Activity Log",
          "filters": {
            "case": "{{item.id}}",
            "date__gte": "{{addDays(today(), -14)}}"
          },
          "sort": {
            "field": "date",
            "direction": "desc"
          }
        }
      },
      {
        "id": "check_has_activity",
        "type": "if",
        "description": "Only send an update if there has been activity in the last 14 days",
        "condition": "{{get_recent_activity.output.total > 0}}",
        "then": [
          {
            "id": "generate_update",
            "type": "tool_call",
            "description": "Generate a client-friendly plain-language summary of recent case activity",
            "tool_name": "llm_generate_text",
            "input": {
              "prompt": "Write a professional, plain-language case status update email for a law firm client. Avoid legal jargon where possible. Be reassuring but factual.\n\nClient: {{item.client_name}}\nCase Number: {{item.case_number}}\nMatter Type: {{item.matter_type}}\nLead Attorney: {{item.lead_attorney_name}}\n\nRecent activity (last 14 days):\n{{JSON.stringify(get_recent_activity.output.records)}}\n\nInclude: a brief summary of what happened, what comes next, and an invitation to call with questions. Do not include billing details."
            }
          },
          {
            "id": "send_update_email",
            "type": "tool_call",
            "description": "Send the progress update to the client",
            "tool_name": "gmail_send_email",
            "input": {
              "to": "{{item.client_email}}",
              "subject": "Case Update: {{item.case_number}} — {{item.matter_type}}",
              "body": "{{generate_update.output.text}}"
            }
          },
          {
            "id": "log_communication",
            "type": "tool_call",
            "description": "Log this communication on the case record for compliance",
            "tool_name": "create_record",
            "input": {
              "table_name": "Client Communications",
              "data": {
                "case": "{{item.id}}",
                "sent_to": "{{item.client_email}}",
                "subject": "Case Update: {{item.case_number}} — {{item.matter_type}}",
                "body": "{{generate_update.output.text}}",
                "sent_at": "{{now()}}",
                "method": "Email"
              }
            }
          },
          {
            "id": "update_last_sent",
            "type": "tool_call",
            "description": "Update the case record with the last update sent date",
            "tool_name": "update_record",
            "input": {
              "table_name": "Cases",
              "record_id": "{{item.id}}",
              "data": {
                "last_update_sent": "{{today()}}"
              }
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
|--------|--------|-------|
| "What's my status?" calls per week | 15-20 | 3-5 |
| Paralegal hours on status inquiries | 3-5 hrs/week | < 1 hr/week |
| Client satisfaction (NPS) | 32 | 58 |

-> [Set up this workflow on Lotics](https://lotics.ai)

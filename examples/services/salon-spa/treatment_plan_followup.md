# Treatment Plan Follow-Up Reminder

## The problem

Aesthetic clinics sell treatment plans spanning 4-8 sessions (e.g., laser hair removal, chemical peels). After the first 2 sessions, 35% of clients drop off because no one follows up to schedule the next appointment. The revenue loss is significant: a 6-session laser package at $200/session means $800 left on the table per dropout. Staff don't have time to manually track who is overdue for their next session.

## Data model

| Table | Key fields | Purpose |
|-------|-----------|---------|
| Treatment Plans | client_id, plan_name, total_sessions, completed_sessions, next_session_due, status | Multi-session treatment tracking |
| Clients | full_name, phone, email | Client contact information |
| Appointments | client_id, treatment_plan_id, date_time, status | Individual session records |

## Example prompts

- "When a treatment session is completed, check if the client has remaining sessions on their plan. If they do and no next appointment is booked, send them a reminder email 3 days later to schedule."
- "After completing a treatment appointment, automatically follow up with clients who haven't booked their next session within 3 days."

## Workflow

**Trigger:** When an appointment status is updated to "completed"

```json
[
  {
    "id": "check_plan",
    "type": "tool_call",
    "description": "Fetch the associated treatment plan",
    "tool_name": "get_record",
    "input": {
      "table_name": "Treatment Plans",
      "record_id": "{{trigger.record_updated.next_data.treatment_plan_id}}"
    }
  },
  {
    "id": "has_remaining_sessions",
    "type": "if",
    "description": "Check if client has remaining sessions on the plan",
    "condition": "{{check_plan.output.completed_sessions < check_plan.output.total_sessions}}",
    "then": [
      {
        "id": "update_plan",
        "type": "tool_call",
        "description": "Increment completed sessions count on the plan",
        "tool_name": "update_record",
        "input": {
          "table_name": "Treatment Plans",
          "record_id": "{{check_plan.output.id}}",
          "data": {
            "completed_sessions": "{{check_plan.output.completed_sessions + 1}}"
          }
        }
      },
      {
        "id": "wait_3_days",
        "type": "wait",
        "description": "Wait 3 days before checking if next session is booked",
        "duration": "P3D"
      },
      {
        "id": "check_next_booking",
        "type": "tool_call",
        "description": "Look for any upcoming appointments on this plan",
        "tool_name": "query_records",
        "input": {
          "table_name": "Appointments",
          "filters": {
            "treatment_plan_id": "{{check_plan.output.id}}",
            "status": "scheduled"
          }
        }
      },
      {
        "id": "no_booking_found",
        "type": "if",
        "description": "If no upcoming appointment exists, send a reminder",
        "condition": "{{check_next_booking.output.records.length === 0}}",
        "then": [
          {
            "id": "get_client",
            "type": "tool_call",
            "description": "Fetch client details for the reminder email",
            "tool_name": "get_record",
            "input": {
              "table_name": "Clients",
              "record_id": "{{trigger.record_updated.next_data.client_id}}"
            }
          },
          {
            "id": "send_reminder",
            "type": "tool_call",
            "description": "Email the client to schedule their next session",
            "tool_name": "gmail_send_email",
            "input": {
              "to": "{{get_client.output.email}}",
              "subject": "Time to book your next {{check_plan.output.plan_name}} session",
              "body": "Hi {{get_client.output.full_name}},\n\nGreat progress on your {{check_plan.output.plan_name}} plan! You've completed {{update_plan.output.completed_sessions}} of {{check_plan.output.total_sessions}} sessions.\n\nWe noticed you haven't scheduled your next session yet. For best results, we recommend booking within the next week.\n\nReply to this email or call us to book your next appointment.\n\nLooking forward to seeing you!"
            }
          }
        ],
        "else": []
      }
    ],
    "else": []
  }
]
```

## Results

| Metric | Before | After |
|--------|--------|-------|
| Treatment plan completion rate | 65% | 88% |
| Revenue recovered per month | $0 | ~$4,800 (from retained dropoffs) |
| Staff time on follow-up calls | 5 hrs/week | 0 hrs/week |

-> [Set up this workflow on Lotics](https://lotics.ai)

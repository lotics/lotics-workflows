# Tuition Payment Overdue Alert

## The problem

A language school with 150 students collects tuition monthly. The accountant manually cross-references bank statements with the enrollment list every week to find who hasn't paid. Students 2-3 weeks overdue often claim they "forgot" or "didn't know when it was due." By the time the school follows up, some students have quietly dropped out. The school loses an average of $3,000/month in uncollected tuition because follow-up happens too late.

## Data model

| Table | Key fields | Purpose |
|-------|-----------|---------|
| Tuition Payments | student_id, enrollment_id, amount_due, amount_paid, due_date, status | Monthly payment tracking |
| Students | full_name, email, phone | Student contact info |
| Enrollments | student_id, class_id, status | Active enrollment records |

## Example prompts

- "Every Monday morning, find tuition payments that are overdue by more than 3 days and send reminder emails to those students."
- "Check for overdue tuition payments weekly and notify students who haven't paid, including the amount due and how many days late they are."

## Workflow

**Trigger:** Recurring schedule, every Monday at 09:00

```json
[
  {
    "id": "find_overdue",
    "type": "tool_call",
    "description": "Query tuition payments that are past due and unpaid",
    "tool_name": "query_records",
    "input": {
      "table_name": "Tuition Payments",
      "filters": {
        "status": "unpaid",
        "due_date_before": "{{trigger.recurring_schedule.scheduled_at_date_minus_3d}}"
      }
    }
  },
  {
    "id": "process_overdue",
    "type": "foreach",
    "description": "Send reminder to each student with overdue tuition",
    "items": "{{find_overdue.output.records}}",
    "steps": [
      {
        "id": "get_student",
        "type": "tool_call",
        "description": "Fetch student contact details",
        "tool_name": "get_record",
        "input": {
          "table_name": "Students",
          "record_id": "{{item.student_id}}"
        }
      },
      {
        "id": "send_overdue_email",
        "type": "tool_call",
        "description": "Send overdue tuition reminder email",
        "tool_name": "gmail_send_email",
        "input": {
          "to": "{{get_student.output.email}}",
          "subject": "Tuition payment overdue - Action required",
          "body": "Hi {{get_student.output.full_name}},\n\nOur records show that your tuition payment of ${{item.amount_due}} was due on {{item.due_date}} and remains unpaid.\n\nPlease arrange payment at your earliest convenience to avoid any interruption to your classes.\n\nIf you've already made this payment, please reply to this email with the transfer confirmation so we can update our records.\n\nThank you."
        }
      },
      {
        "id": "log_reminder",
        "type": "tool_call",
        "description": "Add a comment to the payment record tracking the reminder",
        "tool_name": "create_record_comments",
        "input": {
          "table_name": "Tuition Payments",
          "record_id": "{{item.id}}",
          "comment": "Automated overdue reminder sent to {{get_student.output.email}}"
        }
      }
    ]
  }
]
```

## Results

| Metric | Before | After |
|--------|--------|-------|
| Overdue payments caught within 1 week | 40% | 100% |
| Average days to payment after due date | 18 days | 5 days |
| Monthly uncollected tuition | $3,000 | $400 |

-> [Set up this workflow on Lotics](https://lotics.ai)

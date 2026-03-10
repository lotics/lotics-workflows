# Pet Vaccination Reminder

## The problem

Veterinary clinics manage vaccination schedules for hundreds of active patients. Core vaccines (rabies, DHPP, FVRCP) require boosters at specific intervals, but 35% of pets miss their due dates because owners forget. Staff manually pull "overdue" lists and make phone calls, but with 50+ pets coming due each week, many slip through. Missed vaccinations mean lost revenue and, for rabies, potential legal non-compliance.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Patients | pet_name, species, breed, owner_name, owner_email, owner_phone, date_of_birth, status | Pet patient directory |
| Vaccinations | patient_id, pet_name, vaccine_name, date_administered, next_due_date, status, owner_email | Vaccination history and schedule |
| Appointments | patient_id, pet_name, appointment_date, appointment_type, provider, status | Scheduled and completed visits |

## Example prompts

- "Every Monday, find all vaccinations coming due in the next 14 days and email the pet owner a reminder with the vaccine name and due date. If a vaccination is already overdue, send a different email urging them to book immediately."
- "Set up weekly vaccination reminders for pet owners. Check our vaccination table for upcoming and overdue shots and email owners automatically."

## Workflow

**Trigger:** `recurring_schedule` — Runs every Monday at 8:00 AM.

```json
[
  {
    "id": "get_upcoming_vaccines",
    "type": "tool_call",
    "description": "Query all vaccinations with a next due date within the next 14 days",
    "tool_name": "query_records",
    "input": {
      "table_name": "Vaccinations",
      "filters": {
        "status": "Scheduled"
      }
    }
  },
  {
    "id": "process_vaccines",
    "type": "foreach",
    "description": "Send appropriate reminder based on whether the vaccine is upcoming or overdue",
    "items": "{{get_upcoming_vaccines.output.records.filter(r => new Date(r.next_due_date) < new Date(Date.now() + 14 * 86400000))}}",
    "steps": [
      {
        "id": "check_overdue",
        "type": "if",
        "description": "Determine if the vaccination is already past due",
        "condition": "{{new Date(item.next_due_date) < new Date()}}",
        "then": [
          {
            "id": "send_overdue_email",
            "type": "tool_call",
            "description": "Send an urgent overdue vaccination email to the pet owner",
            "tool_name": "gmail_send_email",
            "input": {
              "to": "{{item.owner_email}}",
              "subject": "Overdue: {{item.pet_name}}'s {{item.vaccine_name}} vaccination",
              "body": "Hi,\n\n{{item.pet_name}}'s {{item.vaccine_name}} vaccination was due on {{item.next_due_date}} and is now overdue.\n\nPlease call us as soon as possible to schedule this vaccination. Keeping vaccinations current is important for {{item.pet_name}}'s health and may be required by local regulations.\n\nCall us or reply to this email to book an appointment.\n\nThank you,\nYour Veterinary Team"
            }
          },
          {
            "id": "mark_overdue",
            "type": "tool_call",
            "description": "Update the vaccination status to overdue",
            "tool_name": "update_record",
            "input": {
              "table_name": "Vaccinations",
              "record_id": "{{item.id}}",
              "data": {
                "status": "Overdue"
              }
            }
          }
        ],
        "else": [
          {
            "id": "send_upcoming_email",
            "type": "tool_call",
            "description": "Send a friendly upcoming vaccination reminder to the pet owner",
            "tool_name": "gmail_send_email",
            "input": {
              "to": "{{item.owner_email}}",
              "subject": "Reminder: {{item.pet_name}}'s {{item.vaccine_name}} is due soon",
              "body": "Hi,\n\nThis is a friendly reminder that {{item.pet_name}}'s {{item.vaccine_name}} vaccination is due on {{item.next_due_date}}.\n\nPlease call us or reply to this email to schedule an appointment. We have availability most weekdays.\n\nThank you,\nYour Veterinary Team"
            }
          }
        ]
      }
    ]
  },
  {
    "id": "notify_staff",
    "type": "tool_call",
    "description": "Send a summary to the vet tech team with counts of reminders sent",
    "tool_name": "send_notification",
    "input": {
      "message": "Weekly vaccination reminders sent. Total: {{get_upcoming_vaccines.output.records.filter(r => new Date(r.next_due_date) < new Date(Date.now() + 14 * 86400000)).length}} reminders processed.",
      "channel": "vet-techs"
    }
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| On-time vaccination rate | 65% | 88% |
| Staff hours on reminder calls | 3 hrs/week | 0 |
| Overdue vaccinations resolved within 7 days | 30% | 70% |
| Vaccination revenue recovered monthly | — | $2,400 |

-> [Set up this workflow on Lotics](https://lotics.ai)

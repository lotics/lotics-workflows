# Appointment Reminder with No-Show Follow-Up

## The problem

Clinics lose 15-20% of scheduled appointments to no-shows. Front desk staff manually call patients the day before, but with 40+ appointments daily, reminders slip through. When a patient doesn't show up, there's no systematic follow-up to reschedule, and the open slot goes unfilled.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Appointments | patient_name, appointment_date, appointment_time, provider, status, reminder_sent, patient_email | Tracks all scheduled visits |
| Patients | full_name, email, phone, preferred_contact, no_show_count | Patient contact directory |

## Example prompts

- "When an appointment is created, send the patient an email reminder with the date, time, and provider name. If the appointment status changes to no-show, send a follow-up email asking them to reschedule."
- "Set up a workflow that emails patients when they get a new appointment and follows up automatically if they don't show up."

## Workflow

**Trigger:** When an appointment record is updated (status changes to "scheduled" or "no_show").

```json
[
  {
    "id": "check_status",
    "type": "switch",
    "description": "Route based on appointment status change",
    "value": "{{trigger.record_updated.next_data.status}}",
    "cases": {
      "scheduled": [
        {
          "id": "send_reminder",
          "type": "tool_call",
          "description": "Send appointment confirmation email to patient",
          "tool_name": "gmail_send_email",
          "input": {
            "to": "{{trigger.record_updated.next_data.patient_email}}",
            "subject": "Your appointment on {{trigger.record_updated.next_data.appointment_date}}",
            "body": "Hi {{trigger.record_updated.next_data.patient_name}},\n\nThis is a reminder that you have an appointment scheduled:\n\nDate: {{trigger.record_updated.next_data.appointment_date}}\nTime: {{trigger.record_updated.next_data.appointment_time}}\nProvider: {{trigger.record_updated.next_data.provider}}\n\nPlease reply to this email if you need to reschedule.\n\nThank you."
          }
        },
        {
          "id": "mark_reminder_sent",
          "type": "tool_call",
          "description": "Flag the appointment as reminder sent",
          "tool_name": "update_record",
          "input": {
            "table_name": "Appointments",
            "record_id": "{{trigger.record_updated.record_id}}",
            "data": {
              "reminder_sent": true
            }
          }
        }
      ],
      "no_show": [
        {
          "id": "get_patient",
          "type": "tool_call",
          "description": "Look up patient to update no-show count",
          "tool_name": "query_records",
          "input": {
            "table_name": "Patients",
            "filters": {
              "full_name": "{{trigger.record_updated.next_data.patient_name}}"
            }
          }
        },
        {
          "id": "update_no_show_count",
          "type": "tool_call",
          "description": "Increment the patient no-show counter",
          "tool_name": "update_record",
          "input": {
            "table_name": "Patients",
            "record_id": "{{get_patient.output.records[0].id}}",
            "data": {
              "no_show_count": "{{get_patient.output.records[0].no_show_count + 1}}"
            }
          }
        },
        {
          "id": "send_noshow_followup",
          "type": "tool_call",
          "description": "Email the patient asking them to reschedule",
          "tool_name": "gmail_send_email",
          "input": {
            "to": "{{trigger.record_updated.next_data.patient_email}}",
            "subject": "We missed you today - please reschedule",
            "body": "Hi {{trigger.record_updated.next_data.patient_name}},\n\nWe noticed you were unable to make your appointment on {{trigger.record_updated.next_data.appointment_date}} at {{trigger.record_updated.next_data.appointment_time}} with {{trigger.record_updated.next_data.provider}}.\n\nPlease reply to this email or call us to reschedule at your earliest convenience.\n\nThank you."
          }
        }
      ]
    }
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| No-show rate | 18% | 9% |
| Manual reminder calls per day | 40+ | 0 |
| Rescheduled no-shows | 15% | 55% |
| Staff hours on phone reminders | 2 hrs/day | 0 |

-> [Set up this workflow on Lotics](https://lotics.ai)

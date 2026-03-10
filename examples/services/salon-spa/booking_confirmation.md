# Booking Confirmation & Prep Notification

## The problem

A spa handling 40+ appointments daily loses 15-20% to no-shows because confirmation messages go out late or not at all. Front-desk staff manually text clients the night before, but forget during busy periods. Therapists arrive unprepared because they don't know which treatment room is assigned or what products to stage.

## Data model

| Table | Key fields | Purpose |
|-------|-----------|---------|
| Appointments | client_name, service_type, therapist, date_time, status, room, duration_minutes | Tracks every booking |
| Clients | full_name, phone, email, preferred_therapist, allergies, notes | Client profiles and preferences |
| Services | service_name, category, duration, price, prep_requirements | Treatment catalog with prep details |

## Example prompts

- "When a new appointment is booked, email the client a confirmation with the service name, date, time, and therapist. Also notify the assigned therapist with the room number and any client allergy notes."
- "Send a booking confirmation to the client and a prep checklist to the therapist every time an appointment status changes to confirmed."

## Workflow

**Trigger:** When an appointment record is updated and status becomes "confirmed"

```json
[
  {
    "id": "get_client",
    "type": "tool_call",
    "description": "Fetch the client record for contact details and allergy info",
    "tool_name": "get_record",
    "input": {
      "table_name": "Clients",
      "record_id": "{{trigger.record_updated.next_data.client_id}}"
    }
  },
  {
    "id": "get_service",
    "type": "tool_call",
    "description": "Fetch the service record for prep requirements",
    "tool_name": "get_record",
    "input": {
      "table_name": "Services",
      "record_id": "{{trigger.record_updated.next_data.service_id}}"
    }
  },
  {
    "id": "email_client",
    "type": "tool_call",
    "description": "Send booking confirmation email to the client",
    "tool_name": "gmail_send_email",
    "input": {
      "to": "{{get_client.output.email}}",
      "subject": "Your appointment is confirmed - {{get_service.output.service_name}}",
      "body": "Hi {{get_client.output.full_name}},\n\nYour appointment has been confirmed:\n\nService: {{get_service.output.service_name}}\nDate & Time: {{trigger.record_updated.next_data.date_time}}\nTherapist: {{trigger.record_updated.next_data.therapist}}\nDuration: {{get_service.output.duration}} minutes\n\nPlease arrive 10 minutes early. If you need to reschedule, reply to this email at least 24 hours in advance.\n\nSee you soon!"
    }
  },
  {
    "id": "notify_therapist",
    "type": "tool_call",
    "description": "Send prep notification to the assigned therapist",
    "tool_name": "send_notification",
    "input": {
      "message": "Upcoming appointment confirmed: {{get_client.output.full_name}} for {{get_service.output.service_name}} at {{trigger.record_updated.next_data.date_time}}. Room: {{trigger.record_updated.next_data.room}}. Allergies: {{get_client.output.allergies}}. Prep: {{get_service.output.prep_requirements}}",
      "member_id": "{{trigger.record_updated.next_data.therapist_member_id}}"
    }
  }
]
```

## Results

| Metric | Before | After |
|--------|--------|-------|
| No-show rate | 18% | 6% |
| Therapist prep time | 15 min (hunting for info) | 2 min (info pushed automatically) |
| Front-desk time on confirmations | 90 min/day | 0 min/day |

-> [Set up this workflow on Lotics](https://lotics.ai)

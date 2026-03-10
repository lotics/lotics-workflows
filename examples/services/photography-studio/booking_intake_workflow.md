# Booking Intake & Shot List Preparation

## The problem

A photography studio books 8-12 sessions per week across weddings, portraits, corporate headshots, and product shoots. When a client fills out the booking form, the studio manager manually creates a project record, sends a confirmation email, and asks the photographer to prepare a shot list. Details get lost between the form submission and the actual shoot day. Photographers arrive without knowing the client's vision, preferred poses, or venue details, leading to wasted time on-site and reshoots.

## Data model

| Table | Key fields | Purpose |
|-------|-----------|---------|
| Projects | client_name, client_email, shoot_type, shoot_date, location, photographer_id, status, special_requests, budget | Photography project records |
| Photographers | full_name, email, specialization, availability | Photographer roster |

## Example prompts

- "When a client submits the booking form, create a project record, email the client a confirmation with shoot details, and notify the assigned photographer with the client's special requests."
- "Automatically process new photography booking submissions: create the project, confirm with the client, and brief the photographer."

## Workflow

**Trigger:** Form submitted (Photography Booking Form)

```json
[
  {
    "id": "create_project",
    "type": "tool_call",
    "description": "Create a new project record from the booking form data",
    "tool_name": "create_record",
    "input": {
      "table_name": "Projects",
      "data": {
        "client_name": "{{trigger.form_submitted.data.client_name}}",
        "client_email": "{{trigger.form_submitted.data.client_email}}",
        "shoot_type": "{{trigger.form_submitted.data.shoot_type}}",
        "shoot_date": "{{trigger.form_submitted.data.shoot_date}}",
        "location": "{{trigger.form_submitted.data.location}}",
        "photographer_id": "{{trigger.form_submitted.data.photographer_id}}",
        "special_requests": "{{trigger.form_submitted.data.special_requests}}",
        "budget": "{{trigger.form_submitted.data.budget}}",
        "status": "confirmed"
      }
    }
  },
  {
    "id": "get_photographer",
    "type": "tool_call",
    "description": "Fetch photographer details for the confirmation",
    "tool_name": "get_record",
    "input": {
      "table_name": "Photographers",
      "record_id": "{{trigger.form_submitted.data.photographer_id}}"
    }
  },
  {
    "id": "confirm_client",
    "type": "tool_call",
    "description": "Send booking confirmation to the client",
    "tool_name": "gmail_send_email",
    "input": {
      "to": "{{trigger.form_submitted.data.client_email}}",
      "subject": "Booking Confirmed - {{trigger.form_submitted.data.shoot_type}} Session on {{trigger.form_submitted.data.shoot_date}}",
      "body": "Hi {{trigger.form_submitted.data.client_name}},\n\nYour photography session is confirmed!\n\nShoot Type: {{trigger.form_submitted.data.shoot_type}}\nDate: {{trigger.form_submitted.data.shoot_date}}\nLocation: {{trigger.form_submitted.data.location}}\nPhotographer: {{get_photographer.output.full_name}}\n\nWe've noted your requests: {{trigger.form_submitted.data.special_requests}}\n\nWe'll send you a detailed prep guide 3 days before your shoot. If you have reference photos or a mood board, reply to this email with them.\n\nLooking forward to working with you!"
    }
  },
  {
    "id": "brief_photographer",
    "type": "tool_call",
    "description": "Send shoot brief to the assigned photographer",
    "tool_name": "gmail_send_email",
    "input": {
      "to": "{{get_photographer.output.email}}",
      "subject": "New Booking: {{trigger.form_submitted.data.shoot_type}} - {{trigger.form_submitted.data.shoot_date}}",
      "body": "Hi {{get_photographer.output.full_name}},\n\nYou've been assigned a new shoot:\n\nClient: {{trigger.form_submitted.data.client_name}}\nType: {{trigger.form_submitted.data.shoot_type}}\nDate: {{trigger.form_submitted.data.shoot_date}}\nLocation: {{trigger.form_submitted.data.location}}\nBudget: ${{trigger.form_submitted.data.budget}}\n\nClient's Special Requests:\n{{trigger.form_submitted.data.special_requests}}\n\nPlease prepare your shot list and equipment checklist. Add any notes to the project record."
    }
  }
]
```

## Results

| Metric | Before | After |
|--------|--------|-------|
| Booking-to-confirmation time | 2-4 hours | Instant |
| Photographer prep issues on shoot day | 3-4/month | Near zero |
| Admin time per booking | 25 min | 0 min |

-> [Set up this workflow on Lotics](https://lotics.ai)

# Placement Onboarding Checklist

## The problem

When a staffing agency places a candidate, the onboarding process involves 8-12 steps: generating an offer letter, collecting signed documents, running background checks, coordinating a start date with the client, and setting up payroll. These steps span multiple people — the recruiter, the back office, the candidate, and the client. Tasks are tracked in email threads and sticky notes. At a firm making 15-20 placements per month, 25% of start dates get delayed because a document wasn't collected or a background check wasn't initiated on time. Each delayed start costs the agency an average of $2,000 in lost billing days.

## Data model

| Table | Key fields | Purpose |
|-------|-----------|---------|
| Placements | candidate (link), job_order (link), start_date, pay_rate, bill_rate, status, offer_letter_file, background_check_status | Confirmed placements linking candidates to positions |
| Onboarding Tasks | placement (link), task_name, assigned_to, due_date, status, completed_at | Checklist items for each placement |
| Job Orders | job_title, client_company, client_contact_email, recruiter | Open positions |

## Example prompts

- "When a placement status changes to 'Confirmed', generate an offer letter, create all onboarding tasks with deadlines, and notify everyone involved of their responsibilities."
- "Automate placement onboarding: create the checklist, generate the offer letter PDF, email it to the candidate for signature, and track completion of every step."

## Workflow

**Trigger:** `button_pressed` — recruiter clicks "Start Onboarding" on a confirmed placement record.

```json
[
  {
    "id": "get_placement",
    "type": "tool_call",
    "description": "Fetch the full placement record",
    "tool_name": "get_record",
    "input": {
      "table_name": "Placements",
      "record_id": "{{trigger.button_pressed.record_id}}"
    }
  },
  {
    "id": "get_job_order",
    "type": "tool_call",
    "description": "Fetch the job order details for the offer letter",
    "tool_name": "get_record",
    "input": {
      "table_name": "Job Orders",
      "record_id": "{{get_placement.output.job_order}}"
    }
  },
  {
    "id": "generate_offer_letter",
    "type": "tool_call",
    "description": "Generate the offer letter PDF from the template",
    "tool_name": "generate_pdf_from_pdf_form_template",
    "input": {
      "template_name": "offer_letter",
      "data": {
        "candidate_name": "{{get_placement.output.candidate_name}}",
        "job_title": "{{get_job_order.output.job_title}}",
        "client_company": "{{get_job_order.output.client_company}}",
        "start_date": "{{get_placement.output.start_date}}",
        "pay_rate": "{{get_placement.output.pay_rate}}",
        "offer_date": "{{today()}}"
      }
    }
  },
  {
    "id": "attach_offer_letter",
    "type": "tool_call",
    "description": "Attach the offer letter to the placement record",
    "tool_name": "add_files_to_record",
    "input": {
      "table_name": "Placements",
      "record_id": "{{trigger.button_pressed.record_id}}",
      "field_name": "offer_letter_file",
      "file_ids": ["{{generate_offer_letter.output.file_id}}"]
    }
  },
  {
    "id": "create_onboarding_tasks",
    "type": "foreach",
    "description": "Create each onboarding checklist task with calculated deadlines",
    "items": [
      { "task_name": "Send offer letter for signature", "days_before_start": 14, "assign_to": "recruiter" },
      { "task_name": "Collect signed offer letter", "days_before_start": 10, "assign_to": "recruiter" },
      { "task_name": "Initiate background check", "days_before_start": 12, "assign_to": "back_office" },
      { "task_name": "Collect I-9 and W-4 forms", "days_before_start": 10, "assign_to": "back_office" },
      { "task_name": "Confirm background check clearance", "days_before_start": 5, "assign_to": "back_office" },
      { "task_name": "Set up payroll", "days_before_start": 5, "assign_to": "back_office" },
      { "task_name": "Confirm start date with client", "days_before_start": 3, "assign_to": "recruiter" },
      { "task_name": "Send first-day instructions to candidate", "days_before_start": 1, "assign_to": "recruiter" }
    ],
    "steps": [
      {
        "id": "create_task",
        "type": "tool_call",
        "description": "Create an onboarding task record with the calculated due date",
        "tool_name": "create_record",
        "input": {
          "table_name": "Onboarding Tasks",
          "data": {
            "placement": "{{trigger.button_pressed.record_id}}",
            "task_name": "{{item.task_name}}",
            "assigned_to": "{{item.assign_to}}",
            "due_date": "{{addDays(get_placement.output.start_date, -1 * item.days_before_start)}}",
            "status": "Pending"
          }
        }
      }
    ]
  },
  {
    "id": "update_placement_status",
    "type": "tool_call",
    "description": "Update placement status to Onboarding",
    "tool_name": "update_record",
    "input": {
      "table_name": "Placements",
      "record_id": "{{trigger.button_pressed.record_id}}",
      "data": {
        "status": "Onboarding"
      }
    }
  },
  {
    "id": "email_offer_to_candidate",
    "type": "tool_call",
    "description": "Email the offer letter to the candidate for review and signature",
    "tool_name": "outlook_send_email",
    "input": {
      "to": "{{get_placement.output.candidate_email}}",
      "subject": "Offer Letter — {{get_job_order.output.job_title}} at {{get_job_order.output.client_company}}",
      "body": "Hi {{get_placement.output.candidate_name}},\n\nCongratulations! Please find your offer letter attached for the {{get_job_order.output.job_title}} position at {{get_job_order.output.client_company}}.\n\nStart Date: {{get_placement.output.start_date}}\nPay Rate: ${{get_placement.output.pay_rate}}/hr\n\nPlease review, sign, and return within 3 business days. We will also need your I-9 and W-4 forms — instructions to follow.\n\nWelcome aboard!\nRecruitment Team"
    }
  },
  {
    "id": "notify_client",
    "type": "tool_call",
    "description": "Notify the client that onboarding has started and confirm the start date",
    "tool_name": "outlook_send_email",
    "input": {
      "to": "{{get_job_order.output.client_contact_email}}",
      "subject": "Placement Confirmed: {{get_placement.output.candidate_name}} — {{get_job_order.output.job_title}}",
      "body": "Hi,\n\nWe're pleased to confirm that {{get_placement.output.candidate_name}} has been placed for the {{get_job_order.output.job_title}} role.\n\nPlanned Start Date: {{get_placement.output.start_date}}\n\nWe're completing onboarding steps (background check, documentation) and will confirm the start date as we get closer. Please let us know if there are any site-specific onboarding requirements.\n\nThank you for your partnership."
    }
  }
]
```

## Results

| Metric | Before | After |
|--------|--------|-------|
| Delayed start dates | 25% of placements | < 5% |
| Avg. onboarding completion time | 12 days | 7 days |
| Lost revenue from delayed starts | $2,000/placement | Near zero |

-> [Set up this workflow on Lotics](https://lotics.ai)

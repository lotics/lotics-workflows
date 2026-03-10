# New Case Intake

## The problem

A 15-attorney law firm receives 30-40 potential case inquiries per month through their website contact form. The intake coordinator manually reviews each submission, checks for conflicts of interest against existing clients, assigns it to a practice group, and sends a follow-up email. During busy periods, leads sit unresponded for 2-3 days. Conflict checks are done from memory or a manual spreadsheet search, creating malpractice risk. The firm estimates they lose 15-20% of viable cases to slow response times.

## Data model

| Table | Key fields | Purpose |
|-------|-----------|---------|
| Case Inquiries | contact_name, contact_email, phone, matter_type, description, status, assigned_attorney, conflict_check_result | Incoming potential client inquiries |
| Cases | case_number, client_name, matter_type, opposing_party, status, lead_attorney, opened_date | Active and closed case records |
| Attorneys | attorney_name, practice_areas, email, case_load, status | Firm attorneys and their specializations |

## Example prompts

- "When someone submits a case inquiry form, run a conflict check against existing cases, assign it to an available attorney in the right practice area, and send the potential client a confirmation email."
- "Automate new matter intake: check for conflicts, route to the right attorney based on practice area and workload, and follow up with the prospect within minutes."

## Workflow

**Trigger:** `form_submitted` — potential client submits inquiry through the website intake form.

```json
[
  {
    "id": "create_inquiry",
    "type": "tool_call",
    "description": "Create a case inquiry record from the form submission",
    "tool_name": "create_record",
    "input": {
      "table_name": "Case Inquiries",
      "data": {
        "contact_name": "{{trigger.form_submitted.data.contact_name}}",
        "contact_email": "{{trigger.form_submitted.data.contact_email}}",
        "phone": "{{trigger.form_submitted.data.phone}}",
        "matter_type": "{{trigger.form_submitted.data.matter_type}}",
        "description": "{{trigger.form_submitted.data.description}}",
        "status": "Conflict Check"
      }
    }
  },
  {
    "id": "conflict_check",
    "type": "tool_call",
    "description": "Search existing cases for conflicts — matching client names or opposing parties",
    "tool_name": "query_records",
    "input": {
      "table_name": "Cases",
      "filters": {
        "or": [
          { "client_name__contains": "{{trigger.form_submitted.data.contact_name}}" },
          { "opposing_party__contains": "{{trigger.form_submitted.data.contact_name}}" }
        ]
      }
    }
  },
  {
    "id": "evaluate_conflict",
    "type": "if",
    "description": "Check if any potential conflicts were found",
    "condition": "{{conflict_check.output.total > 0}}",
    "then": [
      {
        "id": "flag_conflict",
        "type": "tool_call",
        "description": "Mark the inquiry as having a potential conflict for manual review",
        "tool_name": "update_record",
        "input": {
          "table_name": "Case Inquiries",
          "record_id": "{{create_inquiry.output.record_id}}",
          "data": {
            "status": "Conflict Flagged",
            "conflict_check_result": "Potential conflict found with {{conflict_check.output.total}} existing case(s). Manual review required."
          }
        }
      },
      {
        "id": "notify_managing_partner",
        "type": "tool_call",
        "description": "Alert the managing partner about the conflict for review",
        "tool_name": "send_notification",
        "input": {
          "member_id": "{{managing_partner_id}}",
          "title": "Conflict flagged: {{trigger.form_submitted.data.contact_name}}",
          "message": "A new case inquiry from {{trigger.form_submitted.data.contact_name}} ({{trigger.form_submitted.data.matter_type}}) has a potential conflict with {{conflict_check.output.total}} existing case(s). Please review before proceeding."
        }
      }
    ],
    "else": [
      {
        "id": "mark_clear",
        "type": "tool_call",
        "description": "Mark conflict check as clear",
        "tool_name": "update_record",
        "input": {
          "table_name": "Case Inquiries",
          "record_id": "{{create_inquiry.output.record_id}}",
          "data": {
            "conflict_check_result": "No conflicts found"
          }
        }
      },
      {
        "id": "find_attorney",
        "type": "tool_call",
        "description": "Find an available attorney in the matching practice area with the lowest case load",
        "tool_name": "query_records",
        "input": {
          "table_name": "Attorneys",
          "filters": {
            "practice_areas__contains": "{{trigger.form_submitted.data.matter_type}}",
            "status": "Active"
          },
          "sort": {
            "field": "case_load",
            "direction": "asc"
          },
          "limit": 1
        }
      },
      {
        "id": "assign_attorney",
        "type": "tool_call",
        "description": "Assign the inquiry to the selected attorney",
        "tool_name": "update_record",
        "input": {
          "table_name": "Case Inquiries",
          "record_id": "{{create_inquiry.output.record_id}}",
          "data": {
            "assigned_attorney": "{{find_attorney.output.records[0].id}}",
            "status": "Assigned"
          }
        }
      },
      {
        "id": "notify_attorney",
        "type": "tool_call",
        "description": "Notify the assigned attorney about the new inquiry",
        "tool_name": "send_notification",
        "input": {
          "member_id": "{{find_attorney.output.records[0].id}}",
          "title": "New {{trigger.form_submitted.data.matter_type}} inquiry assigned to you",
          "message": "{{trigger.form_submitted.data.contact_name}} has submitted a {{trigger.form_submitted.data.matter_type}} inquiry. Conflict check is clear. Please review and schedule an initial consultation."
        }
      },
      {
        "id": "send_confirmation",
        "type": "tool_call",
        "description": "Send the potential client a confirmation email with next steps",
        "tool_name": "gmail_send_email",
        "input": {
          "to": "{{trigger.form_submitted.data.contact_email}}",
          "subject": "Your inquiry has been received — {{trigger.form_submitted.data.matter_type}}",
          "body": "Dear {{trigger.form_submitted.data.contact_name}},\n\nThank you for reaching out to our firm regarding your {{trigger.form_submitted.data.matter_type}} matter.\n\nYour inquiry has been assigned to {{find_attorney.output.records[0].attorney_name}}, who will contact you within one business day to schedule an initial consultation.\n\nIf you have any urgent questions, please call our office directly.\n\nBest regards,\nIntake Team"
        }
      }
    ]
  }
]
```

## Results

| Metric | Before | After |
|--------|--------|-------|
| Avg. response time to inquiries | 2-3 days | < 15 minutes |
| Conflict check accuracy | Manual/incomplete | 100% automated |
| Viable cases lost to slow response | 15-20% | < 3% |

-> [Set up this workflow on Lotics](https://lotics.ai)

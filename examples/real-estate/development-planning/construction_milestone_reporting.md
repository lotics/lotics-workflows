# Construction Milestone Reporting

## The problem

A development company builds 3-5 projects concurrently, each with 20-40 construction milestones (foundation pour, framing complete, rough-in inspections, drywall, finishes). When a superintendent marks a milestone complete in the field, the project manager, investor relations team, and lender all need different reports. Superintendents update a shared system, but generating the investor PDF, updating the lender draw schedule, and emailing stakeholders takes the back office 3-4 hours per milestone across all projects -- roughly 20 hours per week during peak construction.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Projects | project_id, name, address, total_budget, lender_contact_email, investor_group_email | Development project master record |
| Milestones | milestone_id, project_id, name, phase, planned_date, actual_date, status, completion_pct, superintendent_notes | Construction milestone tracker |
| Draw Requests | draw_id, project_id, milestone_id, amount, status, submitted_at | Lender draw schedule tied to milestones |

## Example prompts

- "When a superintendent updates a milestone status to Complete, generate a progress report PDF, email it to the investor group, create a draw request record for the lender, and email the lender with the draw details."
- "Automate milestone completion: when a milestone is marked done, pull the project info, generate the investor report, file the draw request, and notify everyone in one go."

## Workflow

**Trigger:** A milestone record's status field is updated to "Complete".

```json
[
  {
    "id": "get_project",
    "type": "tool_call",
    "description": "Fetch the parent project record for contact and budget details",
    "tool_name": "get_record",
    "input": {
      "table_name": "Projects",
      "record_id": "{{trigger.record_updated.next_data.project_id}}"
    }
  },
  {
    "id": "get_all_milestones",
    "type": "tool_call",
    "description": "Query all milestones for this project to calculate overall progress",
    "tool_name": "query_records",
    "input": {
      "table_name": "Milestones",
      "filters": {
        "project_id": "{{trigger.record_updated.next_data.project_id}}"
      }
    }
  },
  {
    "id": "generate_progress_report",
    "type": "tool_call",
    "description": "Generate a PDF progress report for investors",
    "tool_name": "generate_pdf_from_html_template",
    "input": {
      "template_name": "Construction Progress Report",
      "data": {
        "project_name": "{{get_project.output.record.name}}",
        "project_address": "{{get_project.output.record.address}}",
        "milestone_completed": "{{trigger.record_updated.next_data.name}}",
        "completion_date": "{{trigger.record_updated.next_data.actual_date}}",
        "superintendent_notes": "{{trigger.record_updated.next_data.superintendent_notes}}",
        "overall_completion_pct": "{{trigger.record_updated.next_data.completion_pct}}",
        "total_milestones": "{{get_all_milestones.output.records.length}}",
        "report_date": "{{NOW()}}"
      }
    }
  },
  {
    "id": "attach_report",
    "type": "tool_call",
    "description": "Attach the progress report PDF to the milestone record",
    "tool_name": "add_files_to_record",
    "input": {
      "table_name": "Milestones",
      "record_id": "{{trigger.record_updated.record_id}}",
      "files": ["{{generate_progress_report.output.file_url}}"]
    }
  },
  {
    "id": "email_investors",
    "type": "tool_call",
    "description": "Email the progress report to the investor group",
    "tool_name": "gmail_send_email",
    "input": {
      "to": "{{get_project.output.record.investor_group_email}}",
      "subject": "{{get_project.output.record.name}} - Milestone Complete: {{trigger.record_updated.next_data.name}}",
      "body": "Please find attached the latest construction progress report for {{get_project.output.record.name}}.\n\nMilestone completed: {{trigger.record_updated.next_data.name}}\nCompletion date: {{trigger.record_updated.next_data.actual_date}}\nOverall project progress: {{trigger.record_updated.next_data.completion_pct}}%\n\nSuperintendent notes: {{trigger.record_updated.next_data.superintendent_notes}}",
      "attachments": ["{{generate_progress_report.output.file_url}}"]
    }
  },
  {
    "id": "create_draw_request",
    "type": "tool_call",
    "description": "Create a lender draw request linked to this milestone",
    "tool_name": "create_record",
    "input": {
      "table_name": "Draw Requests",
      "data": {
        "project_id": "{{trigger.record_updated.next_data.project_id}}",
        "milestone_id": "{{trigger.record_updated.record_id}}",
        "amount": "{{trigger.record_updated.next_data.draw_amount}}",
        "status": "Submitted",
        "submitted_at": "{{NOW()}}"
      }
    }
  },
  {
    "id": "email_lender",
    "type": "tool_call",
    "description": "Email the lender with the draw request details",
    "tool_name": "outlook_send_email",
    "input": {
      "to": "{{get_project.output.record.lender_contact_email}}",
      "subject": "Draw Request - {{get_project.output.record.name}} - {{trigger.record_updated.next_data.name}}",
      "body": "This is to notify you that the following construction milestone has been completed and a draw request has been submitted.\n\nProject: {{get_project.output.record.name}}\nMilestone: {{trigger.record_updated.next_data.name}}\nDraw Amount: ${{trigger.record_updated.next_data.draw_amount}}\nCompletion Date: {{trigger.record_updated.next_data.actual_date}}\n\nPlease process the draw at your earliest convenience. The inspection report is available upon request."
    }
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| Back office hours per milestone | 3-4 hours | Under 5 minutes |
| Time from milestone completion to investor notification | 1-2 business days | Immediate |
| Draw request submission delay | 3-5 business days | Same day |
| Weekly admin hours during peak construction | 20 hours | 2 hours (review only) |

-> [Set up this workflow on Lotics](https://lotics.ai)

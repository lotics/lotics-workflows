# Milestone Progress Report

## The problem

Construction project managers juggle 15-30 active milestones across multiple projects. When a milestone is marked complete, the PM must manually update the project schedule, notify stakeholders, recalculate budget burn, and file a progress report. This takes 45 minutes per milestone and delays are common -- 60% of stakeholder updates go out more than 48 hours late.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Projects | project_name, owner, total_budget, start_date, target_completion | Master project record |
| Milestones | milestone_name, project (link), status, due_date, actual_completion_date, budget_allocated, budget_spent | Individual deliverables within a project |
| Stakeholders | name, email, role, project (link) | People who receive progress updates |

## Example prompts

- "When a milestone is marked complete, calculate the project's overall progress percentage, generate a PDF progress report, and email it to all stakeholders on that project."
- "Every time a milestone status changes to Complete, update the parent project's budget spent, then send a summary email to stakeholders with the milestone name, completion date, and remaining budget."

## Workflow

**Trigger:** When a milestone record is updated and status changes to "Complete"

```json
[
  {
    "id": "check_status_change",
    "type": "if",
    "description": "Only proceed if status just changed to Complete",
    "condition": "{{trigger.record_updated.next_data.status === 'Complete' && trigger.record_updated.prev_data.status !== 'Complete'}}",
    "then": [
      {
        "id": "get_project",
        "type": "tool_call",
        "description": "Fetch the parent project record",
        "tool_name": "get_record",
        "input": {
          "table_name": "Projects",
          "record_id": "{{trigger.record_updated.next_data.project}}"
        }
      },
      {
        "id": "get_all_milestones",
        "type": "tool_call",
        "description": "Query all milestones for this project to calculate progress",
        "tool_name": "query_records",
        "input": {
          "table_name": "Milestones",
          "filters": {
            "project": "{{trigger.record_updated.next_data.project}}"
          }
        }
      },
      {
        "id": "update_project_budget",
        "type": "tool_call",
        "description": "Update the project with recalculated budget spent and progress percentage",
        "tool_name": "update_record",
        "input": {
          "table_name": "Projects",
          "record_id": "{{trigger.record_updated.next_data.project}}",
          "data": {
            "budget_spent": "{{get_all_milestones.output.records.reduce((sum, m) => sum + (m.budget_spent || 0), 0)}}",
            "progress_percent": "{{Math.round((get_all_milestones.output.records.filter(m => m.status === 'Complete').length / get_all_milestones.output.records.length) * 100)}}"
          }
        }
      },
      {
        "id": "generate_report",
        "type": "tool_call",
        "description": "Generate a PDF progress report from the HTML template",
        "tool_name": "generate_pdf_from_html_template",
        "input": {
          "template_name": "milestone_progress_report",
          "data": {
            "project_name": "{{get_project.output.record.project_name}}",
            "milestone_name": "{{trigger.record_updated.next_data.milestone_name}}",
            "completion_date": "{{trigger.record_updated.next_data.actual_completion_date}}",
            "total_budget": "{{get_project.output.record.total_budget}}",
            "budget_spent": "{{get_all_milestones.output.records.reduce((sum, m) => sum + (m.budget_spent || 0), 0)}}",
            "progress_percent": "{{Math.round((get_all_milestones.output.records.filter(m => m.status === 'Complete').length / get_all_milestones.output.records.length) * 100)}}"
          }
        }
      },
      {
        "id": "get_stakeholders",
        "type": "tool_call",
        "description": "Query all stakeholders for this project",
        "tool_name": "query_records",
        "input": {
          "table_name": "Stakeholders",
          "filters": {
            "project": "{{trigger.record_updated.next_data.project}}"
          }
        }
      },
      {
        "id": "notify_stakeholders",
        "type": "foreach",
        "description": "Email each stakeholder the progress report",
        "items": "{{get_stakeholders.output.records}}",
        "steps": [
          {
            "id": "send_email",
            "type": "tool_call",
            "description": "Send progress report email to stakeholder",
            "tool_name": "gmail_send_email",
            "input": {
              "to": "{{notify_stakeholders.item.email}}",
              "subject": "Milestone Complete: {{trigger.record_updated.next_data.milestone_name}} — {{get_project.output.record.project_name}}",
              "body": "Hi {{notify_stakeholders.item.name}},\n\nThe milestone \"{{trigger.record_updated.next_data.milestone_name}}\" on {{get_project.output.record.project_name}} was completed on {{trigger.record_updated.next_data.actual_completion_date}}.\n\nProject progress: {{Math.round((get_all_milestones.output.records.filter(m => m.status === 'Complete').length / get_all_milestones.output.records.length) * 100)}}%\nBudget remaining: ${{get_project.output.record.total_budget - get_all_milestones.output.records.reduce((sum, m) => sum + (m.budget_spent || 0), 0)}}\n\nSee the attached PDF for the full progress report.",
              "attachments": ["{{generate_report.output.file_url}}"]
            }
          }
        ]
      }
    ],
    "else": []
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| Time per milestone update | 45 minutes | 0 minutes (automated) |
| Stakeholder notification delay | 48+ hours | Under 5 minutes |
| Budget accuracy | Updated weekly | Real-time on each milestone |

-> [Set up this workflow on Lotics](https://lotics.ai)

# Weekly Project Status Report

## The problem

Delivery managers at consulting firms spend 2-3 hours every Monday morning pulling together status updates from 20+ active projects. They check each project's milestones, count overdue tasks, and manually compose a summary email to leadership. The reports are inconsistent, sometimes late, and always tedious. Leadership wants a single snapshot — not a scavenger hunt across spreadsheets.

## Data model

| Table | Key fields | Purpose |
|-------|-----------|---------|
| Projects | project_name, client, delivery_lead, status, health_rating, budget_remaining | Active client engagements |
| Tasks | task_name, project (link), assignee, status, due_date, priority | Individual work items within projects |
| Status Reports | report_date, summary, project_count, overdue_count, generated_by | Weekly snapshots sent to leadership |

## Example prompts

- "Every Monday at 8am, pull all active projects, count overdue tasks for each, and email a summary report to the leadership team."
- "Set up a weekly status digest that aggregates project health across the portfolio and sends it to the partners."

## Workflow

**Trigger:** `recurring_schedule` — every Monday at 08:00.

```json
[
  {
    "id": "get_active_projects",
    "type": "tool_call",
    "description": "Query all projects with status Active",
    "tool_name": "query_records",
    "input": {
      "table_name": "Projects",
      "filters": {
        "status": "Active"
      }
    }
  },
  {
    "id": "get_overdue_tasks",
    "type": "tool_call",
    "description": "Query all tasks that are overdue and not completed",
    "tool_name": "query_records",
    "input": {
      "table_name": "Tasks",
      "filters": {
        "status__not": "Completed",
        "due_date__lt": "{{today()}}"
      }
    }
  },
  {
    "id": "generate_summary",
    "type": "tool_call",
    "description": "Use LLM to compose a structured status report from the raw data",
    "tool_name": "llm_generate_text",
    "input": {
      "prompt": "Generate a concise weekly project status report for leadership. Format it as an HTML email with sections for Portfolio Overview, At-Risk Projects, and Overdue Tasks.\n\nActive Projects ({{get_active_projects.output.total}} total):\n{{JSON.stringify(get_active_projects.output.records)}}\n\nOverdue Tasks ({{get_overdue_tasks.output.total}} total):\n{{JSON.stringify(get_overdue_tasks.output.records)}}\n\nToday's date: {{today()}}. Keep the tone professional and highlight anything that needs immediate attention."
    }
  },
  {
    "id": "save_report",
    "type": "tool_call",
    "description": "Save the generated report as a record for audit trail",
    "tool_name": "create_record",
    "input": {
      "table_name": "Status Reports",
      "data": {
        "report_date": "{{today()}}",
        "summary": "{{generate_summary.output.text}}",
        "project_count": "{{get_active_projects.output.total}}",
        "overdue_count": "{{get_overdue_tasks.output.total}}",
        "generated_by": "Workflow"
      }
    }
  },
  {
    "id": "email_leadership",
    "type": "tool_call",
    "description": "Send the formatted report to the leadership distribution list",
    "tool_name": "gmail_send_email",
    "input": {
      "to": "leadership@company.com",
      "subject": "Weekly Project Status — {{today()}} | {{get_active_projects.output.total}} Active, {{get_overdue_tasks.output.total}} Overdue",
      "body": "{{generate_summary.output.text}}"
    }
  }
]
```

## Results

| Metric | Before | After |
|--------|--------|-------|
| Time to compile weekly report | 2-3 hours | 0 (automated) |
| Report delivery consistency | ~70% on time | 100% on time |
| Data staleness in reports | Up to 3 days old | Real-time at generation |

-> [Set up this workflow on Lotics](https://lotics.ai)

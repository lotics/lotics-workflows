# Post-Production Pipeline & Delivery Deadline Alert

## The problem

After a shoot, photos enter a post-production pipeline: culling, editing, retouching, and final export. A studio processing 10 projects simultaneously loses track of where each project sits. Editors work from a shared folder with no status tracking. The studio owner discovers a project is overdue only when the client emails asking where their photos are. Average delivery is 3 weeks, but clients were promised 2 weeks. Late deliveries damage reputation and generate negative reviews.

## Data model

| Table | Key fields | Purpose |
|-------|-----------|---------|
| Projects | client_name, client_email, shoot_type, shoot_date, status, delivery_deadline, photographer_id, editor_id | Project lifecycle tracking |
| Post Production Tasks | project_id, task_type, assigned_to, status, due_date, completed_at | Individual editing tasks |

## Example prompts

- "Every morning, check for post-production tasks that are due today or overdue. Notify the assigned editor and flag the project if the delivery deadline is within 2 days."
- "Alert editors about upcoming and overdue tasks daily, and escalate to me if any project's final delivery deadline is at risk."

## Workflow

**Trigger:** Recurring schedule, daily at 09:00

```json
[
  {
    "id": "find_urgent_tasks",
    "type": "tool_call",
    "description": "Query post-production tasks due today or overdue",
    "tool_name": "query_records",
    "input": {
      "table_name": "Post Production Tasks",
      "filters": {
        "due_date_on_or_before": "{{trigger.recurring_schedule.scheduled_at_date}}",
        "status_not": "completed"
      }
    }
  },
  {
    "id": "process_tasks",
    "type": "foreach",
    "description": "Notify each editor about their urgent tasks",
    "items": "{{find_urgent_tasks.output.records}}",
    "steps": [
      {
        "id": "get_project",
        "type": "tool_call",
        "description": "Fetch the parent project for deadline context",
        "tool_name": "get_record",
        "input": {
          "table_name": "Projects",
          "record_id": "{{item.project_id}}"
        }
      },
      {
        "id": "notify_editor",
        "type": "tool_call",
        "description": "Send task reminder to the assigned editor",
        "tool_name": "send_notification",
        "input": {
          "message": "Urgent: {{item.task_type}} for project {{get_project.output.client_name}} ({{get_project.output.shoot_type}}) is due {{item.due_date}}. Client delivery deadline: {{get_project.output.delivery_deadline}}. Please update task status when complete.",
          "member_id": "{{item.assigned_to}}"
        }
      },
      {
        "id": "add_task_comment",
        "type": "tool_call",
        "description": "Log the reminder on the task record",
        "tool_name": "create_record_comments",
        "input": {
          "table_name": "Post Production Tasks",
          "record_id": "{{item.id}}",
          "comment": "Automated reminder sent. Task due: {{item.due_date}}, Project delivery deadline: {{get_project.output.delivery_deadline}}"
        }
      }
    ]
  }
]
```

## Results

| Metric | Before | After |
|--------|--------|-------|
| Projects delivered late | 40% | 10% |
| Overdue tasks discovered reactively | Most of them | 0 (caught proactively) |
| Average delivery time | 3 weeks | Under 2 weeks |

-> [Set up this workflow on Lotics](https://lotics.ai)

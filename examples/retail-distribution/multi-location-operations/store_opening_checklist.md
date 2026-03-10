# New Store Opening Checklist

## The problem

Opening a new branch takes 8-12 weeks and involves 40+ tasks across facilities, IT, HR, merchandising, and compliance. The process lives in a shared document that the project lead copies and edits for each new location. Tasks get missed — the last three openings had at least one compliance item (fire inspection, business license, POS certification) completed after the store was already open, exposing the company to fines. Handoffs between departments are verbal, so delays compound silently.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Store Openings | `project_id`, `branch_name`, `address`, `target_open_date`, `status`, `project_lead` | Master record for each new store project |
| Opening Tasks | `task_id`, `project_id`, `task_name`, `department`, `status`, `due_date`, `assigned_to`, `depends_on`, `completed_at` | Individual checklist items with dependencies |
| Task Templates | `template_id`, `task_name`, `department`, `offset_days`, `depends_on_template` | Standard tasks and their relative timing |
| Branches | `branch_id`, `branch_name`, `region`, `manager_name`, `manager_email`, `status` | Store directory |

## Example prompts

- "When a new store opening record is created, generate the full checklist of 40+ tasks from our template, set due dates based on the target open date, and assign each task to the right department."
- "Automatically spin up the entire store opening project plan whenever we add a new location — create all tasks with deadlines, dependencies, and department assignments."

## Workflow

**Trigger:** `record_submit` on the Store Openings table — fires when a new store opening project is submitted.

```json
[
  {
    "id": "get_templates",
    "type": "tool_call",
    "description": "Fetch all standard task templates for store openings",
    "tool_name": "query_records",
    "input": {
      "table": "Task Templates",
      "sort": { "field": "offset_days", "direction": "asc" }
    }
  },
  {
    "id": "create_tasks",
    "type": "foreach",
    "description": "Create an opening task for each template item, calculating due dates from the target open date",
    "items": "{{ get_templates.output.records }}",
    "steps": [
      {
        "id": "create_task",
        "type": "tool_call",
        "description": "Create the task with a due date offset from the target opening date",
        "tool_name": "create_record",
        "input": {
          "table": "Opening Tasks",
          "data": {
            "project_id": "{{ trigger.record_submit.data.project_id }}",
            "task_name": "{{ item.task_name }}",
            "department": "{{ item.department }}",
            "status": "Not Started",
            "due_date": "{{ new Date(new Date(trigger.record_submit.data.target_open_date).getTime() - item.offset_days * 24 * 60 * 60 * 1000).toISOString().split('T')[0] }}",
            "depends_on": "{{ item.depends_on_template }}"
          }
        }
      }
    ]
  },
  {
    "id": "get_departments",
    "type": "tool_call",
    "description": "Query team members to identify department leads for notification",
    "tool_name": "query_members",
    "input": {}
  },
  {
    "id": "notify_lead",
    "type": "tool_call",
    "description": "Notify the project lead that the checklist has been generated",
    "tool_name": "send_notification",
    "input": {
      "message": "Store opening project for {{ trigger.record_submit.data.branch_name }} has been initialized with {{ get_templates.output.records.length }} tasks. Target open date: {{ trigger.record_submit.data.target_open_date }}. All department leads have been assigned. Review the task list and adjust dates as needed."
    }
  },
  {
    "id": "create_branch_record",
    "type": "tool_call",
    "description": "Create a placeholder branch record in the store directory",
    "tool_name": "create_record",
    "input": {
      "table": "Branches",
      "data": {
        "branch_name": "{{ trigger.record_submit.data.branch_name }}",
        "region": "{{ trigger.record_submit.data.region }}",
        "status": "Pre-Opening"
      }
    }
  }
]
```

-> [Set up this workflow on Lotics](https://lotics.ai)

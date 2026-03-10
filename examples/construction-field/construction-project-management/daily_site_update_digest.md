# Daily Site Update Digest

## The problem

Site superintendents submit daily logs from the field -- weather conditions, crew headcount, equipment on site, safety incidents, and work completed. Project managers overseeing 4-8 sites must manually review each log, flag issues, and compile a summary for the weekly owner meeting. With 20-40 daily logs per week, PMs spend 5+ hours just reading and synthesizing updates. Critical safety incidents get buried in routine reports.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Projects | project_name, owner, status, site_address | Master project record |
| Daily Logs | project (link), log_date, weather, crew_count, equipment_on_site, work_completed, safety_incidents, submitted_by | Field-submitted daily updates |
| Project Managers | name, email, projects (link) | PM assignments |

## Example prompts

- "Every day at 6 PM, pull all daily logs submitted today, use AI to summarize them into a digest grouped by project, flag any safety incidents, and email the digest to each project manager."
- "At the end of each workday, compile today's site logs into a single summary email per PM. Highlight any projects with zero logs submitted and any safety incidents reported."

## Workflow

**Trigger:** Recurring schedule, daily at 6:00 PM

```json
[
  {
    "id": "get_todays_logs",
    "type": "tool_call",
    "description": "Query all daily logs submitted today",
    "tool_name": "query_records",
    "input": {
      "table_name": "Daily Logs",
      "filters": {
        "log_date": "{{new Date().toISOString().split('T')[0]}}"
      }
    }
  },
  {
    "id": "get_active_projects",
    "type": "tool_call",
    "description": "Get all active projects to check for missing logs",
    "tool_name": "query_records",
    "input": {
      "table_name": "Projects",
      "filters": {
        "status": "Active"
      }
    }
  },
  {
    "id": "generate_digest",
    "type": "tool_call",
    "description": "Use LLM to synthesize daily logs into a structured digest",
    "tool_name": "llm_generate_text",
    "input": {
      "prompt": "You are a construction project management assistant. Summarize the following daily site logs into a digest grouped by project. For each project, include: crew count, weather, key work completed, and equipment on site. Create a separate SAFETY ALERTS section at the top listing any safety incidents with the project name and details. Also list any active projects that have NO log submitted today as 'Missing Reports'.\n\nDaily logs:\n{{JSON.stringify(get_todays_logs.output.records)}}\n\nActive projects:\n{{JSON.stringify(get_active_projects.output.records.map(p => p.project_name))}}"
    }
  },
  {
    "id": "get_project_managers",
    "type": "tool_call",
    "description": "Query all project managers to send the digest",
    "tool_name": "query_records",
    "input": {
      "table_name": "Project Managers"
    }
  },
  {
    "id": "send_digests",
    "type": "foreach",
    "description": "Email the digest to each project manager",
    "items": "{{get_project_managers.output.records}}",
    "steps": [
      {
        "id": "email_pm",
        "type": "tool_call",
        "description": "Send daily digest email",
        "tool_name": "gmail_send_email",
        "input": {
          "to": "{{send_digests.item.email}}",
          "subject": "Daily Site Digest — {{new Date().toISOString().split('T')[0]}} ({{get_todays_logs.output.records.length}} logs, {{get_todays_logs.output.records.filter(l => l.safety_incidents).length}} safety alerts)",
          "body": "{{generate_digest.output.text}}"
        }
      }
    ]
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| PM time reviewing daily logs | 5+ hours/week | 15 minutes/week (review digest) |
| Safety incident response time | Next morning at earliest | Same evening |
| Missing log detection | End of week | Same day |

-> [Set up this workflow on Lotics](https://lotics.ai)

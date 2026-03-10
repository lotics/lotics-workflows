# Permit Status Tracking and Escalation

## The problem

A real estate development firm manages 8-12 active projects, each requiring 5-15 municipal permits (grading, building, mechanical, electrical, plumbing, fire, environmental). Permit review timelines range from 2 weeks to 6 months. Project managers check the county portal manually, often discovering expired or rejected permits days late. A single missed permit rejection can delay a project phase by 4-6 weeks, costing $50,000-200,000 in idle contractor fees and carrying costs.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Projects | project_id, name, address, project_manager_id, phase, target_completion | Development project registry |
| Permits | permit_id, project_id, permit_type, jurisdiction, application_date, status, expiry_date, portal_url, last_checked | Individual permit applications and their current status |
| Status Changes | change_id, permit_id, previous_status, new_status, changed_at, notes | Audit log of all permit status transitions |

## Example prompts

- "Every weekday morning, scrape the permit portal URL for each pending permit, compare the status to what we have on file, and if anything changed, log it and notify the project manager immediately."
- "Set up a daily permit status checker that visits each permit's portal page, detects status changes, updates our records, and escalates rejections to the project manager."

## Workflow

**Trigger:** Recurring schedule, weekdays at 7:00 AM.

```json
[
  {
    "id": "get_pending_permits",
    "type": "tool_call",
    "description": "Query all permits with a status of Submitted or Under Review",
    "tool_name": "query_records",
    "input": {
      "table_name": "Permits",
      "filters": {
        "status": ["Submitted", "Under Review"]
      }
    }
  },
  {
    "id": "check_each_permit",
    "type": "foreach",
    "description": "Visit each permit's portal URL and check for status changes",
    "items": "{{get_pending_permits.output.records}}",
    "steps": [
      {
        "id": "scrape_portal",
        "type": "tool_call",
        "description": "Scrape the permit portal page for current status",
        "tool_name": "web_scrape_url",
        "input": {
          "url": "{{item.portal_url}}"
        }
      },
      {
        "id": "extract_status",
        "type": "tool_call",
        "description": "Use AI to extract the current permit status from the scraped page",
        "tool_name": "llm_generate_text",
        "input": {
          "prompt": "Extract the permit status from this municipal portal page content. Return a JSON object with one field: \"status\" (one of: Submitted, Under Review, Approved, Rejected, Corrections Required, Expired). Page content: {{scrape_portal.output.content}}"
        }
      },
      {
        "id": "check_changed",
        "type": "if",
        "description": "Only proceed if the status has changed from what we have on file",
        "condition": "{{extract_status.output.status !== item.status}}",
        "then": [
          {
            "id": "log_change",
            "type": "tool_call",
            "description": "Create a status change audit record",
            "tool_name": "create_record",
            "input": {
              "table_name": "Status Changes",
              "data": {
                "permit_id": "{{item.permit_id}}",
                "previous_status": "{{item.status}}",
                "new_status": "{{extract_status.output.status}}",
                "changed_at": "{{NOW()}}",
                "notes": "Detected via automated portal check"
              }
            }
          },
          {
            "id": "update_permit",
            "type": "tool_call",
            "description": "Update the permit record with the new status",
            "tool_name": "update_record",
            "input": {
              "table_name": "Permits",
              "record_id": "{{item.permit_id}}",
              "data": {
                "status": "{{extract_status.output.status}}",
                "last_checked": "{{NOW()}}"
              }
            }
          },
          {
            "id": "check_escalation",
            "type": "if",
            "description": "Escalate immediately if the permit was rejected or requires corrections",
            "condition": "{{extract_status.output.status === 'Rejected' || extract_status.output.status === 'Corrections Required'}}",
            "then": [
              {
                "id": "get_project",
                "type": "tool_call",
                "description": "Fetch the parent project to find the project manager",
                "tool_name": "get_record",
                "input": {
                  "table_name": "Projects",
                  "record_id": "{{item.project_id}}"
                }
              },
              {
                "id": "escalate_notification",
                "type": "tool_call",
                "description": "Send urgent notification to the project manager",
                "tool_name": "send_notification",
                "input": {
                  "recipient_id": "{{get_project.output.record.project_manager_id}}",
                  "message": "URGENT: Permit {{item.permit_type}} for project {{get_project.output.record.name}} has been {{extract_status.output.status}}. Review and respond to the jurisdiction immediately to avoid project delays."
                }
              }
            ],
            "else": []
          }
        ],
        "else": [
          {
            "id": "mark_checked",
            "type": "tool_call",
            "description": "Update last_checked timestamp even if status unchanged",
            "tool_name": "update_record",
            "input": {
              "table_name": "Permits",
              "record_id": "{{item.permit_id}}",
              "data": {
                "last_checked": "{{NOW()}}"
              }
            }
          }
        ]
      }
    ]
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| Time to detect permit status change | 1-5 business days | Same day |
| Permit rejections discovered late | 30% | 0% |
| Project delays from missed permits | 2-3 per year | 0 |
| Staff hours on manual portal checks | 10 hours/week | 0 |

-> [Set up this workflow on Lotics](https://lotics.ai)

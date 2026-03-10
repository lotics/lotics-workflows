# Change Order Review

## The problem

Public works projects routinely encounter unforeseen conditions — soil contamination, utility conflicts, design errors — that require formal change orders. A mid-sized city processes 150+ change orders per year across its capital program. Each change order must be reviewed by the project engineer, checked against remaining contingency budget, and routed to the appropriate authority based on dollar threshold (project manager under $25K, public works director $25K-$100K, city council over $100K). Today this is a paper-and-email process that averages 14 days per change order, and contractors frequently start work before approval is granted, creating payment disputes.

## Data model

| Table | Key fields | Purpose |
|-------|-----------|---------|
| Change Orders | project_id, contractor_id, description, justification, amount, category, status, submitted_by, approver, approval_level, submitted_date, approved_date | Track each change order from submission to approval |
| Projects | project_name, original_budget, contingency_budget, contingency_remaining, project_manager, project_engineer | Project budget and personnel |
| Contractors | company_name, contact_email | Contractor contact information |

## Example prompts

- "When a contractor submits a change order, check the project contingency budget, determine the right approval authority based on the dollar amount, route it for approval, and notify the contractor that review is underway."
- "Automate change order intake: validate the amount against remaining contingency, assign the right approver by threshold, and create a comment trail on the project record."

## Workflow

**Trigger:** `button_pressed` — the project engineer clicks "Submit for Review" on a change order record.

```json
[
  {
    "id": "get_change_order",
    "type": "tool_call",
    "description": "Read the full change order record",
    "tool_name": "get_record",
    "input": {
      "table_id": "change_orders",
      "record_id": "{{trigger.button_pressed.record_id}}"
    }
  },
  {
    "id": "get_project",
    "type": "tool_call",
    "description": "Look up the project to check contingency budget",
    "tool_name": "get_record",
    "input": {
      "table_id": "projects",
      "record_id": "{{get_change_order.output.project_id}}"
    }
  },
  {
    "id": "check_contingency",
    "type": "if",
    "description": "Verify the change order amount does not exceed remaining contingency",
    "condition": "{{get_change_order.output.amount > get_project.output.contingency_remaining}}",
    "then": [
      {
        "id": "flag_over_budget",
        "type": "tool_call",
        "description": "Mark the change order as over-contingency and notify the project manager",
        "tool_name": "update_record",
        "input": {
          "table_id": "change_orders",
          "record_id": "{{trigger.button_pressed.record_id}}",
          "data": {
            "status": "Over Contingency - Requires Budget Amendment"
          }
        }
      },
      {
        "id": "comment_over_budget",
        "type": "tool_call",
        "description": "Add a comment explaining the budget shortfall",
        "tool_name": "create_record_comments",
        "input": {
          "table_id": "change_orders",
          "record_id": "{{trigger.button_pressed.record_id}}",
          "comment": "Change order amount (${{get_change_order.output.amount}}) exceeds remaining contingency (${{get_project.output.contingency_remaining}}). A budget amendment is required before this can be approved."
        }
      },
      {
        "id": "notify_pm_over_budget",
        "type": "tool_call",
        "description": "Alert the project manager about the budget shortfall",
        "tool_name": "send_notification",
        "input": {
          "user_email": "{{get_project.output.project_manager}}",
          "title": "Change order exceeds contingency: {{get_project.output.project_name}}",
          "message": "Change order for ${{get_change_order.output.amount}} exceeds remaining contingency of ${{get_project.output.contingency_remaining}}. A budget amendment must be initiated before approval can proceed."
        }
      }
    ],
    "else": [
      {
        "id": "determine_authority",
        "type": "switch",
        "description": "Route to the correct approval authority based on dollar threshold",
        "expression": "{{get_change_order.output.amount <= 25000 ? 'pm' : get_change_order.output.amount <= 100000 ? 'director' : 'council'}}",
        "cases": {
          "pm": [
            {
              "id": "route_to_pm",
              "type": "tool_call",
              "description": "Assign the change order to the project manager for approval",
              "tool_name": "update_record",
              "input": {
                "table_id": "change_orders",
                "record_id": "{{trigger.button_pressed.record_id}}",
                "data": {
                  "status": "Pending Approval",
                  "approval_level": "Project Manager",
                  "approver": "{{get_project.output.project_manager}}"
                }
              }
            },
            {
              "id": "notify_pm",
              "type": "tool_call",
              "description": "Notify the project manager to review",
              "tool_name": "send_notification",
              "input": {
                "user_email": "{{get_project.output.project_manager}}",
                "title": "Change order pending your approval (${{get_change_order.output.amount}})",
                "message": "A change order for {{get_project.output.project_name}} requires your approval. Amount: ${{get_change_order.output.amount}}. Justification: {{get_change_order.output.justification}}"
              }
            }
          ],
          "director": [
            {
              "id": "route_to_director",
              "type": "tool_call",
              "description": "Assign the change order to the public works director",
              "tool_name": "update_record",
              "input": {
                "table_id": "change_orders",
                "record_id": "{{trigger.button_pressed.record_id}}",
                "data": {
                  "status": "Pending Approval",
                  "approval_level": "Public Works Director",
                  "approver": "pw_director@city.gov"
                }
              }
            },
            {
              "id": "notify_director",
              "type": "tool_call",
              "description": "Notify the director to review",
              "tool_name": "send_notification",
              "input": {
                "user_email": "pw_director@city.gov",
                "title": "Change order requires director approval (${{get_change_order.output.amount}})",
                "message": "A change order for {{get_project.output.project_name}} exceeds the PM threshold. Amount: ${{get_change_order.output.amount}}. Category: {{get_change_order.output.category}}. Justification: {{get_change_order.output.justification}}"
              }
            }
          ],
          "council": [
            {
              "id": "route_to_council",
              "type": "tool_call",
              "description": "Flag the change order for city council agenda",
              "tool_name": "update_record",
              "input": {
                "table_id": "change_orders",
                "record_id": "{{trigger.button_pressed.record_id}}",
                "data": {
                  "status": "Pending Council Approval",
                  "approval_level": "City Council"
                }
              }
            },
            {
              "id": "notify_council_liaison",
              "type": "tool_call",
              "description": "Notify the council liaison to add this to the next agenda",
              "tool_name": "send_notification",
              "input": {
                "user_email": "council_liaison@city.gov",
                "title": "Change order over $100K requires council action",
                "message": "{{get_project.output.project_name}} has a change order for ${{get_change_order.output.amount}} that must go to council for approval. Category: {{get_change_order.output.category}}. Please add to the next council meeting agenda."
              }
            }
          ]
        }
      },
      {
        "id": "log_comment",
        "type": "tool_call",
        "description": "Add an audit trail comment on the change order",
        "tool_name": "create_record_comments",
        "input": {
          "table_id": "change_orders",
          "record_id": "{{trigger.button_pressed.record_id}}",
          "comment": "Change order submitted for review. Amount (${{get_change_order.output.amount}}) is within remaining contingency (${{get_project.output.contingency_remaining}}). Routed for approval."
        }
      },
      {
        "id": "get_contractor",
        "type": "tool_call",
        "description": "Look up the contractor to send a status update",
        "tool_name": "get_record",
        "input": {
          "table_id": "contractors",
          "record_id": "{{get_change_order.output.contractor_id}}"
        }
      },
      {
        "id": "notify_contractor",
        "type": "tool_call",
        "description": "Inform the contractor that their change order is under review",
        "tool_name": "gmail_send_email",
        "input": {
          "to": "{{get_contractor.output.contact_email}}",
          "subject": "Change Order Under Review - {{get_project.output.project_name}}",
          "body": "{{get_contractor.output.company_name}},\n\nYour change order for ${{get_change_order.output.amount}} on {{get_project.output.project_name}} has been submitted for review. Please do not begin work on the changed scope until you receive formal written approval.\n\nCity of Public Works Department"
        }
      }
    ]
  }
]
```

## Results

| Metric | Before | After |
|--------|--------|-------|
| Average change order cycle time | 14 days | 4 days |
| Work started before approval | 30% of change orders | 0% (contractor notified to wait) |
| Change orders routed to wrong authority | 18% | 0% (threshold-based routing) |
| Budget overruns from unapproved changes | 3 per year | 0 (contingency check enforced) |

-> [Set up this workflow on Lotics](https://lotics.ai)

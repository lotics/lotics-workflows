# Client Project Milestone Approval

## The problem

Professional services firms juggle 30-50 active client engagements at once. When a project manager marks a milestone as complete, the delivery lead must review it, the client must approve it, and billing must be triggered — all before the next phase can start. Without automation, milestone approvals sit in inboxes for 3-5 days, delaying revenue recognition and blocking downstream teams.

## Data model

| Table | Key fields | Purpose |
|-------|-----------|---------|
| Projects | project_name, client, delivery_lead, status, start_date, end_date | Master record for each client engagement |
| Milestones | milestone_name, project (link), phase, status, due_date, deliverables_complete, approved_by, approved_at | Tracks each deliverable checkpoint within a project |
| Billing Events | milestone (link), project (link), invoice_amount, billing_status, triggered_at | Records billing actions tied to approved milestones |

## Example prompts

- "When a milestone status changes to 'Delivered', notify the delivery lead for review and send the client an approval email. If the client approves, create a billing event automatically."
- "Build a workflow that handles milestone sign-off: internal review first, then client approval via email, then auto-generate the billing record once approved."

## Workflow

**Trigger:** `record_updated` on the Milestones table — fires when `status` changes to `Delivered`.

```json
[
  {
    "id": "get_project",
    "type": "tool_call",
    "description": "Fetch the parent project to get the delivery lead and client details",
    "tool_name": "get_record",
    "input": {
      "table_name": "Projects",
      "record_id": "{{trigger.record_updated.next_data.project}}"
    }
  },
  {
    "id": "notify_delivery_lead",
    "type": "tool_call",
    "description": "Send an in-app notification to the delivery lead for internal review",
    "tool_name": "send_notification",
    "input": {
      "member_id": "{{get_project.output.delivery_lead}}",
      "title": "Milestone ready for review",
      "message": "{{trigger.record_updated.next_data.milestone_name}} on {{get_project.output.project_name}} has been marked Delivered. Please review before client approval."
    }
  },
  {
    "id": "wait_for_internal_review",
    "type": "wait_for_event",
    "description": "Wait for the delivery lead to update milestone status to 'Reviewed'",
    "event": "record_updated",
    "condition": "{{trigger.record_updated.next_data.status == 'Reviewed'}}"
  },
  {
    "id": "email_client_for_approval",
    "type": "tool_call",
    "description": "Email the client requesting formal approval of the milestone",
    "tool_name": "gmail_send_email",
    "input": {
      "to": "{{get_project.output.client_email}}",
      "subject": "Approval Required: {{trigger.record_updated.next_data.milestone_name}} — {{get_project.output.project_name}}",
      "body": "Hi,\n\nThe milestone \"{{trigger.record_updated.next_data.milestone_name}}\" for project {{get_project.output.project_name}} is ready for your review and approval.\n\nPlease reply to this email with your approval or any feedback.\n\nThank you."
    }
  },
  {
    "id": "wait_for_client_reply",
    "type": "wait_for_event",
    "description": "Wait for the client to reply via email",
    "event": "receive_gmail_email",
    "condition": "{{true}}"
  },
  {
    "id": "analyze_reply",
    "type": "tool_call",
    "description": "Use LLM to determine if the client approved or requested changes",
    "tool_name": "llm_generate_text",
    "input": {
      "prompt": "Analyze this client email reply and determine if they approved the milestone or requested changes. Reply with exactly 'approved' or 'changes_requested'.\n\nEmail body: {{wait_for_client_reply.output.body}}"
    }
  },
  {
    "id": "check_approval",
    "type": "if",
    "description": "Branch based on whether the client approved",
    "condition": "{{analyze_reply.output.text == 'approved'}}",
    "then": [
      {
        "id": "mark_approved",
        "type": "tool_call",
        "description": "Update milestone status to Approved",
        "tool_name": "update_record",
        "input": {
          "table_name": "Milestones",
          "record_id": "{{trigger.record_updated.record_id}}",
          "data": {
            "status": "Approved",
            "approved_at": "{{now()}}"
          }
        }
      },
      {
        "id": "create_billing_event",
        "type": "tool_call",
        "description": "Create a billing event to trigger invoicing",
        "tool_name": "create_record",
        "input": {
          "table_name": "Billing Events",
          "data": {
            "milestone": "{{trigger.record_updated.record_id}}",
            "project": "{{trigger.record_updated.next_data.project}}",
            "invoice_amount": "{{trigger.record_updated.next_data.invoice_amount}}",
            "billing_status": "Pending",
            "triggered_at": "{{now()}}"
          }
        }
      }
    ],
    "else": [
      {
        "id": "mark_changes_requested",
        "type": "tool_call",
        "description": "Update milestone status back to In Progress with client feedback",
        "tool_name": "update_record",
        "input": {
          "table_name": "Milestones",
          "record_id": "{{trigger.record_updated.record_id}}",
          "data": {
            "status": "Changes Requested"
          }
        }
      },
      {
        "id": "notify_pm_of_feedback",
        "type": "tool_call",
        "description": "Notify the project manager about client feedback",
        "tool_name": "send_notification",
        "input": {
          "member_id": "{{trigger.record_updated.next_data.assigned_to}}",
          "title": "Client requested changes",
          "message": "The client has requested changes on {{trigger.record_updated.next_data.milestone_name}}. Check the email thread for details."
          }
      }
    ]
  }
]
```

## Results

| Metric | Before | After |
|--------|--------|-------|
| Avg. milestone approval time | 4.2 days | 1.1 days |
| Billing events missed per quarter | 6-8 | 0 |
| Manual follow-up emails per week | 15 | 0 |

-> [Set up this workflow on Lotics](https://lotics.ai)

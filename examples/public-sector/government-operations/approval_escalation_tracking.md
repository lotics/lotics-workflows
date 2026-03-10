# Approval Escalation Tracking

## The problem

County budget requests, policy changes, and procurement authorizations require multi-level approval chains — analyst, division chief, deputy director, and sometimes the board. Requests stall at a single approver for an average of 6 days, and staff have no visibility into where a request is stuck. In Q3 last year, 22% of procurement requests missed their fiscal-year deadline because approvals were not escalated when an approver was unavailable.

## Data model

| Table | Key fields | Purpose |
|-------|-----------|---------|
| Approval Requests | title, request_type, requested_by, current_approver, approval_level, status, due_date, amount, justification | Track each request through the approval chain |
| Approval Chain | request_type, level, approver_email, approver_name, escalation_approver_email, max_wait_days | Define who approves at each level and when to escalate |

## Example prompts

- "Every morning at 8 AM, check for approval requests that have been waiting more than the allowed days at their current level. If overdue, escalate to the backup approver and notify the requestor."
- "Build a daily escalation workflow that flags stalled approvals, reassigns them, and sends a summary to the department head."

## Workflow

**Trigger:** `recurring_schedule` — runs daily at 08:00 AM local time.

```json
[
  {
    "id": "find_stalled",
    "type": "tool_call",
    "description": "Query all approval requests still in pending status",
    "tool_name": "query_records",
    "input": {
      "table_id": "approval_requests",
      "filters": {
        "status": "Pending Approval"
      }
    }
  },
  {
    "id": "process_each",
    "type": "foreach",
    "description": "Evaluate each pending request for escalation",
    "items": "{{find_stalled.output.records}}",
    "steps": [
      {
        "id": "get_chain_rule",
        "type": "tool_call",
        "description": "Look up the escalation rule for this request type and level",
        "tool_name": "query_records",
        "input": {
          "table_id": "approval_chain",
          "filters": {
            "request_type": "{{process_each.item.request_type}}",
            "level": "{{process_each.item.approval_level}}"
          }
        }
      },
      {
        "id": "check_overdue",
        "type": "if",
        "description": "Determine whether the request has exceeded the max wait days at this level",
        "condition": "{{(Date.now() - new Date(process_each.item.last_status_change).getTime()) / 86400000 > get_chain_rule.output.records[0].max_wait_days}}",
        "then": [
          {
            "id": "escalate_request",
            "type": "tool_call",
            "description": "Reassign to the escalation approver",
            "tool_name": "update_record",
            "input": {
              "table_id": "approval_requests",
              "record_id": "{{process_each.item.id}}",
              "data": {
                "current_approver": "{{get_chain_rule.output.records[0].escalation_approver_email}}",
                "status": "Escalated"
              }
            }
          },
          {
            "id": "notify_escalation_approver",
            "type": "tool_call",
            "description": "Alert the backup approver that a request needs their attention",
            "tool_name": "send_notification",
            "input": {
              "user_email": "{{get_chain_rule.output.records[0].escalation_approver_email}}",
              "title": "Escalated approval: {{process_each.item.title}}",
              "message": "This {{process_each.item.request_type}} request ({{process_each.item.amount}}) has been waiting {{get_chain_rule.output.records[0].max_wait_days}}+ days. Please review and approve or deny."
            }
          },
          {
            "id": "notify_requestor",
            "type": "tool_call",
            "description": "Let the requestor know their approval has been escalated",
            "tool_name": "send_notification",
            "input": {
              "user_email": "{{process_each.item.requested_by}}",
              "title": "Your request has been escalated",
              "message": "Your {{process_each.item.request_type}} request '{{process_each.item.title}}' was escalated because the original approver did not respond within {{get_chain_rule.output.records[0].max_wait_days}} days. It is now with {{get_chain_rule.output.records[0].escalation_approver_email}}."
            }
          }
        ],
        "else": []
      }
    ]
  }
]
```

## Results

| Metric | Before | After |
|--------|--------|-------|
| Average approval cycle time | 11 days | 4 days |
| Requests missing fiscal deadline | 22% | Under 5% |
| Staff time spent chasing approvers | 6 hours/week | 0 (automated) |
| Visibility into request status | Email threads | Real-time dashboard |

-> [Set up this workflow on Lotics](https://lotics.ai)

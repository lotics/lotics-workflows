# Overdue Work Order Escalation

## The problem

Field service companies commit to SLA response times -- 4 hours for emergency, 24 hours for standard, 72 hours for routine. But with 100+ active work orders, dispatchers lose track of aging tickets. About 12% of work orders breach SLA, costing $500-2,000 per violation in contract penalties and damaging customer relationships.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Work Orders | title, status, priority, created_at, sla_deadline, assigned_tech (link), customer (link) | Service requests with SLA tracking |
| Technicians | name, email, phone, manager (link) | Field technicians |
| Managers | name, email, phone | Service managers for escalation |

## Example prompts

- "Every hour, check for work orders approaching their SLA deadline. If a work order is within 1 hour of its SLA and still open, notify the assigned tech and their manager. If it's past the SLA, escalate to the service director."
- "Run an hourly check on all open work orders. Escalate any that are about to breach SLA by notifying the tech and manager, and flag overdue ones for the director."

## Workflow

**Trigger:** Recurring schedule, every hour

```json
[
  {
    "id": "get_open_orders",
    "type": "tool_call",
    "description": "Query all open work orders that have an SLA deadline",
    "tool_name": "query_records",
    "input": {
      "table_name": "Work Orders",
      "filters": {
        "status": ["Open", "Assigned", "In Progress"]
      }
    }
  },
  {
    "id": "check_each_order",
    "type": "foreach",
    "description": "Evaluate SLA status for each open work order",
    "items": "{{get_open_orders.output.records}}",
    "steps": [
      {
        "id": "evaluate_sla",
        "type": "switch",
        "description": "Route based on SLA proximity",
        "expression": "{{new Date(check_each_order.item.sla_deadline) < new Date() ? 'breached' : (new Date(check_each_order.item.sla_deadline) - new Date()) < 3600000 ? 'at_risk' : 'ok'}}",
        "cases": {
          "breached": [
            {
              "id": "mark_breached",
              "type": "tool_call",
              "description": "Update work order status to SLA Breached",
              "tool_name": "update_record",
              "input": {
                "table_name": "Work Orders",
                "record_id": "{{check_each_order.item.id}}",
                "data": {
                  "status": "SLA Breached"
                }
              }
            },
            {
              "id": "notify_director",
              "type": "tool_call",
              "description": "Send escalation notification for breached SLA",
              "tool_name": "send_notification",
              "input": {
                "message": "SLA BREACH: Work order \"{{check_each_order.item.title}}\" ({{check_each_order.item.priority}} priority) has passed its SLA deadline of {{check_each_order.item.sla_deadline}}. Immediate action required.",
                "record_id": "{{check_each_order.item.id}}",
                "table_name": "Work Orders"
              }
            },
            {
              "id": "log_breach",
              "type": "tool_call",
              "description": "Add a comment documenting the SLA breach",
              "tool_name": "create_record_comments",
              "input": {
                "table_name": "Work Orders",
                "record_id": "{{check_each_order.item.id}}",
                "comment": "SLA breached at {{new Date().toISOString()}}. Deadline was {{check_each_order.item.sla_deadline}}. Escalated to service director."
              }
            }
          ],
          "at_risk": [
            {
              "id": "warn_tech",
              "type": "tool_call",
              "description": "Notify the assigned technician about approaching SLA",
              "tool_name": "send_notification",
              "input": {
                "message": "SLA WARNING: Work order \"{{check_each_order.item.title}}\" is due in less than 1 hour (deadline: {{check_each_order.item.sla_deadline}}). Please update status or complete the job.",
                "record_id": "{{check_each_order.item.id}}",
                "table_name": "Work Orders"
              }
            }
          ],
          "ok": []
        }
      }
    ]
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| SLA breach rate | 12% of work orders | Under 3% |
| SLA penalty costs | $4,000-8,000/month | Under $1,000/month |
| Manager awareness of at-risk tickets | Reactive (after breach) | 1 hour advance warning |

-> [Set up this workflow on Lotics](https://lotics.ai)

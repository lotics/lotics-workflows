# Budget Overrun Alert

## The problem

Construction projects frequently exceed budgets because cost overruns in individual line items go unnoticed until the monthly finance review. By then, a $12,000 concrete overspend has compounded into a $80,000 problem across related trades. Project controllers review 200+ cost entries per week but only catch overruns when they're already 15-20% over budget.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Projects | project_name, total_budget, contingency_budget, project_manager (link) | Master project record |
| Cost Lines | project (link), trade, description, budgeted_amount, actual_amount, invoice_date, vendor | Individual cost entries against budget |
| Change Orders | project (link), description, amount, status, requested_by, approved_by | Approved scope changes that adjust budget |

## Example prompts

- "When a cost line is created or updated, check if the total actual spend for that trade exceeds 90% of the budgeted amount. If so, lock the cost line and send a notification to the project manager with the trade name, overspend amount, and a link to the record."
- "Whenever a new cost entry is submitted, compare total actuals against budget for that trade category. If it's over 90%, alert the PM and lock the record to prevent further spend without approval."

## Workflow

**Trigger:** When a form is submitted for the Cost Lines table

```json
[
  {
    "id": "get_trade_costs",
    "type": "tool_call",
    "description": "Aggregate total actual spend for this trade on this project",
    "tool_name": "aggregate_records",
    "input": {
      "table_name": "Cost Lines",
      "filters": {
        "project": "{{trigger.form_submitted.data.project}}",
        "trade": "{{trigger.form_submitted.data.trade}}"
      },
      "aggregation": {
        "field": "actual_amount",
        "function": "sum"
      }
    }
  },
  {
    "id": "get_trade_budget",
    "type": "tool_call",
    "description": "Query all cost lines for this trade to get total budgeted amount",
    "tool_name": "aggregate_records",
    "input": {
      "table_name": "Cost Lines",
      "filters": {
        "project": "{{trigger.form_submitted.data.project}}",
        "trade": "{{trigger.form_submitted.data.trade}}"
      },
      "aggregation": {
        "field": "budgeted_amount",
        "function": "sum"
      }
    }
  },
  {
    "id": "get_project",
    "type": "tool_call",
    "description": "Fetch the project record for PM details",
    "tool_name": "get_record",
    "input": {
      "table_name": "Projects",
      "record_id": "{{trigger.form_submitted.data.project}}"
    }
  },
  {
    "id": "check_overrun",
    "type": "if",
    "description": "Check if actual spend exceeds 90% of budget for this trade",
    "condition": "{{get_trade_costs.output.result > (get_trade_budget.output.result * 0.9)}}",
    "then": [
      {
        "id": "lock_cost_line",
        "type": "tool_call",
        "description": "Lock the cost line record to prevent further unapproved spend",
        "tool_name": "lock_record",
        "input": {
          "table_name": "Cost Lines",
          "record_id": "{{trigger.form_submitted.record_id}}"
        }
      },
      {
        "id": "alert_pm",
        "type": "tool_call",
        "description": "Send an in-app notification to the project manager",
        "tool_name": "send_notification",
        "input": {
          "message": "Budget alert: {{trigger.form_submitted.data.trade}} on {{get_project.output.record.project_name}} is at {{Math.round((get_trade_costs.output.result / get_trade_budget.output.result) * 100)}}% of budget (${{get_trade_costs.output.result}} of ${{get_trade_budget.output.result}}). The cost line has been locked pending review.",
          "record_id": "{{trigger.form_submitted.record_id}}",
          "table_name": "Cost Lines"
        }
      },
      {
        "id": "add_comment",
        "type": "tool_call",
        "description": "Add a comment to the cost line explaining the lock",
        "tool_name": "create_record_comments",
        "input": {
          "table_name": "Cost Lines",
          "record_id": "{{trigger.form_submitted.record_id}}",
          "comment": "Automatically locked: {{trigger.form_submitted.data.trade}} trade has reached {{Math.round((get_trade_costs.output.result / get_trade_budget.output.result) * 100)}}% of budgeted amount. A change order or budget reallocation is required before additional costs can be entered."
        }
      }
    ],
    "else": []
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| Overrun detection time | Monthly (30 days) | Immediate on each cost entry |
| Average overrun before intervention | 15-20% over budget | Caught at 90% threshold |
| Unauthorized overspend incidents | 8-12 per project | Near zero (records locked) |

-> [Set up this workflow on Lotics](https://lotics.ai)

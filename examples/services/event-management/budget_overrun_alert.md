# Budget Overrun Alert

## The problem

Event budgets are tracked in spreadsheets that the coordinator updates after each expense. By the time a budget overrun is noticed, it's too late to adjust. A corporate event with a $50,000 budget can quietly exceed $55,000 because vendor costs, last-minute additions, and scope changes aren't reflected in real time. The client discovers the overrun in the final invoice, leading to disputes and damaged relationships. The company absorbs $8,000-$12,000/year in unrecoverable cost overruns.

## Data model

| Table | Key fields | Purpose |
|-------|-----------|---------|
| Events | event_name, event_date, client_name, total_budget, coordinator_id | Event master record with budget |
| Expenses | event_id, category, description, amount, vendor_id, approved_by, date | Individual expense line items |

## Example prompts

- "When a new expense is recorded, check if total expenses for that event exceed 85% of the budget. If so, notify the coordinator and lock the expense record for review."
- "Alert me whenever an event's spending crosses 85% of its total budget so I can review before it overruns."

## Workflow

**Trigger:** When an expense record is created

```json
[
  {
    "id": "get_event",
    "type": "tool_call",
    "description": "Fetch the event record for budget information",
    "tool_name": "get_record",
    "input": {
      "table_name": "Events",
      "record_id": "{{trigger.record_updated.next_data.event_id}}"
    }
  },
  {
    "id": "total_spent",
    "type": "tool_call",
    "description": "Calculate total expenses for this event",
    "tool_name": "aggregate_records",
    "input": {
      "table_name": "Expenses",
      "filters": {
        "event_id": "{{trigger.record_updated.next_data.event_id}}"
      },
      "aggregations": [
        {"field": "amount", "function": "sum"}
      ]
    }
  },
  {
    "id": "check_threshold",
    "type": "if",
    "description": "Check if spending has crossed 85% of budget",
    "condition": "{{total_spent.output.sum > get_event.output.total_budget * 0.85}}",
    "then": [
      {
        "id": "lock_expense",
        "type": "tool_call",
        "description": "Lock the expense record to prevent further unreviewed spending",
        "tool_name": "lock_record",
        "input": {
          "table_name": "Expenses",
          "record_id": "{{trigger.record_updated.next_data.id}}"
        }
      },
      {
        "id": "alert_coordinator",
        "type": "tool_call",
        "description": "Notify the event coordinator about the budget threshold",
        "tool_name": "send_notification",
        "input": {
          "message": "BUDGET ALERT: {{get_event.output.event_name}} has spent ${{total_spent.output.sum}} of ${{get_event.output.total_budget}} budget (over 85%). Latest expense: {{trigger.record_updated.next_data.description}} for ${{trigger.record_updated.next_data.amount}}. The expense has been locked for your review.",
          "member_id": "{{get_event.output.coordinator_id}}"
        }
      },
      {
        "id": "log_alert",
        "type": "tool_call",
        "description": "Add a comment to the event record documenting the alert",
        "tool_name": "create_record_comments",
        "input": {
          "table_name": "Events",
          "record_id": "{{get_event.output.id}}",
          "comment": "Budget threshold alert triggered. Total spent: ${{total_spent.output.sum}} / ${{get_event.output.total_budget}} ({{total_spent.output.sum / get_event.output.total_budget * 100}}%). Latest expense locked for review."
        }
      }
    ],
    "else": []
  }
]
```

## Results

| Metric | Before | After |
|--------|--------|-------|
| Budget overruns discovered after event | 60% of overruns | 0% (caught at 85%) |
| Annual unrecoverable cost overruns | $8,000-$12,000 | Under $1,000 |
| Client disputes over final invoices | 3-4/year | 0/year |

-> [Set up this workflow on Lotics](https://lotics.ai)

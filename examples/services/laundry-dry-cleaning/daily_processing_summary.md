# Daily Processing Pipeline Summary

## The problem

A dry cleaning operation with 3 processing stations (sorting, cleaning, pressing) has no visibility into how many orders are stuck at each stage. The manager walks the floor twice a day to count orders manually and identify bottlenecks. When the pressing station falls behind, orders miss their promised pickup time and customers are disappointed. Without daily data, staffing decisions and capacity planning are guesswork.

## Data model

| Table | Key fields | Purpose |
|-------|-----------|---------|
| Orders | order_number, status, service_type, received_at, promised_at, completed_at, assigned_station | Order pipeline tracking |

## Example prompts

- "Every day at 6 PM, count how many orders are at each processing stage and how many are overdue past their promised time. Send me a summary."
- "Generate a daily operations dashboard showing orders by status, overdue count, and average processing time."

## Workflow

**Trigger:** Recurring schedule, daily at 18:00

```json
[
  {
    "id": "count_received",
    "type": "tool_call",
    "description": "Count orders in received/intake status",
    "tool_name": "aggregate_records",
    "input": {
      "table_name": "Orders",
      "filters": {
        "status": "received"
      },
      "aggregations": [
        {"field": "order_number", "function": "count"}
      ]
    }
  },
  {
    "id": "count_processing",
    "type": "tool_call",
    "description": "Count orders currently being processed",
    "tool_name": "aggregate_records",
    "input": {
      "table_name": "Orders",
      "filters": {
        "status": "processing"
      },
      "aggregations": [
        {"field": "order_number", "function": "count"}
      ]
    }
  },
  {
    "id": "count_ready",
    "type": "tool_call",
    "description": "Count orders ready for pickup but not yet collected",
    "tool_name": "aggregate_records",
    "input": {
      "table_name": "Orders",
      "filters": {
        "status": "ready_for_pickup"
      },
      "aggregations": [
        {"field": "order_number", "function": "count"}
      ]
    }
  },
  {
    "id": "find_overdue",
    "type": "tool_call",
    "description": "Find orders past their promised pickup time that aren't completed",
    "tool_name": "query_records",
    "input": {
      "table_name": "Orders",
      "filters": {
        "promised_at_before": "{{trigger.recurring_schedule.scheduled_at}}",
        "status_not": "completed"
      }
    }
  },
  {
    "id": "send_summary",
    "type": "tool_call",
    "description": "Send the daily pipeline summary to the manager",
    "tool_name": "send_notification",
    "input": {
      "message": "Daily Pipeline Summary ({{trigger.recurring_schedule.scheduled_at_date}}):\n\nReceived/Intake: {{count_received.output.count}} orders\nIn Processing: {{count_processing.output.count}} orders\nReady for Pickup: {{count_ready.output.count}} orders\nOverdue Orders: {{find_overdue.output.records.length}} orders\n\nReview overdue orders to reassign or expedite."
    }
  }
]
```

## Results

| Metric | Before | After |
|--------|--------|-------|
| Time to assess daily pipeline | 40 min (floor walk) | 0 min (automated report) |
| Overdue orders caught same day | 50% | 100% |
| On-time delivery rate | 82% | 95% |

-> [Set up this workflow on Lotics](https://lotics.ai)

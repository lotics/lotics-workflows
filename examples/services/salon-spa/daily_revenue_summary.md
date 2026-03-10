# Daily Revenue Summary Report

## The problem

A salon owner with 3 locations checks revenue by scrolling through appointment records at the end of each day, manually tallying completed services and product add-ons. This takes 30-45 minutes and the numbers are often wrong because cancelled appointments get included. Without a reliable daily snapshot, pricing and staffing decisions are based on gut feel rather than data.

## Data model

| Table | Key fields | Purpose |
|-------|-----------|---------|
| Appointments | client_name, service_type, therapist, date_time, status, total_charged, location, payment_method | Completed appointment records |
| Locations | location_name, manager_email | Salon branches |

## Example prompts

- "Every night at 9 PM, calculate total revenue from today's completed appointments grouped by location and email the summary to me."
- "Generate a daily revenue report at end of business showing completed appointments count, total revenue, and average ticket size per location."

## Workflow

**Trigger:** Recurring schedule, daily at 21:00

```json
[
  {
    "id": "query_locations",
    "type": "tool_call",
    "description": "Get all active salon locations",
    "tool_name": "query_records",
    "input": {
      "table_name": "Locations"
    }
  },
  {
    "id": "process_locations",
    "type": "foreach",
    "description": "Calculate revenue for each location",
    "items": "{{query_locations.output.records}}",
    "steps": [
      {
        "id": "aggregate_revenue",
        "type": "tool_call",
        "description": "Sum revenue for completed appointments at this location today",
        "tool_name": "aggregate_records",
        "input": {
          "table_name": "Appointments",
          "filters": {
            "status": "completed",
            "location": "{{item.location_name}}",
            "date_time_date": "{{trigger.recurring_schedule.scheduled_at_date}}"
          },
          "aggregations": [
            {"field": "total_charged", "function": "sum"},
            {"field": "total_charged", "function": "avg"},
            {"field": "total_charged", "function": "count"}
          ]
        }
      },
      {
        "id": "email_manager",
        "type": "tool_call",
        "description": "Email daily summary to the location manager",
        "tool_name": "gmail_send_email",
        "input": {
          "to": "{{item.manager_email}}",
          "subject": "Daily Revenue Summary - {{item.location_name}} - {{trigger.recurring_schedule.scheduled_at_date}}",
          "body": "Daily Revenue Report for {{item.location_name}}\n\nDate: {{trigger.recurring_schedule.scheduled_at_date}}\nCompleted Appointments: {{aggregate_revenue.output.count}}\nTotal Revenue: ${{aggregate_revenue.output.sum}}\nAverage Ticket: ${{aggregate_revenue.output.avg}}\n\nThis report is generated automatically from completed appointments only."
        }
      }
    ]
  }
]
```

## Results

| Metric | Before | After |
|--------|--------|-------|
| Time to compile daily revenue | 45 min/day across locations | 0 min (automated) |
| Reporting accuracy | ~85% (cancelled appts included) | 100% (filters by status) |
| Days with missing reports | 8-10/month | 0/month |

-> [Set up this workflow on Lotics](https://lotics.ai)

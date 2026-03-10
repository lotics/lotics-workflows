# Automated Storage Billing Calculation

## The problem

Container depots charge shipping lines daily storage fees based on container size, type, and duration in yard. Billing staff manually count days per container, apply tiered rates (free days, standard rate, overtime rate), and generate invoices weekly. With 300+ containers in the yard, a single billing run takes 4-6 hours and errors in free-day calculations or rate application cost the depot $2,000-5,000/month in undercharges.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Containers | container_number, size, type, owner_line, gate_in_date, gate_out_date, current_status | Container with yard dates |
| Storage Rates | owner_line, size, free_days, daily_rate, overtime_rate, overtime_after_days | Rate schedule per line |
| Storage Invoices | owner_line, period_start, period_end, total_amount, line_count, status | Monthly/weekly invoice |
| Storage Line Items | invoice_id, container_id, days_stored, free_days_used, billable_days, amount | Per-container billing detail |

## Example prompts

- "Every Monday, calculate storage charges for all containers in the yard and generate invoice records per shipping line."
- "Run weekly storage billing — compute days, apply rates, and create invoices for each container owner."

## Workflow

**Trigger:** Recurring weekly schedule every Monday at 07:00 UTC.

```json
[
  {
    "id": "get_containers_in_yard",
    "type": "tool_call",
    "description": "Query all containers currently stored in the yard",
    "tool_name": "query_records",
    "input": {
      "table_id": "containers",
      "filter": {
        "field": "current_status",
        "operator": "equals",
        "value": "in_yard"
      }
    }
  },
  {
    "id": "get_all_rates",
    "type": "tool_call",
    "description": "Fetch all storage rate schedules",
    "tool_name": "query_records",
    "input": {
      "table_id": "storage_rates"
    }
  },
  {
    "id": "calculate_billing",
    "type": "tool_call",
    "description": "Calculate storage charges for all containers using rate schedules",
    "tool_name": "llm_generate_text",
    "input": {
      "prompt": "Calculate weekly storage billing. Current date: {{NOW()}}. Containers in yard: {{JSON.stringify(get_containers_in_yard.output.data.map(function(c) { return { id: c.id, container_number: c.container_number, size: c.size, type: c.type, owner_line: c.owner_line, gate_in_date: c.gate_in_date } }))}}. Rate schedules: {{JSON.stringify(get_all_rates.output.data)}}. For each container, calculate: days_stored (from gate_in_date to now), free_days_used, billable_days, amount (using daily_rate for days up to overtime_after_days, overtime_rate after). Group results by owner_line. Return JSON: { \"by_owner\": { \"[owner_line]\": { \"containers\": [{ \"container_id\": string, \"container_number\": string, \"days_stored\": number, \"billable_days\": number, \"amount\": number }], \"total\": number } } }. Only return JSON."
    }
  },
  {
    "id": "get_owner_lines",
    "type": "tool_call",
    "description": "Get unique shipping line names from containers for invoice creation",
    "tool_name": "llm_generate_text",
    "input": {
      "prompt": "Extract unique owner_line values from: {{JSON.stringify(get_containers_in_yard.output.data.map(function(c) { return c.owner_line }))}}. Return a JSON array of unique strings. Only return JSON."
    }
  },
  {
    "id": "create_invoices",
    "type": "foreach",
    "description": "Create a storage invoice for each shipping line",
    "items": "{{JSON.parse(get_owner_lines.output.text)}}",
    "steps": [
      {
        "id": "create_invoice",
        "type": "tool_call",
        "description": "Create the storage invoice record for this shipping line",
        "tool_name": "create_record",
        "input": {
          "table_id": "storage_invoices",
          "data": {
            "owner_line": "{{create_invoices.item}}",
            "period_start": "{{DATE_ADD(NOW(), -7, 'days')}}",
            "period_end": "{{NOW()}}",
            "total_amount": "{{JSON.parse(calculate_billing.output.text).by_owner[create_invoices.item].total}}",
            "status": "draft"
          }
        }
      },
      {
        "id": "notify_billing_team",
        "type": "tool_call",
        "description": "Notify billing team that a new storage invoice is ready for review",
        "tool_name": "send_notification",
        "input": {
          "message": "Weekly storage invoice created for {{create_invoices.item}}: ${{JSON.parse(calculate_billing.output.text).by_owner[create_invoices.item].total}}. Please review and finalize.",
          "record_id": "{{create_invoice.output.data.id}}"
        }
      }
    ]
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| Time per weekly billing run | 4-6 hours | Under 15 minutes |
| Billing errors per month | $2,000-5,000 undercharged | Under $200 variance |
| Invoice generation delay | 1-2 days after period end | Same day |

→ [Set up this workflow on Lotics](https://lotics.ai)

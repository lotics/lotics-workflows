# Monthly Supply Usage Report

## The problem

Clinic administrators need monthly supply consumption data to negotiate vendor contracts, forecast budgets, and identify waste. Without automated reporting, the office manager exports spreadsheets from three different systems, manually cross-references purchase orders against inventory counts, and builds the report in Excel. This takes a full day each month and the numbers are often inconsistent because inventory was counted on different days.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Supply Usage | item_name, sku, category, quantity_used, usage_date, department, recorded_by | Daily consumption records |
| Supplies | item_name, sku, category, current_quantity, unit_cost, supplier | Current inventory snapshot |
| Purchase Orders | supplier, items, total_cost, status, order_date | Historical purchase records |
| Reports | report_name, report_type, period, file, generated_date, sent_to | Archive of generated reports |

## Example prompts

- "On the first of every month, aggregate last month's supply usage by category, generate an Excel report with totals and top-consumed items, and email it to the office manager and CFO."
- "Create a monthly workflow that pulls supply usage data, builds a report showing consumption by department and category, and sends it to the admin team."

## Workflow

**Trigger:** `recurring_schedule` — Runs on the 1st of each month at 9:00 AM.

```json
[
  {
    "id": "get_usage_data",
    "type": "tool_call",
    "description": "Aggregate last month's supply usage totals by category",
    "tool_name": "aggregate_records",
    "input": {
      "table_name": "Supply Usage",
      "group_by": "category",
      "aggregations": {
        "quantity_used": "sum"
      }
    }
  },
  {
    "id": "get_department_usage",
    "type": "tool_call",
    "description": "Aggregate last month's supply usage by department for cost allocation",
    "tool_name": "aggregate_records",
    "input": {
      "table_name": "Supply Usage",
      "group_by": "department",
      "aggregations": {
        "quantity_used": "sum"
      }
    }
  },
  {
    "id": "get_top_items",
    "type": "tool_call",
    "description": "Query the highest-volume items consumed last month",
    "tool_name": "query_records",
    "input": {
      "table_name": "Supply Usage",
      "sort": {
        "field": "quantity_used",
        "direction": "desc"
      },
      "limit": 20
    }
  },
  {
    "id": "generate_summary",
    "type": "tool_call",
    "description": "Use LLM to produce a narrative summary of usage trends",
    "tool_name": "llm_generate_text",
    "input": {
      "prompt": "Write a concise monthly supply usage summary for clinic leadership. Category totals: {{JSON.stringify(get_usage_data.output)}}. Department breakdown: {{JSON.stringify(get_department_usage.output)}}. Top 20 consumed items: {{JSON.stringify(get_top_items.output.records)}}. Highlight any category that increased more than 20% and call out the top 5 most expensive items. Keep it under 300 words."
    }
  },
  {
    "id": "generate_report",
    "type": "tool_call",
    "description": "Generate an Excel report from the usage data template",
    "tool_name": "generate_excel_from_template",
    "input": {
      "template_name": "Monthly Supply Usage Report",
      "data": {
        "category_summary": "{{get_usage_data.output}}",
        "department_summary": "{{get_department_usage.output}}",
        "top_items": "{{get_top_items.output.records}}",
        "narrative": "{{generate_summary.output.text}}"
      }
    }
  },
  {
    "id": "archive_report",
    "type": "tool_call",
    "description": "Save the generated report to the reports table for future reference",
    "tool_name": "create_record",
    "input": {
      "table_name": "Reports",
      "data": {
        "report_name": "Supply Usage Report",
        "report_type": "Monthly",
        "period": "{{new Date(Date.now() - 86400000).toISOString().slice(0, 7)}}",
        "generated_date": "{{new Date().toISOString().split('T')[0]}}"
      }
    }
  },
  {
    "id": "email_report",
    "type": "tool_call",
    "description": "Email the usage report to the office manager and CFO",
    "tool_name": "gmail_send_email",
    "input": {
      "to": "office.manager@clinic.com, cfo@clinic.com",
      "subject": "Monthly Supply Usage Report - {{new Date(Date.now() - 86400000).toISOString().slice(0, 7)}}",
      "body": "Hi,\n\nPlease find attached the monthly supply usage report.\n\nSummary:\n{{generate_summary.output.text}}\n\nThe full Excel report is attached with category breakdowns, department allocation, and top consumed items.\n\nPlease reach out if you have questions.\n\nBest,\nSupply Management"
    }
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| Time to produce monthly report | 8 hrs | 0 (automated) |
| Report delivery date | 5th-8th of month | 1st of month |
| Data consistency issues | 2-3 per report | 0 |
| Budget forecast accuracy | +/- 15% | +/- 5% |

-> [Set up this workflow on Lotics](https://lotics.ai)

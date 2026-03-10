# Branch Daily Sales Report

## The problem

A retail chain with 14 branches has store managers email daily sales summaries to the regional director in different formats — some use spreadsheets, some send bullet points, a few forget entirely. The regional director spends 45 minutes each morning compiling numbers into a comparison view. Weekend and holiday reports frequently go missing. Without a standardized daily pulse, underperforming locations go unnoticed for weeks until the monthly P&L review.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Branches | `branch_id`, `branch_name`, `region`, `manager_name`, `manager_email` | Store directory with management contacts |
| Daily Sales | `branch_id`, `date`, `total_revenue`, `transaction_count`, `avg_basket_size`, `top_category`, `notes` | Standardized daily sales submissions |
| Sales Targets | `branch_id`, `month`, `revenue_target`, `transaction_target` | Monthly targets for performance comparison |
| Daily Reports | `report_id`, `date`, `status`, `generated_at`, `file` | Generated consolidated report archive |

## Example prompts

- "Every evening at 8 PM, pull today's sales data from all branches, generate a consolidated PDF report comparing each branch against target, and email it to the regional director."
- "Aggregate daily sales across all 14 stores every night, flag any branch that missed its daily revenue target by more than 15%, and send a summary report to leadership."

## Workflow

**Trigger:** `recurring_schedule` — runs daily at 8:00 PM after all branches have closed.

```json
[
  {
    "id": "get_branches",
    "type": "tool_call",
    "description": "Fetch all active branch locations",
    "tool_name": "query_records",
    "input": {
      "table": "Branches"
    }
  },
  {
    "id": "get_today_sales",
    "type": "tool_call",
    "description": "Query all branch sales entries for today",
    "tool_name": "query_records",
    "input": {
      "table": "Daily Sales",
      "filters": {
        "date": "{{ new Date().toISOString().split('T')[0] }}"
      }
    }
  },
  {
    "id": "get_targets",
    "type": "tool_call",
    "description": "Fetch this month's sales targets for all branches",
    "tool_name": "query_records",
    "input": {
      "table": "Sales Targets",
      "filters": {
        "month": "{{ new Date().toISOString().slice(0, 7) }}"
      }
    }
  },
  {
    "id": "generate_summary",
    "type": "tool_call",
    "description": "Use LLM to generate a narrative summary highlighting top and bottom performers",
    "tool_name": "llm_generate_text",
    "input": {
      "prompt": "Generate a concise daily sales summary for {{ new Date().toISOString().split('T')[0] }}. Branch sales data: {{ JSON.stringify(get_today_sales.output.records) }}. Monthly targets: {{ JSON.stringify(get_targets.output.records) }}. Highlight branches exceeding target, flag any branch more than 15% below their daily pro-rated target, and note any branches with missing reports."
    }
  },
  {
    "id": "generate_pdf",
    "type": "tool_call",
    "description": "Generate a formatted PDF report from the daily sales data",
    "tool_name": "generate_pdf_from_html_template",
    "input": {
      "template": "daily_sales_report",
      "data": {
        "date": "{{ new Date().toISOString().split('T')[0] }}",
        "branches": "{{ get_today_sales.output.records }}",
        "targets": "{{ get_targets.output.records }}",
        "summary": "{{ generate_summary.output.text }}"
      }
    }
  },
  {
    "id": "save_report",
    "type": "tool_call",
    "description": "Archive the generated report record",
    "tool_name": "create_record",
    "input": {
      "table": "Daily Reports",
      "data": {
        "date": "{{ new Date().toISOString().split('T')[0] }}",
        "status": "Generated"
      }
    }
  },
  {
    "id": "attach_pdf",
    "type": "tool_call",
    "description": "Attach the PDF file to the report record",
    "tool_name": "add_files_to_record",
    "input": {
      "table": "Daily Reports",
      "record_id": "{{ save_report.output.record.id }}",
      "files": ["{{ generate_pdf.output.file_url }}"]
    }
  },
  {
    "id": "email_report",
    "type": "tool_call",
    "description": "Email the consolidated report to the regional director",
    "tool_name": "gmail_send_email",
    "input": {
      "to": "regional.director@company.com",
      "subject": "Daily Sales Report — {{ new Date().toISOString().split('T')[0] }}",
      "body": "{{ generate_summary.output.text }}\n\nThe full report is attached as a PDF.\n\nThis report was generated automatically at {{ new Date().toISOString() }}."
    }
  },
  {
    "id": "flag_underperformers",
    "type": "foreach",
    "description": "Notify managers of branches significantly below target",
    "items": "{{ get_today_sales.output.records.filter(function(s) { var target = get_targets.output.records.find(function(t) { return t.branch_id === s.branch_id }); return target && s.total_revenue < (target.revenue_target / 30) * 0.85; }) }}",
    "steps": [
      {
        "id": "notify_manager",
        "type": "tool_call",
        "description": "Send a performance alert to the branch manager",
        "tool_name": "send_notification",
        "input": {
          "message": "Daily sales at {{ item.branch_id }} came in at ${{ item.total_revenue }}, which is more than 15% below the daily pro-rated target. Please review staffing, promotions, and foot traffic for this location."
        }
      }
    ]
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| Time to compile daily report | 45 minutes | 0 minutes |
| Report delivery consistency | ~75% of days | 100% of days |
| Days to identify underperforming branch | 20-30 days | 1 day |
| Missing branch submissions flagged | Never | Same day |

-> [Set up this workflow on Lotics](https://lotics.ai)

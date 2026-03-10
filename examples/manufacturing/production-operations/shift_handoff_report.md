# Automated Shift Handoff Report

## The problem

A plastics injection molding plant operates three 8-hour shifts. At each handoff, the outgoing shift lead spends 30-45 minutes compiling a summary of completed runs, open issues, machine downtime, and scrap counts. Information is passed via paper logs or verbal handoffs, and incoming leads frequently miss critical details -- resulting in repeated setups, overlooked defects, and an estimated 6% loss in first-shift OEE due to poor continuity.

## Data model

| Table | Key fields | Purpose |
|-------|-----------|---------|
| Shift Logs | log_id, shift, date, line_id, operator, status | One record per shift per production line |
| Production Runs | run_id, shift_log_id, product, qty_produced, qty_scrapped, cycle_time, status | Individual runs completed during a shift |
| Downtime Events | event_id, shift_log_id, machine_id, reason, duration_minutes | Unplanned stops and their causes |
| Shift Reports | report_id, shift, date, summary_html, pdf_file | Generated handoff document |

## Example prompts

- "At the end of every shift, compile all production runs, downtime events, and scrap rates into a PDF report and email it to the incoming shift lead."
- "Generate an automatic shift handoff report at 6am, 2pm, and 10pm with production totals, top downtime reasons, and open issues."

## Workflow

**Trigger:** Recurring schedule at 05:50, 13:50, and 21:50 daily (10 minutes before each shift change)

```json
[
  {
    "id": "determine_shift",
    "type": "tool_call",
    "description": "Use LLM to determine the current shift label from the trigger time",
    "tool_name": "llm_generate_text",
    "input": {
      "prompt": "The current time is {{trigger.recurring_schedule.triggered_at}}. Shifts are: Night (22:00-06:00), Day (06:00-14:00), Afternoon (14:00-22:00). Return ONLY the shift name that is currently ending."
    }
  },
  {
    "id": "get_shift_logs",
    "type": "tool_call",
    "description": "Fetch all shift logs for the ending shift today",
    "tool_name": "query_records",
    "input": {
      "table_id": "shift_logs",
      "filters": {
        "shift": "{{determine_shift.output.text}}",
        "date": "{{trigger.recurring_schedule.triggered_at}}",
        "status": "Active"
      }
    }
  },
  {
    "id": "get_runs",
    "type": "tool_call",
    "description": "Query all production runs linked to the ending shift",
    "tool_name": "query_records",
    "input": {
      "table_id": "production_runs",
      "filters": {
        "shift": "{{determine_shift.output.text}}",
        "date": "{{trigger.recurring_schedule.triggered_at}}"
      }
    }
  },
  {
    "id": "get_downtime",
    "type": "tool_call",
    "description": "Query all downtime events for the ending shift",
    "tool_name": "query_records",
    "input": {
      "table_id": "downtime_events",
      "filters": {
        "shift": "{{determine_shift.output.text}}",
        "date": "{{trigger.recurring_schedule.triggered_at}}"
      }
    }
  },
  {
    "id": "generate_summary",
    "type": "tool_call",
    "description": "Use LLM to compile the shift data into a structured HTML summary",
    "tool_name": "llm_generate_text",
    "input": {
      "prompt": "Generate an HTML shift handoff report for the {{determine_shift.output.text}} shift. Production runs: {{get_runs.output.records}}. Downtime events: {{get_downtime.output.records}}. Include total units produced, total scrap, scrap rate %, top 3 downtime reasons by duration, and any runs with status 'In Progress' that need continuation."
    }
  },
  {
    "id": "generate_pdf",
    "type": "tool_call",
    "description": "Render the HTML summary into a PDF handoff report",
    "tool_name": "generate_pdf_from_html_template",
    "input": {
      "template_id": "shift_handoff_template",
      "data": {
        "shift": "{{determine_shift.output.text}}",
        "date": "{{trigger.recurring_schedule.triggered_at}}",
        "body_html": "{{generate_summary.output.text}}"
      }
    }
  },
  {
    "id": "save_report",
    "type": "tool_call",
    "description": "Create a shift report record and attach the PDF",
    "tool_name": "create_record",
    "input": {
      "table_id": "shift_reports",
      "data": {
        "shift": "{{determine_shift.output.text}}",
        "date": "{{trigger.recurring_schedule.triggered_at}}",
        "summary_html": "{{generate_summary.output.text}}"
      }
    }
  },
  {
    "id": "attach_pdf",
    "type": "tool_call",
    "description": "Attach the generated PDF to the report record",
    "tool_name": "add_files_to_record",
    "input": {
      "table_id": "shift_reports",
      "record_id": "{{save_report.output.record_id}}",
      "files": ["{{generate_pdf.output.file}}"]
    }
  },
  {
    "id": "email_lead",
    "type": "tool_call",
    "description": "Email the handoff report to the incoming shift lead",
    "tool_name": "gmail_send_email",
    "input": {
      "to": "shift-leads@factory.com",
      "subject": "Shift Handoff Report - {{determine_shift.output.text}} Shift - {{trigger.recurring_schedule.triggered_at}}",
      "body": "Attached is the handoff report for the {{determine_shift.output.text}} shift. Please review open issues before starting your shift.",
      "attachments": ["{{generate_pdf.output.file}}"]
    }
  }
]
```

## Results

| Metric | Before | After |
|--------|--------|-------|
| Handoff preparation time | 30-45 min per shift | 0 min (fully automated) |
| Missed handoff items | 3-5 per week | 0 (comprehensive report) |
| First-hour OEE loss | 6% | < 2% |

-> [Set up this workflow on Lotics](https://lotics.ai)

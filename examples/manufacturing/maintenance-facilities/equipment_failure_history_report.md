# Equipment Failure History Report via Webhook

## The problem

A semiconductor fab tracks equipment failures across 200+ tools, but generating failure history reports for engineering reviews requires a data analyst to manually query multiple systems, compile CSV exports, and build Excel reports. Engineers request these reports 3-4 times per week for root cause investigations and reliability reviews. Each report takes 2-3 hours to produce, and the data is often stale by the time it reaches the engineer. During critical yield excursion investigations, this delay can cost $50,000+ per day in lost wafer output.

## Data model

| Table | Key fields | Purpose |
|-------|-----------|---------|
| Equipment | equipment_id, name, tool_type, area, install_date, status | Tool registry |
| Failure Events | failure_id, equipment_id, failure_mode, severity, occurred_at, resolved_at, downtime_minutes, root_cause | Historical failure log |
| Work Orders | wo_id, equipment_id, type, status, completed_at, technician, parts_used | Maintenance records linked to failures |
| Failure Reports | report_id, equipment_id, requested_by, period_start, period_end, file | Generated report records |

## Example prompts

- "When a report request comes in via webhook with an equipment ID and date range, pull all failure events and maintenance records for that tool, generate an Excel summary, and email it to the requester."
- "Automatically generate equipment failure history reports on demand when triggered by our engineering portal."

## Workflow

**Trigger:** Incoming webhook from the engineering portal with equipment_id, period_start, period_end, and requester_email

```json
[
  {
    "id": "get_equipment",
    "type": "tool_call",
    "description": "Fetch the equipment details",
    "tool_name": "get_record",
    "input": {
      "table_id": "equipment",
      "record_id": "{{trigger.receive_webhook.body.equipment_id}}"
    }
  },
  {
    "id": "get_failures",
    "type": "tool_call",
    "description": "Query all failure events for this equipment in the specified period",
    "tool_name": "query_records",
    "input": {
      "table_id": "failure_events",
      "filters": {
        "equipment_id": "{{trigger.receive_webhook.body.equipment_id}}",
        "occurred_at__gte": "{{trigger.receive_webhook.body.period_start}}",
        "occurred_at__lte": "{{trigger.receive_webhook.body.period_end}}"
      },
      "sort": "-occurred_at"
    }
  },
  {
    "id": "get_work_orders",
    "type": "tool_call",
    "description": "Query all maintenance work orders for this equipment in the period",
    "tool_name": "query_records",
    "input": {
      "table_id": "work_orders",
      "filters": {
        "equipment_id": "{{trigger.receive_webhook.body.equipment_id}}",
        "completed_at__gte": "{{trigger.receive_webhook.body.period_start}}",
        "completed_at__lte": "{{trigger.receive_webhook.body.period_end}}"
      },
      "sort": "-completed_at"
    }
  },
  {
    "id": "calc_metrics",
    "type": "tool_call",
    "description": "Calculate summary metrics for the failure history",
    "tool_name": "llm_generate_text",
    "input": {
      "prompt": "Given these failure events for equipment {{get_equipment.output.data.name}}:\n{{get_failures.output.records}}\n\nCalculate and return JSON with: total_failures (count), total_downtime_minutes (sum of downtime_minutes), mtbf_hours (mean time between failures in hours), top_failure_modes (array of {mode, count} sorted by count desc), top_root_causes (array of {cause, count} sorted by count desc)."
    }
  },
  {
    "id": "generate_report",
    "type": "tool_call",
    "description": "Generate an Excel report with failure history and summary metrics",
    "tool_name": "generate_excel_from_template",
    "input": {
      "template_id": "equipment_failure_report_template",
      "data": {
        "equipment_name": "{{get_equipment.output.data.name}}",
        "equipment_id": "{{get_equipment.output.data.equipment_id}}",
        "tool_type": "{{get_equipment.output.data.tool_type}}",
        "area": "{{get_equipment.output.data.area}}",
        "period_start": "{{trigger.receive_webhook.body.period_start}}",
        "period_end": "{{trigger.receive_webhook.body.period_end}}",
        "summary": "{{calc_metrics.output.text}}",
        "failures": "{{get_failures.output.records}}",
        "work_orders": "{{get_work_orders.output.records}}"
      }
    }
  },
  {
    "id": "save_report",
    "type": "tool_call",
    "description": "Create a report record to track the generated report",
    "tool_name": "create_record",
    "input": {
      "table_id": "failure_reports",
      "data": {
        "equipment_id": "{{trigger.receive_webhook.body.equipment_id}}",
        "requested_by": "{{trigger.receive_webhook.body.requester_email}}",
        "period_start": "{{trigger.receive_webhook.body.period_start}}",
        "period_end": "{{trigger.receive_webhook.body.period_end}}"
      }
    }
  },
  {
    "id": "attach_file",
    "type": "tool_call",
    "description": "Attach the Excel report to the report record",
    "tool_name": "add_files_to_record",
    "input": {
      "table_id": "failure_reports",
      "record_id": "{{save_report.output.record_id}}",
      "files": ["{{generate_report.output.file}}"]
    }
  },
  {
    "id": "email_report",
    "type": "tool_call",
    "description": "Email the completed report to the requesting engineer",
    "tool_name": "outlook_send_email",
    "input": {
      "to": "{{trigger.receive_webhook.body.requester_email}}",
      "subject": "Failure History Report: {{get_equipment.output.data.name}} ({{trigger.receive_webhook.body.period_start}} to {{trigger.receive_webhook.body.period_end}})",
      "body": "Your requested failure history report for {{get_equipment.output.data.name}} is attached.\n\nSummary:\n- Total failures: {{calc_metrics.output.text.total_failures}}\n- Total downtime: {{calc_metrics.output.text.total_downtime_minutes}} minutes\n- MTBF: {{calc_metrics.output.text.mtbf_hours}} hours\n\nThe full breakdown including failure modes, root causes, and associated work orders is in the attached Excel file.",
      "attachments": ["{{generate_report.output.file}}"]
    }
  }
]
```

## Results

| Metric | Before | After |
|--------|--------|-------|
| Report generation time | 2-3 hours | < 2 minutes |
| Data staleness | Hours to days | Real-time |
| Analyst hours on ad-hoc reports | 8-10 hrs/week | < 1 hr/week |

-> [Set up this workflow on Lotics](https://lotics.ai)

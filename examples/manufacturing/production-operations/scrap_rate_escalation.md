# Scrap Rate Escalation and Root Cause Logging

## The problem

A CNC machining shop produces precision automotive parts with a target scrap rate below 2%. Operators log scrap counts at the end of each batch, but nobody reviews the numbers until the weekly quality meeting. By then, a single miscalibrated machine can produce 500+ defective parts over several days. The plant estimates $18,000/month in avoidable scrap because problems are caught too late.

## Data model

| Table | Key fields | Purpose |
|-------|-----------|---------|
| Production Batches | batch_id, machine_id, part_number, qty_produced, qty_scrapped, operator, completed_at | Output record for each completed batch |
| Scrap Escalations | escalation_id, batch_id, machine_id, scrap_rate, severity, root_cause, corrective_action, status | Tracks flagged scrap events and resolution |
| Machines | machine_id, name, line, last_calibration, status | Machine registry with maintenance state |

## Example prompts

- "When a production batch is submitted with a scrap rate above 2%, create an escalation ticket, lock the batch record, and notify the quality supervisor. If it's above 5%, also flag the machine for maintenance review."
- "Automatically escalate any batch with excessive scrap, log a root cause investigation record, and hold the batch for review."

## Workflow

**Trigger:** A production batch record is submitted via form

```json
[
  {
    "id": "get_batch",
    "type": "tool_call",
    "description": "Retrieve the submitted batch record",
    "tool_name": "get_record",
    "input": {
      "table_id": "production_batches",
      "record_id": "{{trigger.record_submit.record_id}}"
    }
  },
  {
    "id": "calc_scrap_rate",
    "type": "if",
    "description": "Check if scrap rate exceeds the 2% threshold",
    "condition": "{{(get_batch.output.data.qty_scrapped / get_batch.output.data.qty_produced) * 100 > 2}}",
    "then": [
      {
        "id": "lock_batch",
        "type": "tool_call",
        "description": "Lock the batch record to prevent edits during investigation",
        "tool_name": "lock_record",
        "input": {
          "table_id": "production_batches",
          "record_id": "{{trigger.record_submit.record_id}}"
        }
      },
      {
        "id": "determine_severity",
        "type": "if",
        "description": "Determine if this is a critical escalation (above 5%)",
        "condition": "{{(get_batch.output.data.qty_scrapped / get_batch.output.data.qty_produced) * 100 > 5}}",
        "then": [
          {
            "id": "create_critical_escalation",
            "type": "tool_call",
            "description": "Create a critical severity escalation record",
            "tool_name": "create_record",
            "input": {
              "table_id": "scrap_escalations",
              "data": {
                "batch_id": "{{get_batch.output.data.batch_id}}",
                "machine_id": "{{get_batch.output.data.machine_id}}",
                "scrap_rate": "{{(get_batch.output.data.qty_scrapped / get_batch.output.data.qty_produced) * 100}}",
                "severity": "Critical",
                "status": "Open"
              }
            }
          },
          {
            "id": "flag_machine",
            "type": "tool_call",
            "description": "Flag the machine for immediate maintenance review",
            "tool_name": "update_record",
            "input": {
              "table_id": "machines",
              "record_id": "{{get_batch.output.data.machine_id}}",
              "data": {
                "status": "Maintenance Review Required"
              }
            }
          },
          {
            "id": "notify_critical",
            "type": "tool_call",
            "description": "Notify both quality supervisor and maintenance lead",
            "tool_name": "send_notification",
            "input": {
              "title": "CRITICAL: Scrap rate {{(get_batch.output.data.qty_scrapped / get_batch.output.data.qty_produced) * 100}}% on {{get_batch.output.data.machine_id}}",
              "message": "Batch {{get_batch.output.data.batch_id}} for part {{get_batch.output.data.part_number}} exceeded 5% scrap. Machine flagged for maintenance review. Batch locked pending investigation."
            }
          }
        ],
        "else": [
          {
            "id": "create_warning_escalation",
            "type": "tool_call",
            "description": "Create a warning severity escalation record",
            "tool_name": "create_record",
            "input": {
              "table_id": "scrap_escalations",
              "data": {
                "batch_id": "{{get_batch.output.data.batch_id}}",
                "machine_id": "{{get_batch.output.data.machine_id}}",
                "scrap_rate": "{{(get_batch.output.data.qty_scrapped / get_batch.output.data.qty_produced) * 100}}",
                "severity": "Warning",
                "status": "Open"
              }
            }
          },
          {
            "id": "notify_warning",
            "type": "tool_call",
            "description": "Notify the quality supervisor of elevated scrap",
            "tool_name": "send_notification",
            "input": {
              "title": "Elevated scrap rate on batch {{get_batch.output.data.batch_id}}",
              "message": "Scrap rate of {{(get_batch.output.data.qty_scrapped / get_batch.output.data.qty_produced) * 100}}% on machine {{get_batch.output.data.machine_id}} for part {{get_batch.output.data.part_number}}. Batch locked for review."
            }
          }
        ]
      }
    ],
    "else": []
  }
]
```

## Results

| Metric | Before | After |
|--------|--------|-------|
| Time to detect scrap anomaly | 3-7 days | Immediate |
| Monthly avoidable scrap cost | $18,000 | $3,200 |
| Unreviewed high-scrap batches | 40% | 0% |

-> [Set up this workflow on Lotics](https://lotics.ai)

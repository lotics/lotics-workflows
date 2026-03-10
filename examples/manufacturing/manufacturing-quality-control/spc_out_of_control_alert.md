# SPC Out-of-Control Alert and Containment

## The problem

A precision stamping facility runs statistical process control (SPC) on critical dimensions across 14 presses. Operators record measurements every 30 minutes, but SPC charts are reviewed only at the end of each shift by the quality engineer. When a process drifts out of control, 4-8 hours of production may pass before anyone acts. Each hour of undetected drift produces 200+ suspect parts that must be sorted or scrapped, costing $2,500 per incident on average.

## Data model

| Table | Key fields | Purpose |
|-------|-----------|---------|
| SPC Measurements | measurement_id, press_id, part_number, dimension_name, value, ucl, lcl, target, operator, measured_at | Individual measurement entries |
| Containment Actions | containment_id, press_id, part_number, trigger_measurement_id, qty_suspect, action_taken, status | Tracks containment of suspect product |
| Press Machines | press_id, name, line, current_job, status | Press registry and current state |

## Example prompts

- "When an operator logs an SPC measurement that falls outside the control limits, immediately notify the quality engineer, flag the press, and create a containment action for all parts produced since the last good measurement."
- "Detect out-of-control SPC readings in real time and trigger automatic containment and notification."

## Workflow

**Trigger:** A button is pressed by the operator after entering a suspect measurement

```json
[
  {
    "id": "get_measurement",
    "type": "tool_call",
    "description": "Fetch the flagged SPC measurement",
    "tool_name": "get_record",
    "input": {
      "table_id": "spc_measurements",
      "record_id": "{{trigger.button_pressed.record_id}}"
    }
  },
  {
    "id": "check_limits",
    "type": "if",
    "description": "Verify the measurement is actually outside control limits",
    "condition": "{{get_measurement.output.data.value > get_measurement.output.data.ucl || get_measurement.output.data.value < get_measurement.output.data.lcl}}",
    "then": [
      {
        "id": "find_last_good",
        "type": "tool_call",
        "description": "Find the most recent in-control measurement for this press and dimension",
        "tool_name": "query_records",
        "input": {
          "table_id": "spc_measurements",
          "filters": {
            "press_id": "{{get_measurement.output.data.press_id}}",
            "dimension_name": "{{get_measurement.output.data.dimension_name}}",
            "value__gte": "{{get_measurement.output.data.lcl}}",
            "value__lte": "{{get_measurement.output.data.ucl}}"
          },
          "sort": "-measured_at",
          "limit": 1
        }
      },
      {
        "id": "aggregate_suspect",
        "type": "tool_call",
        "description": "Count parts produced on this press since the last good measurement",
        "tool_name": "aggregate_records",
        "input": {
          "table_id": "spc_measurements",
          "filters": {
            "press_id": "{{get_measurement.output.data.press_id}}",
            "measured_at__gt": "{{find_last_good.output.records[0].measured_at}}"
          },
          "aggregate": "count"
        }
      },
      {
        "id": "flag_press",
        "type": "tool_call",
        "description": "Flag the press as out-of-control",
        "tool_name": "update_record",
        "input": {
          "table_id": "press_machines",
          "record_id": "{{get_measurement.output.data.press_id}}",
          "data": {
            "status": "Out of Control - Held"
          }
        }
      },
      {
        "id": "create_containment",
        "type": "tool_call",
        "description": "Create a containment action for the suspect production",
        "tool_name": "create_record",
        "input": {
          "table_id": "containment_actions",
          "data": {
            "press_id": "{{get_measurement.output.data.press_id}}",
            "part_number": "{{get_measurement.output.data.part_number}}",
            "trigger_measurement_id": "{{get_measurement.output.data.measurement_id}}",
            "qty_suspect": "{{aggregate_suspect.output.result}}",
            "action_taken": "Press held. All parts since {{find_last_good.output.records[0].measured_at}} quarantined for 100% sort.",
            "status": "Open"
          }
        }
      },
      {
        "id": "notify_qe",
        "type": "tool_call",
        "description": "Immediately notify the quality engineer",
        "tool_name": "send_notification",
        "input": {
          "title": "SPC ALERT: {{get_measurement.output.data.press_id}} out of control",
          "message": "Dimension {{get_measurement.output.data.dimension_name}} on part {{get_measurement.output.data.part_number}} measured {{get_measurement.output.data.value}} (UCL: {{get_measurement.output.data.ucl}}, LCL: {{get_measurement.output.data.lcl}}). Press held. Approximately {{aggregate_suspect.output.result}} suspect parts quarantined."
        }
      },
      {
        "id": "email_shift_lead",
        "type": "tool_call",
        "description": "Email the shift lead with containment details",
        "tool_name": "outlook_send_email",
        "input": {
          "to": "shift-lead@stamping-plant.com",
          "subject": "CONTAINMENT: Press {{get_measurement.output.data.press_id}} - {{get_measurement.output.data.part_number}}",
          "body": "Press {{get_measurement.output.data.press_id}} has been held due to an out-of-control reading on {{get_measurement.output.data.dimension_name}}.\n\nMeasured value: {{get_measurement.output.data.value}}\nUCL: {{get_measurement.output.data.ucl}} / LCL: {{get_measurement.output.data.lcl}}\nSuspect quantity: {{aggregate_suspect.output.result}} parts\n\nPlease initiate 100% sort of quarantined material and investigate root cause before restarting the press."
        }
      }
    ],
    "else": [
      {
        "id": "notify_false_alarm",
        "type": "tool_call",
        "description": "Notify the operator that the measurement is within limits",
        "tool_name": "send_notification",
        "input": {
          "title": "Measurement within control limits",
          "message": "The measurement of {{get_measurement.output.data.value}} for {{get_measurement.output.data.dimension_name}} is within UCL ({{get_measurement.output.data.ucl}}) and LCL ({{get_measurement.output.data.lcl}}). No action required."
        }
      }
    ]
  }
]
```

## Results

| Metric | Before | After |
|--------|--------|-------|
| Time to detect out-of-control condition | 4-8 hours | Immediate |
| Suspect parts per incident | 200+ | 30-50 |
| Average containment cost per incident | $2,500 | $400 |

-> [Set up this workflow on Lotics](https://lotics.ai)

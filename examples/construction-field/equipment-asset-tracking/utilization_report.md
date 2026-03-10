# Equipment Utilization Report

## The problem

Construction companies own equipment worth millions but have no clear picture of utilization rates. A $300,000 excavator sitting idle on one site while another site rents one for $4,500/week is a common and invisible problem. Fleet managers only discover underutilization during quarterly reviews, by which point they've wasted $50,000-100,000 in unnecessary rentals and idle ownership costs.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Equipment | name, type, serial_number, current_site (link), status, daily_rate, hours_this_week | Asset registry |
| Sites | site_name, project_manager_email | Job site directory |
| Utilization Reports | report_date, equipment (link), hours_used, utilization_pct, status_flag | Weekly utilization snapshots |

## Example prompts

- "Every Monday at 6 AM, calculate the utilization percentage for each piece of equipment based on hours used last week. Flag anything below 40% as underutilized, generate a summary report, and email it to the fleet manager."
- "Weekly, review all equipment hours and create a utilization report. Highlight idle equipment and equipment that could be redeployed to sites that are renting similar types."

## Workflow

**Trigger:** Recurring schedule, weekly on Monday at 6:00 AM

```json
[
  {
    "id": "get_all_equipment",
    "type": "tool_call",
    "description": "Query all active equipment with their weekly hours",
    "tool_name": "query_records",
    "input": {
      "table_name": "Equipment",
      "filters": {
        "status": "Active"
      }
    }
  },
  {
    "id": "create_utilization_records",
    "type": "foreach",
    "description": "Create a utilization record for each piece of equipment",
    "items": "{{get_all_equipment.output.records}}",
    "steps": [
      {
        "id": "calc_and_save",
        "type": "tool_call",
        "description": "Create utilization snapshot for this equipment",
        "tool_name": "create_record",
        "input": {
          "table_name": "Utilization Reports",
          "data": {
            "report_date": "{{new Date().toISOString().split('T')[0]}}",
            "equipment": "{{create_utilization_records.item.id}}",
            "hours_used": "{{create_utilization_records.item.hours_this_week}}",
            "utilization_pct": "{{Math.round((create_utilization_records.item.hours_this_week / 40) * 100)}}",
            "status_flag": "{{create_utilization_records.item.hours_this_week < 16 ? 'Underutilized' : create_utilization_records.item.hours_this_week >= 36 ? 'High' : 'Normal'}}"
          }
        }
      }
    ]
  },
  {
    "id": "reset_weekly_hours",
    "type": "foreach",
    "description": "Reset hours_this_week to zero for the new tracking period",
    "items": "{{get_all_equipment.output.records}}",
    "steps": [
      {
        "id": "reset_hours",
        "type": "tool_call",
        "description": "Reset weekly hour counter on equipment record",
        "tool_name": "update_record",
        "input": {
          "table_name": "Equipment",
          "record_id": "{{reset_weekly_hours.item.id}}",
          "data": {
            "hours_this_week": 0
          }
        }
      }
    ]
  },
  {
    "id": "generate_summary",
    "type": "tool_call",
    "description": "Use LLM to generate a fleet utilization summary with recommendations",
    "tool_name": "llm_generate_text",
    "input": {
      "prompt": "Generate a fleet utilization report for the week ending {{new Date().toISOString().split('T')[0]}}. Group equipment by utilization tier:\n- UNDERUTILIZED (below 40%): List each with name, type, location, hours, and daily rate. Calculate weekly idle cost.\n- NORMAL (40-90%): Count only.\n- HIGH (above 90%): List each and flag potential overuse/fatigue risk.\n\nEquipment data:\n{{JSON.stringify(get_all_equipment.output.records.map(e => ({name: e.name, type: e.type, site: e.current_site, hours: e.hours_this_week, daily_rate: e.daily_rate})))}}\n\nEnd with total fleet idle cost for the week and redeployment recommendations."
    }
  },
  {
    "id": "email_fleet_manager",
    "type": "tool_call",
    "description": "Email the utilization report to the fleet manager",
    "tool_name": "gmail_send_email",
    "input": {
      "to": "fleet@company.com",
      "subject": "Weekly Fleet Utilization Report — {{new Date().toISOString().split('T')[0]}}",
      "body": "{{generate_summary.output.text}}"
    }
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| Fleet utilization visibility | Quarterly review | Weekly automated report |
| Unnecessary equipment rentals | $50,000-100,000/quarter | Under $10,000/quarter |
| Idle equipment detection time | 3 months | 1 week |

-> [Set up this workflow on Lotics](https://lotics.ai)

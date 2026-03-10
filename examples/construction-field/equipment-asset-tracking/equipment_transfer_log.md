# Equipment Transfer Log

## The problem

Equipment moves between job sites constantly -- a skid steer on Site A today may be needed on Site B tomorrow. Transfers are coordinated by phone and text, and the asset register is updated hours or days later. Result: dispatchers send equipment to the wrong site 2-3 times per month (costing $800-1,500 per wasted transport), and project managers cannot accurately report what equipment is on their site for insurance and billing purposes.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Equipment | name, type, serial_number, current_site (link), status | Asset registry with current location |
| Sites | site_name, address, project_manager (link) | Job site directory |
| Transfer Requests | equipment (link), from_site (link), to_site (link), requested_by, status, transfer_date, received_by | Equipment movement records |

## Example prompts

- "When someone presses the 'Approve Transfer' button on a transfer request, update the equipment's current site, notify both the sending and receiving site PMs, and log the transfer with a timestamp."
- "On transfer approval, move the equipment record to the new site, email both project managers, and add a comment to the equipment record with the transfer details."

## Workflow

**Trigger:** Button pressed on Transfer Requests table ("Approve Transfer")

```json
[
  {
    "id": "get_transfer",
    "type": "tool_call",
    "description": "Fetch the full transfer request record",
    "tool_name": "get_record",
    "input": {
      "table_name": "Transfer Requests",
      "record_id": "{{trigger.button_pressed.record_id}}"
    }
  },
  {
    "id": "get_equipment",
    "type": "tool_call",
    "description": "Fetch the equipment being transferred",
    "tool_name": "get_record",
    "input": {
      "table_name": "Equipment",
      "record_id": "{{trigger.button_pressed.record.equipment}}"
    }
  },
  {
    "id": "get_from_site",
    "type": "tool_call",
    "description": "Fetch the origin site details",
    "tool_name": "get_record",
    "input": {
      "table_name": "Sites",
      "record_id": "{{trigger.button_pressed.record.from_site}}"
    }
  },
  {
    "id": "get_to_site",
    "type": "tool_call",
    "description": "Fetch the destination site details",
    "tool_name": "get_record",
    "input": {
      "table_name": "Sites",
      "record_id": "{{trigger.button_pressed.record.to_site}}"
    }
  },
  {
    "id": "update_transfer_status",
    "type": "tool_call",
    "description": "Mark the transfer request as approved",
    "tool_name": "update_record",
    "input": {
      "table_name": "Transfer Requests",
      "record_id": "{{trigger.button_pressed.record_id}}",
      "data": {
        "status": "Approved",
        "transfer_date": "{{new Date().toISOString().split('T')[0]}}"
      }
    }
  },
  {
    "id": "update_equipment_location",
    "type": "tool_call",
    "description": "Move equipment to the destination site",
    "tool_name": "update_record",
    "input": {
      "table_name": "Equipment",
      "record_id": "{{trigger.button_pressed.record.equipment}}",
      "data": {
        "current_site": "{{trigger.button_pressed.record.to_site}}",
        "status": "In Transit"
      }
    }
  },
  {
    "id": "log_transfer_on_equipment",
    "type": "tool_call",
    "description": "Add a transfer comment to the equipment record",
    "tool_name": "create_record_comments",
    "input": {
      "table_name": "Equipment",
      "record_id": "{{trigger.button_pressed.record.equipment}}",
      "comment": "Transferred from {{get_from_site.output.record.site_name}} to {{get_to_site.output.record.site_name}} on {{new Date().toISOString().split('T')[0]}}. Approved by {{trigger.button_pressed.user_name}}."
    }
  },
  {
    "id": "notify_from_pm",
    "type": "tool_call",
    "description": "Notify the origin site PM about the outgoing transfer",
    "tool_name": "send_notification",
    "input": {
      "message": "Equipment leaving your site: {{get_equipment.output.record.name}} ({{get_equipment.output.record.serial_number}}) is being transferred to {{get_to_site.output.record.site_name}} effective today.",
      "record_id": "{{trigger.button_pressed.record.from_site}}",
      "table_name": "Sites"
    }
  },
  {
    "id": "notify_to_pm",
    "type": "tool_call",
    "description": "Notify the destination site PM about the incoming transfer",
    "tool_name": "send_notification",
    "input": {
      "message": "Equipment arriving: {{get_equipment.output.record.name}} ({{get_equipment.output.record.serial_number}}) is being transferred from {{get_from_site.output.record.site_name}} to your site. Expected today.",
      "record_id": "{{trigger.button_pressed.record.to_site}}",
      "table_name": "Sites"
    }
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| Misdirected equipment transports | 2-3 per month | 0 |
| Asset register accuracy | Updated within 1-3 days | Real-time on approval |
| Time to process a transfer | 30 minutes (calls, texts, manual updates) | 1 click |

-> [Set up this workflow on Lotics](https://lotics.ai)

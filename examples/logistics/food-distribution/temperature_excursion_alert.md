# Temperature Excursion Alert and Hold Protocol

## The problem

Food distributors must maintain cold chain integrity from warehouse to delivery. When a reefer unit malfunctions or a door is left open, temperatures rise above the safe threshold (0-4C for chilled, -18C for frozen). If the excursion goes undetected for more than 30 minutes, the entire load — worth $5,000-50,000 — may need to be condemned. Most teams discover excursions only when the driver arrives and the customer rejects the delivery, turning a recoverable incident into a total loss.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Shipments | shipment_ref, vehicle_id, driver_id, customer_id, product_category, temp_threshold, status | Active delivery shipments |
| Temperature Logs | shipment_id, timestamp, temperature, sensor_id | IoT temperature readings |
| Incidents | shipment_id, incident_type, detected_at, temperature_reading, action_taken, status | Quality incidents |

## Example prompts

- "When a temperature reading webhook comes in above the threshold, create an incident, notify the driver and quality team, and put the shipment on hold."
- "Automatically flag cold chain breaches in real time and trigger the hold protocol."

## Workflow

**Trigger:** Webhook received from IoT temperature monitoring system with a reading above threshold.

```json
[
  {
    "id": "get_shipment",
    "type": "tool_call",
    "description": "Fetch the shipment associated with this temperature sensor",
    "tool_name": "query_records",
    "input": {
      "table_id": "shipments",
      "filter": {
        "and": [
          { "field": "vehicle_id", "operator": "equals", "value": "{{trigger.receive_webhook.data.vehicle_id}}" },
          { "field": "status", "operator": "equals", "value": "in_transit" }
        ]
      }
    }
  },
  {
    "id": "log_temperature",
    "type": "tool_call",
    "description": "Log the temperature reading",
    "tool_name": "create_record",
    "input": {
      "table_id": "temperature_logs",
      "data": {
        "shipment_id": "{{get_shipment.output.data[0].id}}",
        "timestamp": "{{trigger.receive_webhook.data.timestamp}}",
        "temperature": "{{trigger.receive_webhook.data.temperature}}",
        "sensor_id": "{{trigger.receive_webhook.data.sensor_id}}"
      }
    }
  },
  {
    "id": "check_excursion",
    "type": "if",
    "description": "Check if the temperature exceeds the shipment's safe threshold",
    "condition": "{{trigger.receive_webhook.data.temperature > get_shipment.output.data[0].temp_threshold}}",
    "then": [
      {
        "id": "create_incident",
        "type": "tool_call",
        "description": "Create a temperature excursion incident record",
        "tool_name": "create_record",
        "input": {
          "table_id": "incidents",
          "data": {
            "shipment_id": "{{get_shipment.output.data[0].id}}",
            "incident_type": "temperature_excursion",
            "detected_at": "{{trigger.receive_webhook.data.timestamp}}",
            "temperature_reading": "{{trigger.receive_webhook.data.temperature}}",
            "status": "open"
          }
        }
      },
      {
        "id": "hold_shipment",
        "type": "tool_call",
        "description": "Put the shipment on quality hold",
        "tool_name": "update_record",
        "input": {
          "table_id": "shipments",
          "record_id": "{{get_shipment.output.data[0].id}}",
          "data": {
            "status": "quality_hold"
          }
        }
      },
      {
        "id": "lock_shipment",
        "type": "tool_call",
        "description": "Lock the shipment record to prevent status changes until quality review",
        "tool_name": "lock_record",
        "input": {
          "table_id": "shipments",
          "record_id": "{{get_shipment.output.data[0].id}}",
          "reason": "Temperature excursion detected — quality hold pending review"
        }
      },
      {
        "id": "notify_quality",
        "type": "tool_call",
        "description": "Alert the quality team about the cold chain breach",
        "tool_name": "send_notification",
        "input": {
          "message": "COLD CHAIN BREACH: Shipment {{get_shipment.output.data[0].shipment_ref}} recorded {{trigger.receive_webhook.data.temperature}}C (threshold: {{get_shipment.output.data[0].temp_threshold}}C). Product: {{get_shipment.output.data[0].product_category}}. Shipment placed on quality hold. Immediate review required.",
          "record_id": "{{create_incident.output.data.id}}"
        }
      }
    ],
    "else": []
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| Excursion detection time | 1-4 hours | Under 2 minutes |
| Product condemned from undetected excursions | $15,000-30,000/month | Under $3,000/month |
| Customer rejections from cold chain issues | 5-8/month | 1-2/month |

→ [Set up this workflow on Lotics](https://lotics.ai)

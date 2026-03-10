# Shipment Milestone Notification

## The problem

Freight forwarders handle 200+ active shipments across ocean, air, and land. When a shipment hits a milestone — vessel departure, customs clearance, arrival at destination port — the customer expects an update within the hour. Most teams rely on ops staff manually emailing updates after checking carrier portals, leading to 4-6 hour delays and a flood of "where's my cargo?" calls that eat 15+ hours per week.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Shipments | shipment_ref, origin, destination, mode, status, eta, customer_id | Master shipment record |
| Milestones | shipment_id, milestone_type, timestamp, location, remarks | Individual tracking events |
| Customers | name, email, notification_pref | Customer contact info |

## Example prompts

- "When a shipment milestone is logged, email the customer with the updated status and ETA."
- "Every time someone adds a milestone to a shipment, send a notification to the customer and update the shipment status."

## Workflow

**Trigger:** A new milestone record is created, indicating a shipment event has occurred.

```json
[
  {
    "id": "get_milestone",
    "type": "tool_call",
    "description": "Read the newly created milestone record",
    "tool_name": "get_record",
    "input": {
      "table_id": "milestones",
      "record_id": "{{trigger.record_updated.record_id}}"
    }
  },
  {
    "id": "get_shipment",
    "type": "tool_call",
    "description": "Fetch the parent shipment to get customer and ETA details",
    "tool_name": "get_record",
    "input": {
      "table_id": "shipments",
      "record_id": "{{get_milestone.output.data.shipment_id}}"
    }
  },
  {
    "id": "update_shipment_status",
    "type": "tool_call",
    "description": "Update the shipment status to reflect the latest milestone",
    "tool_name": "update_record",
    "input": {
      "table_id": "shipments",
      "record_id": "{{get_shipment.output.data.id}}",
      "data": {
        "status": "{{get_milestone.output.data.milestone_type}}",
        "last_milestone_at": "{{get_milestone.output.data.timestamp}}"
      }
    }
  },
  {
    "id": "get_customer",
    "type": "tool_call",
    "description": "Fetch customer contact details for the notification",
    "tool_name": "get_record",
    "input": {
      "table_id": "customers",
      "record_id": "{{get_shipment.output.data.customer_id}}"
    }
  },
  {
    "id": "compose_email",
    "type": "tool_call",
    "description": "Generate a professional milestone update email using LLM",
    "tool_name": "llm_generate_text",
    "input": {
      "prompt": "Write a brief, professional freight shipment update email. Shipment ref: {{get_shipment.output.data.shipment_ref}}. Milestone: {{get_milestone.output.data.milestone_type}} at {{get_milestone.output.data.location}}. ETA: {{get_shipment.output.data.eta}}. Customer name: {{get_customer.output.data.name}}. Keep it under 100 words."
    }
  },
  {
    "id": "send_update",
    "type": "tool_call",
    "description": "Email the milestone update to the customer",
    "tool_name": "gmail_send_email",
    "input": {
      "to": "{{get_customer.output.data.email}}",
      "subject": "Shipment {{get_shipment.output.data.shipment_ref}} — {{get_milestone.output.data.milestone_type}}",
      "body": "{{compose_email.output.text}}"
    }
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| Average customer notification delay | 4-6 hours | Under 5 minutes |
| Weekly "where's my cargo?" calls | 60+ | ~10 |
| Ops hours spent on manual updates | 15 hrs/week | 1 hr/week |

→ [Set up this workflow on Lotics](https://lotics.ai)

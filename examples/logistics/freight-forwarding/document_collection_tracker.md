# Document Collection Tracker

## The problem

Each freight shipment requires 8-15 documents — bill of lading, commercial invoice, packing list, certificate of origin, phytosanitary certificate, and more. Ops coordinators spend hours chasing shippers and agents for missing docs via email and WhatsApp. When documents arrive late, shipments miss cut-off dates, costing $500-2,000 per rollover in storage and rebooking fees.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Shipments | shipment_ref, shipper_id, consignee_id, cut_off_date, doc_status | Master shipment record |
| Required Documents | shipment_id, doc_type, is_received, received_at, file_id | Checklist of required docs per shipment |
| Contacts | name, email, role, company | Shipper/agent contact info |

## Example prompts

- "Every morning at 8 AM, check for shipments with a cut-off in the next 3 days that still have missing documents, and email the shipper a reminder listing what's outstanding."
- "Send a daily reminder for incomplete document sets on upcoming shipments."

## Workflow

**Trigger:** Recurring daily schedule at 08:00 UTC to catch documents due before cut-off.

```json
[
  {
    "id": "find_upcoming_shipments",
    "type": "tool_call",
    "description": "Query shipments with cut-off in the next 3 days that are not fully documented",
    "tool_name": "query_records",
    "input": {
      "table_id": "shipments",
      "filter": {
        "and": [
          { "field": "doc_status", "operator": "not_equals", "value": "complete" },
          { "field": "cut_off_date", "operator": "less_than_or_equals", "value": "{{DATE_ADD(NOW(), 3, 'days')}}" },
          { "field": "cut_off_date", "operator": "greater_than", "value": "{{NOW()}}" }
        ]
      }
    }
  },
  {
    "id": "process_each_shipment",
    "type": "foreach",
    "description": "For each shipment with missing docs, find gaps and notify the shipper",
    "items": "{{find_upcoming_shipments.output.data}}",
    "steps": [
      {
        "id": "find_missing_docs",
        "type": "tool_call",
        "description": "Query required documents that have not been received for this shipment",
        "tool_name": "query_records",
        "input": {
          "table_id": "required_documents",
          "filter": {
            "and": [
              { "field": "shipment_id", "operator": "equals", "value": "{{process_each_shipment.item.id}}" },
              { "field": "is_received", "operator": "equals", "value": false }
            ]
          }
        }
      },
      {
        "id": "check_has_missing",
        "type": "if",
        "description": "Only send reminder if there are actually missing documents",
        "condition": "{{find_missing_docs.output.data.length > 0}}",
        "then": [
          {
            "id": "get_shipper",
            "type": "tool_call",
            "description": "Get shipper contact details",
            "tool_name": "get_record",
            "input": {
              "table_id": "contacts",
              "record_id": "{{process_each_shipment.item.shipper_id}}"
            }
          },
          {
            "id": "compose_reminder",
            "type": "tool_call",
            "description": "Generate a clear reminder listing missing documents",
            "tool_name": "llm_generate_text",
            "input": {
              "prompt": "Write a polite but urgent document reminder email. Shipment: {{process_each_shipment.item.shipment_ref}}. Cut-off date: {{process_each_shipment.item.cut_off_date}}. Missing documents: {{JSON.stringify(find_missing_docs.output.data.map(function(d) { return d.doc_type }))}}. Recipient: {{get_shipper.output.data.name}}. Emphasize the cut-off deadline. Keep it professional and under 120 words."
            }
          },
          {
            "id": "send_reminder",
            "type": "tool_call",
            "description": "Email the document reminder to the shipper",
            "tool_name": "gmail_send_email",
            "input": {
              "to": "{{get_shipper.output.data.email}}",
              "subject": "URGENT: Missing documents for {{process_each_shipment.item.shipment_ref}} — cut-off {{process_each_shipment.item.cut_off_date}}",
              "body": "{{compose_reminder.output.text}}"
            }
          }
        ],
        "else": []
      }
    ]
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| Shipments missing cut-off due to late docs | 8-12/month | 1-2/month |
| Hours spent chasing documents | 20 hrs/week | 3 hrs/week |
| Rollover fees from late documentation | $6,000-15,000/month | Under $2,000/month |

→ [Set up this workflow on Lotics](https://lotics.ai)

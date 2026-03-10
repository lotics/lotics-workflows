# Inbound Shipment Receiving and Discrepancy Check

## The problem

When inbound shipments arrive at the warehouse, receiving staff count items and check against the ASN (Advanced Shipping Notice). Discrepancies — short shipments, damaged goods, wrong SKUs — are scribbled on paper and often not reported to procurement or the supplier until days later. This delays claims, messes up inventory counts, and causes downstream pick errors. A warehouse handling 30+ inbound shipments per day loses 5-10 hours weekly on manual discrepancy follow-up.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Inbound Shipments | asn_number, supplier_id, expected_arrival, status | ASN tracking |
| ASN Lines | shipment_id, sku, expected_qty, received_qty, condition, discrepancy_type | Line-level receiving data |
| Suppliers | name, email | Supplier contacts |
| Discrepancy Reports | shipment_id, total_discrepancies, report_details, created_at | Discrepancy documentation |

## Example prompts

- "When a receiving form is submitted, compare received quantities to the ASN, flag any discrepancies, and email the supplier about shortages or damages."
- "Automatically check inbound receiving against the ASN and create a discrepancy report if anything is off."

## Workflow

**Trigger:** Form submitted by warehouse receiving staff with shipment ID and received quantities.

```json
[
  {
    "id": "get_shipment",
    "type": "tool_call",
    "description": "Fetch the inbound shipment record",
    "tool_name": "get_record",
    "input": {
      "table_id": "inbound_shipments",
      "record_id": "{{trigger.form_submitted.data.shipment_id}}"
    }
  },
  {
    "id": "get_asn_lines",
    "type": "tool_call",
    "description": "Fetch all ASN line items with expected quantities",
    "tool_name": "query_records",
    "input": {
      "table_id": "asn_lines",
      "filter": {
        "field": "shipment_id",
        "operator": "equals",
        "value": "{{trigger.form_submitted.data.shipment_id}}"
      }
    }
  },
  {
    "id": "analyze_discrepancies",
    "type": "tool_call",
    "description": "Compare received vs expected and identify discrepancies",
    "tool_name": "llm_generate_text",
    "input": {
      "prompt": "Compare these ASN lines (expected) against received data and identify discrepancies. ASN lines: {{JSON.stringify(get_asn_lines.output.data)}}. Received data from form: {{JSON.stringify(trigger.form_submitted.data.line_items)}}. Return JSON: { \"has_discrepancies\": true/false, \"total_discrepancies\": number, \"details\": [{ \"sku\": string, \"expected\": number, \"received\": number, \"type\": \"shortage\"|\"overage\"|\"damage\"|\"wrong_sku\" }] }. Only return JSON."
    }
  },
  {
    "id": "update_shipment_received",
    "type": "tool_call",
    "description": "Mark the shipment as received",
    "tool_name": "update_record",
    "input": {
      "table_id": "inbound_shipments",
      "record_id": "{{get_shipment.output.data.id}}",
      "data": {
        "status": "received"
      }
    }
  },
  {
    "id": "check_discrepancies",
    "type": "if",
    "description": "If discrepancies found, create a report and notify supplier",
    "condition": "{{JSON.parse(analyze_discrepancies.output.text).has_discrepancies === true}}",
    "then": [
      {
        "id": "create_discrepancy_report",
        "type": "tool_call",
        "description": "Create a formal discrepancy report record",
        "tool_name": "create_record",
        "input": {
          "table_id": "discrepancy_reports",
          "data": {
            "shipment_id": "{{get_shipment.output.data.id}}",
            "total_discrepancies": "{{JSON.parse(analyze_discrepancies.output.text).total_discrepancies}}",
            "report_details": "{{analyze_discrepancies.output.text}}",
            "created_at": "{{NOW()}}"
          }
        }
      },
      {
        "id": "get_supplier",
        "type": "tool_call",
        "description": "Fetch supplier contact for notification",
        "tool_name": "get_record",
        "input": {
          "table_id": "suppliers",
          "record_id": "{{get_shipment.output.data.supplier_id}}"
        }
      },
      {
        "id": "email_supplier",
        "type": "tool_call",
        "description": "Email the supplier about the discrepancies",
        "tool_name": "gmail_send_email",
        "input": {
          "to": "{{get_supplier.output.data.email}}",
          "subject": "Receiving Discrepancy — ASN {{get_shipment.output.data.asn_number}}",
          "body": "Dear {{get_supplier.output.data.name}},\n\nWe received shipment {{get_shipment.output.data.asn_number}} today and found {{JSON.parse(analyze_discrepancies.output.text).total_discrepancies}} discrepancy(ies).\n\nDetails:\n{{analyze_discrepancies.output.text}}\n\nPlease investigate and advise on resolution (credit, replacement, or adjustment).\n\nRegards"
        }
      }
    ],
    "else": [
      {
        "id": "notify_all_good",
        "type": "tool_call",
        "description": "Notify warehouse manager that receiving is clean",
        "tool_name": "send_notification",
        "input": {
          "message": "ASN {{get_shipment.output.data.asn_number}} received in full — no discrepancies.",
          "record_id": "{{get_shipment.output.data.id}}"
        }
      }
    ]
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| Discrepancy reporting delay | 2-5 days | Same day |
| Hours on manual discrepancy follow-up | 5-10 hrs/week | 1 hr/week |
| Inventory count accuracy | 94% | 99%+ |

→ [Set up this workflow on Lotics](https://lotics.ai)

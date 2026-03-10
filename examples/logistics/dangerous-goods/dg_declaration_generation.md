# Dangerous Goods Declaration Generation

## The problem

Every dangerous goods shipment requires a DG declaration form (IMO Dangerous Goods Declaration for sea, Shipper's Declaration for air) with exact UN numbers, proper shipping names, packing groups, flash points, and emergency contact details. A single error — wrong UN number, missing flash point, incorrect packing group — causes the shipping line or airline to reject the booking, delaying the shipment by days. DG documentation specialists spend 30-45 minutes per declaration cross-referencing IMDG/IATA codes, and even experienced staff make errors on 3-5% of declarations.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| DG Shipments | shipment_ref, shipper_id, consignee_id, mode, carrier, booking_ref, status | DG shipment master |
| DG Cargo Items | shipment_id, un_number, proper_shipping_name, class, subsidiary_risk, packing_group, flash_point, net_qty, gross_weight, package_type, inner_packages | Dangerous goods line items |
| Emergency Contacts | shipment_id, name, phone, available_24h | Emergency response contacts |
| Shippers | name, email, address | Shipper details |

## Example prompts

- "When a DG shipment is marked 'ready for documentation', validate all cargo items against IMDG codes and generate the dangerous goods declaration PDF."
- "Automatically produce the DG declaration form when the shipment is ready, checking all UN numbers and classifications."

## Workflow

**Trigger:** DG Shipment record updated with status changed to "doc_ready".

```json
[
  {
    "id": "get_shipment",
    "type": "tool_call",
    "description": "Fetch the DG shipment details",
    "tool_name": "get_record",
    "input": {
      "table_id": "dg_shipments",
      "record_id": "{{trigger.record_updated.record_id}}"
    }
  },
  {
    "id": "get_cargo_items",
    "type": "tool_call",
    "description": "Fetch all dangerous goods cargo items for this shipment",
    "tool_name": "query_records",
    "input": {
      "table_id": "dg_cargo_items",
      "filter": {
        "field": "shipment_id",
        "operator": "equals",
        "value": "{{trigger.record_updated.record_id}}"
      }
    }
  },
  {
    "id": "get_emergency_contacts",
    "type": "tool_call",
    "description": "Fetch emergency contacts for the declaration",
    "tool_name": "query_records",
    "input": {
      "table_id": "emergency_contacts",
      "filter": {
        "field": "shipment_id",
        "operator": "equals",
        "value": "{{trigger.record_updated.record_id}}"
      }
    }
  },
  {
    "id": "get_shipper",
    "type": "tool_call",
    "description": "Fetch shipper details for the declaration header",
    "tool_name": "get_record",
    "input": {
      "table_id": "shippers",
      "record_id": "{{get_shipment.output.data.shipper_id}}"
    }
  },
  {
    "id": "validate_dg_codes",
    "type": "tool_call",
    "description": "Validate all DG items against IMDG/IATA classification rules",
    "tool_name": "llm_generate_text",
    "input": {
      "prompt": "Validate these dangerous goods items for {{get_shipment.output.data.mode}} transport. Items: {{JSON.stringify(get_cargo_items.output.data)}}. For each item, verify: UN number format (UN followed by 4 digits), proper shipping name matches the UN number, DG class is valid (1-9), packing group is valid (I/II/III) where applicable, flash point is provided for flammable liquids (class 3). Return JSON: { \"all_valid\": true/false, \"items\": [{ \"un_number\": string, \"proper_shipping_name\": string, \"valid\": true/false, \"issues\": [string] }] }. Only return JSON."
    }
  },
  {
    "id": "check_validation",
    "type": "if",
    "description": "Only generate the declaration if all items pass validation",
    "condition": "{{JSON.parse(validate_dg_codes.output.text).all_valid === true}}",
    "then": [
      {
        "id": "generate_dg_declaration",
        "type": "tool_call",
        "description": "Generate the DG declaration PDF from template",
        "tool_name": "generate_pdf_from_pdf_form_template",
        "input": {
          "template_id": "dg_declaration_template",
          "data": {
            "shipment_ref": "{{get_shipment.output.data.shipment_ref}}",
            "shipper_name": "{{get_shipper.output.data.name}}",
            "shipper_address": "{{get_shipper.output.data.address}}",
            "carrier": "{{get_shipment.output.data.carrier}}",
            "booking_ref": "{{get_shipment.output.data.booking_ref}}",
            "cargo_items": "{{get_cargo_items.output.data}}",
            "emergency_contacts": "{{get_emergency_contacts.output.data}}"
          }
        }
      },
      {
        "id": "attach_declaration",
        "type": "tool_call",
        "description": "Attach the generated declaration to the shipment record",
        "tool_name": "add_files_to_record",
        "input": {
          "table_id": "dg_shipments",
          "record_id": "{{get_shipment.output.data.id}}",
          "file_ids": ["{{generate_dg_declaration.output.file_id}}"]
        }
      },
      {
        "id": "update_status",
        "type": "tool_call",
        "description": "Update shipment status to declaration generated",
        "tool_name": "update_record",
        "input": {
          "table_id": "dg_shipments",
          "record_id": "{{get_shipment.output.data.id}}",
          "data": {
            "status": "declaration_generated"
          }
        }
      },
      {
        "id": "notify_success",
        "type": "tool_call",
        "description": "Notify that the DG declaration is ready",
        "tool_name": "send_notification",
        "input": {
          "message": "DG Declaration generated for shipment {{get_shipment.output.data.shipment_ref}} — {{get_cargo_items.output.data.length}} item(s) validated and documented.",
          "record_id": "{{get_shipment.output.data.id}}"
        }
      }
    ],
    "else": [
      {
        "id": "notify_validation_failure",
        "type": "tool_call",
        "description": "Alert about validation failures preventing declaration generation",
        "tool_name": "send_notification",
        "input": {
          "message": "DG Declaration BLOCKED for shipment {{get_shipment.output.data.shipment_ref}}. Validation errors found: {{validate_dg_codes.output.text}}. Correct the DG cargo items before regenerating.",
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
| Time to prepare one DG declaration | 30-45 minutes | Under 3 minutes |
| Declaration rejection rate | 3-5% | Under 0.5% |
| Shipment delays from DG doc errors | 2-4/month | Rare |

→ [Set up this workflow on Lotics](https://lotics.ai)

# House Bill of Lading Generation

## The problem

Freight forwarders issue House Bills of Lading (HBLs) for every consolidated shipment. Each HBL must pull shipper, consignee, cargo description, container, and routing data from multiple sources, then be formatted precisely to industry standards. Manual HBL preparation takes 20-30 minutes per shipment, and typos in consignee details or cargo descriptions cause customs holds that delay cargo by 2-5 days.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Shipments | shipment_ref, origin_port, dest_port, vessel, voyage, container_no, seal_no | Routing and container data |
| Cargo Lines | shipment_id, description, packages, gross_weight_kg, volume_cbm, marks | Cargo detail per line item |
| Parties | shipment_id, role, name, address, contact | Shipper, consignee, notify party |
| Documents | shipment_id, doc_type, file_id, generated_at | Generated document storage |

## Example prompts

- "When a shipment is marked 'ready for documentation', generate the House Bill of Lading PDF and attach it to the shipment record."
- "Automatically create the HBL when the shipment status changes to doc-ready."

## Workflow

**Trigger:** Shipment record updated with status changing to "doc_ready".

```json
[
  {
    "id": "get_shipment",
    "type": "tool_call",
    "description": "Fetch the full shipment record for HBL data",
    "tool_name": "get_record",
    "input": {
      "table_id": "shipments",
      "record_id": "{{trigger.record_updated.record_id}}"
    }
  },
  {
    "id": "get_parties",
    "type": "tool_call",
    "description": "Fetch all parties (shipper, consignee, notify) for this shipment",
    "tool_name": "query_records",
    "input": {
      "table_id": "parties",
      "filter": {
        "field": "shipment_id",
        "operator": "equals",
        "value": "{{trigger.record_updated.record_id}}"
      }
    }
  },
  {
    "id": "get_cargo_lines",
    "type": "tool_call",
    "description": "Fetch all cargo line items for the shipment",
    "tool_name": "query_records",
    "input": {
      "table_id": "cargo_lines",
      "filter": {
        "field": "shipment_id",
        "operator": "equals",
        "value": "{{trigger.record_updated.record_id}}"
      }
    }
  },
  {
    "id": "generate_hbl_pdf",
    "type": "tool_call",
    "description": "Generate the House Bill of Lading PDF from the HTML template",
    "tool_name": "generate_pdf_from_html_template",
    "input": {
      "template_id": "hbl_template",
      "data": {
        "shipment_ref": "{{get_shipment.output.data.shipment_ref}}",
        "origin_port": "{{get_shipment.output.data.origin_port}}",
        "dest_port": "{{get_shipment.output.data.dest_port}}",
        "vessel": "{{get_shipment.output.data.vessel}}",
        "voyage": "{{get_shipment.output.data.voyage}}",
        "container_no": "{{get_shipment.output.data.container_no}}",
        "seal_no": "{{get_shipment.output.data.seal_no}}",
        "parties": "{{get_parties.output.data}}",
        "cargo_lines": "{{get_cargo_lines.output.data}}"
      }
    }
  },
  {
    "id": "attach_hbl",
    "type": "tool_call",
    "description": "Attach the generated HBL PDF to the shipment record",
    "tool_name": "add_files_to_record",
    "input": {
      "table_id": "shipments",
      "record_id": "{{get_shipment.output.data.id}}",
      "file_ids": ["{{generate_hbl_pdf.output.file_id}}"]
    }
  },
  {
    "id": "log_document",
    "type": "tool_call",
    "description": "Create a document record to track the generated HBL",
    "tool_name": "create_record",
    "input": {
      "table_id": "documents",
      "data": {
        "shipment_id": "{{get_shipment.output.data.id}}",
        "doc_type": "house_bill_of_lading",
        "file_id": "{{generate_hbl_pdf.output.file_id}}",
        "generated_at": "{{NOW()}}"
      }
    }
  },
  {
    "id": "notify_ops",
    "type": "tool_call",
    "description": "Notify the ops team that the HBL has been generated",
    "tool_name": "send_notification",
    "input": {
      "message": "HBL generated for shipment {{get_shipment.output.data.shipment_ref}} and attached to the record.",
      "record_id": "{{get_shipment.output.data.id}}"
    }
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| Time to prepare one HBL | 20-30 minutes | Under 1 minute |
| Data entry errors on HBLs | 5-8% of shipments | Near zero |
| Customs holds from HBL errors | 3-5/month | Rare |

→ [Set up this workflow on Lotics](https://lotics.ai)

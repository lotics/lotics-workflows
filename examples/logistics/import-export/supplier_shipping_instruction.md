# Supplier Shipping Instruction Distribution

## The problem

When a purchase order is confirmed, the import team must send detailed shipping instructions to the overseas supplier — including consignee details, shipping marks, labeling requirements, and booking references. These instructions are often cobbled together manually from 3-4 different systems, and a wrong shipping mark or missing booking reference causes misdirected cargo or missed vessel bookings. Teams handling 50+ POs per week lose 10+ hours just preparing and sending these instructions.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Purchase Orders | po_number, supplier_id, ship_to, incoterm, booking_ref, ship_date, status | PO master |
| PO Line Items | po_id, product_name, qty, unit, shipping_marks, label_instructions | Items with packing specs |
| Suppliers | name, email, country, language | Supplier contact info |

## Example prompts

- "When a PO is approved, generate shipping instructions and email them to the supplier with all line item details."
- "Automatically send the supplier a shipping instruction email when the purchase order status changes to confirmed."

## Workflow

**Trigger:** Purchase order record updated with status changing to "confirmed".

```json
[
  {
    "id": "get_po",
    "type": "tool_call",
    "description": "Fetch the confirmed purchase order",
    "tool_name": "get_record",
    "input": {
      "table_id": "purchase_orders",
      "record_id": "{{trigger.record_updated.record_id}}"
    }
  },
  {
    "id": "get_line_items",
    "type": "tool_call",
    "description": "Fetch all line items for packing and labeling details",
    "tool_name": "query_records",
    "input": {
      "table_id": "po_line_items",
      "filter": {
        "field": "po_id",
        "operator": "equals",
        "value": "{{trigger.record_updated.record_id}}"
      }
    }
  },
  {
    "id": "get_supplier",
    "type": "tool_call",
    "description": "Fetch supplier contact details",
    "tool_name": "get_record",
    "input": {
      "table_id": "suppliers",
      "record_id": "{{get_po.output.data.supplier_id}}"
    }
  },
  {
    "id": "generate_si",
    "type": "tool_call",
    "description": "Generate the shipping instruction document as PDF",
    "tool_name": "generate_pdf_from_html_template",
    "input": {
      "template_id": "shipping_instruction_template",
      "data": {
        "po_number": "{{get_po.output.data.po_number}}",
        "booking_ref": "{{get_po.output.data.booking_ref}}",
        "ship_to": "{{get_po.output.data.ship_to}}",
        "ship_date": "{{get_po.output.data.ship_date}}",
        "incoterm": "{{get_po.output.data.incoterm}}",
        "line_items": "{{get_line_items.output.data}}"
      }
    }
  },
  {
    "id": "compose_email",
    "type": "tool_call",
    "description": "Draft the shipping instruction email body",
    "tool_name": "llm_generate_text",
    "input": {
      "prompt": "Write a professional shipping instruction email to supplier {{get_supplier.output.data.name}}. PO: {{get_po.output.data.po_number}}. Booking ref: {{get_po.output.data.booking_ref}}. Required ship date: {{get_po.output.data.ship_date}}. Incoterm: {{get_po.output.data.incoterm}}. Mention that the full shipping instruction PDF is attached. Ask them to confirm receipt and ship date. Keep it under 100 words."
    }
  },
  {
    "id": "send_si_email",
    "type": "tool_call",
    "description": "Email the shipping instructions with PDF attachment to the supplier",
    "tool_name": "gmail_send_email",
    "input": {
      "to": "{{get_supplier.output.data.email}}",
      "subject": "Shipping Instructions — PO {{get_po.output.data.po_number}} / Booking {{get_po.output.data.booking_ref}}",
      "body": "{{compose_email.output.text}}",
      "attachments": ["{{generate_si.output.file_id}}"]
    }
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| Time to prepare shipping instructions | 15-20 min per PO | Under 1 minute |
| Errors in shipping marks/booking refs | 6-8% of POs | Near zero |
| Hours spent per week on SI prep | 10+ hours | Under 1 hour |

→ [Set up this workflow on Lotics](https://lotics.ai)

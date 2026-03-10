# QC Inspection Report Processing

## The problem

Before goods ship from the factory, sourcing agents arrange third-party QC inspections (pre-shipment, during-production, or container loading). Inspectors submit reports with defect photos, AQL sampling results, and pass/fail verdicts. The sourcing agent must review the report, decide whether to approve shipment, and communicate the result to both the buyer and the factory — often across time zones. When reports come in after business hours, the decision waits until the next morning, delaying container loading and potentially missing the vessel booking.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Orders | order_ref, buyer_id, supplier_id, product, qty, ship_date, status | Purchase order master |
| QC Inspections | order_id, inspection_type, inspector_name, inspection_date, aql_level, sample_size, defects_found, critical_defects, major_defects, minor_defects, verdict, status | Inspection records |
| Buyers | name, email | Buyer contacts |
| Suppliers | name, email, factory_contact_email | Supplier/factory contacts |

## Example prompts

- "When a QC inspection form is submitted, analyze the results against AQL limits, update the order, and notify the buyer and factory of the verdict."
- "Automatically process incoming QC inspection reports and route the approval decision."

## Workflow

**Trigger:** Form submitted by QC inspector with inspection data and photos.

```json
[
  {
    "id": "get_order",
    "type": "tool_call",
    "description": "Fetch the order being inspected",
    "tool_name": "get_record",
    "input": {
      "table_id": "orders",
      "record_id": "{{trigger.form_submitted.data.order_id}}"
    }
  },
  {
    "id": "create_inspection",
    "type": "tool_call",
    "description": "Create the QC inspection record",
    "tool_name": "create_record",
    "input": {
      "table_id": "qc_inspections",
      "data": {
        "order_id": "{{trigger.form_submitted.data.order_id}}",
        "inspection_type": "{{trigger.form_submitted.data.inspection_type}}",
        "inspector_name": "{{trigger.form_submitted.data.inspector_name}}",
        "inspection_date": "{{NOW()}}",
        "aql_level": "{{trigger.form_submitted.data.aql_level}}",
        "sample_size": "{{trigger.form_submitted.data.sample_size}}",
        "defects_found": "{{trigger.form_submitted.data.total_defects}}",
        "critical_defects": "{{trigger.form_submitted.data.critical_defects}}",
        "major_defects": "{{trigger.form_submitted.data.major_defects}}",
        "minor_defects": "{{trigger.form_submitted.data.minor_defects}}",
        "status": "pending_review"
      }
    }
  },
  {
    "id": "attach_photos",
    "type": "tool_call",
    "description": "Attach defect photos to the inspection record",
    "tool_name": "add_files_to_record",
    "input": {
      "table_id": "qc_inspections",
      "record_id": "{{create_inspection.output.data.id}}",
      "file_ids": "{{trigger.form_submitted.data.photo_file_ids}}"
    }
  },
  {
    "id": "evaluate_aql",
    "type": "tool_call",
    "description": "Evaluate inspection results against AQL acceptance criteria",
    "tool_name": "llm_generate_text",
    "input": {
      "prompt": "Evaluate this QC inspection against AQL (Acceptable Quality Level) standards. AQL level: {{trigger.form_submitted.data.aql_level}}. Sample size: {{trigger.form_submitted.data.sample_size}}. Critical defects: {{trigger.form_submitted.data.critical_defects}} (AQL 0 — zero tolerance). Major defects: {{trigger.form_submitted.data.major_defects}}. Minor defects: {{trigger.form_submitted.data.minor_defects}}. Determine: verdict (pass/fail/conditional), reasoning, and recommended action. Return JSON: { \"verdict\": \"pass\" | \"fail\" | \"conditional\", \"reasoning\": string, \"recommended_action\": string }. Only return JSON."
    }
  },
  {
    "id": "update_inspection_verdict",
    "type": "tool_call",
    "description": "Update the inspection record with the verdict",
    "tool_name": "update_record",
    "input": {
      "table_id": "qc_inspections",
      "record_id": "{{create_inspection.output.data.id}}",
      "data": {
        "verdict": "{{JSON.parse(evaluate_aql.output.text).verdict}}",
        "status": "reviewed"
      }
    }
  },
  {
    "id": "get_buyer",
    "type": "tool_call",
    "description": "Fetch buyer contact details",
    "tool_name": "get_record",
    "input": {
      "table_id": "buyers",
      "record_id": "{{get_order.output.data.buyer_id}}"
    }
  },
  {
    "id": "get_supplier",
    "type": "tool_call",
    "description": "Fetch supplier/factory contact details",
    "tool_name": "get_record",
    "input": {
      "table_id": "suppliers",
      "record_id": "{{get_order.output.data.supplier_id}}"
    }
  },
  {
    "id": "notify_buyer",
    "type": "tool_call",
    "description": "Email the QC result to the buyer",
    "tool_name": "gmail_send_email",
    "input": {
      "to": "{{get_buyer.output.data.email}}",
      "subject": "QC Inspection Result: {{JSON.parse(evaluate_aql.output.text).verdict}} — Order {{get_order.output.data.order_ref}}",
      "body": "Dear {{get_buyer.output.data.name}},\n\nThe {{trigger.form_submitted.data.inspection_type}} inspection for order {{get_order.output.data.order_ref}} has been completed.\n\nVerdict: {{JSON.parse(evaluate_aql.output.text).verdict}}\nSample size: {{trigger.form_submitted.data.sample_size}}\nDefects: {{trigger.form_submitted.data.critical_defects}} critical, {{trigger.form_submitted.data.major_defects}} major, {{trigger.form_submitted.data.minor_defects}} minor\n\nReasoning: {{JSON.parse(evaluate_aql.output.text).reasoning}}\nRecommended action: {{JSON.parse(evaluate_aql.output.text).recommended_action}}\n\nPlease confirm how you would like to proceed.\n\nRegards"
    }
  },
  {
    "id": "notify_factory",
    "type": "tool_call",
    "description": "Email the QC result to the factory",
    "tool_name": "gmail_send_email",
    "input": {
      "to": "{{get_supplier.output.data.factory_contact_email}}",
      "subject": "QC Result: {{JSON.parse(evaluate_aql.output.text).verdict}} — Order {{get_order.output.data.order_ref}}",
      "body": "Dear {{get_supplier.output.data.name}},\n\nInspection completed for order {{get_order.output.data.order_ref}}.\n\nResult: {{JSON.parse(evaluate_aql.output.text).verdict}}\nAction required: {{JSON.parse(evaluate_aql.output.text).recommended_action}}\n\nPlease address any defects noted and await further instructions before shipping.\n\nRegards"
    }
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| Time from inspection to buyer notification | 6-24 hours | Under 15 minutes |
| Shipments delayed waiting for QC decision | 20-30% | Under 5% |
| AQL evaluation errors | Occasional | Consistent application |

→ [Set up this workflow on Lotics](https://lotics.ai)

# Incoming Material Inspection

## The problem

A medical device manufacturer is required to inspect every incoming lot of raw materials against specifications before releasing them to production. Inspectors fill out paper forms, attach them to the lot folder, and email the QA manager for sign-off. With 50+ lots arriving per week, 10-15% of inspection forms go missing, and lots occasionally reach production without documented approval -- a serious compliance risk under FDA 21 CFR Part 820.

## Data model

| Table | Key fields | Purpose |
|-------|-----------|---------|
| Material Lots | lot_id, component, supplier_id, qty_received, status, po_id | Incoming material lot registry |
| Inspection Records | inspection_id, lot_id, inspector, result, inspected_at, notes | Inspection outcome and details |
| Inspection Criteria | criteria_id, component, test_name, min_value, max_value, unit | Specification limits per component |
| Non-Conformance Reports | ncr_id, lot_id, inspection_id, description, disposition, status | Tracks material that fails inspection |

## Example prompts

- "When a material lot arrives, create an inspection checklist based on the component's specification criteria. After the inspector submits results, auto-pass or auto-fail the lot and create a non-conformance report for any failures."
- "Automate incoming quality inspection -- generate the checklist from specs, evaluate results, and quarantine failed lots with an NCR."

## Workflow

**Trigger:** A material lot record status changes to "Received"

```json
[
  {
    "id": "get_lot",
    "type": "tool_call",
    "description": "Fetch the material lot details",
    "tool_name": "get_record",
    "input": {
      "table_id": "material_lots",
      "record_id": "{{trigger.record_updated.record_id}}"
    }
  },
  {
    "id": "get_criteria",
    "type": "tool_call",
    "description": "Query the inspection criteria for this component",
    "tool_name": "query_records",
    "input": {
      "table_id": "inspection_criteria",
      "filters": {
        "component": "{{get_lot.output.data.component}}"
      }
    }
  },
  {
    "id": "create_inspection",
    "type": "tool_call",
    "description": "Create an inspection record for the lot",
    "tool_name": "create_record",
    "input": {
      "table_id": "inspection_records",
      "data": {
        "lot_id": "{{get_lot.output.data.lot_id}}",
        "result": "Pending",
        "notes": "Auto-generated. {{get_criteria.output.records.length}} tests required."
      }
    }
  },
  {
    "id": "update_lot_inspecting",
    "type": "tool_call",
    "description": "Move the lot to Inspecting status",
    "tool_name": "update_record",
    "input": {
      "table_id": "material_lots",
      "record_id": "{{trigger.record_updated.record_id}}",
      "data": {
        "status": "Inspecting"
      }
    }
  },
  {
    "id": "notify_inspector",
    "type": "tool_call",
    "description": "Notify the quality inspector that a lot is ready for inspection",
    "tool_name": "send_notification",
    "input": {
      "title": "Lot {{get_lot.output.data.lot_id}} ready for inspection",
      "message": "Component: {{get_lot.output.data.component}}, Supplier: {{get_lot.output.data.supplier_id}}, Quantity: {{get_lot.output.data.qty_received}}. {{get_criteria.output.records.length}} inspection tests required."
    }
  },
  {
    "id": "wait_inspection",
    "type": "wait_for_event",
    "description": "Wait for the inspector to submit the inspection results",
    "event": "record_updated",
    "filters": {
      "record_id": "{{create_inspection.output.record_id}}",
      "field": "result",
      "value_in": ["Pass", "Fail"]
    }
  },
  {
    "id": "evaluate_result",
    "type": "switch",
    "description": "Handle pass or fail outcome",
    "expression": "{{wait_inspection.output.next_data.result}}",
    "cases": {
      "Pass": [
        {
          "id": "release_lot",
          "type": "tool_call",
          "description": "Release the lot for production use",
          "tool_name": "update_record",
          "input": {
            "table_id": "material_lots",
            "record_id": "{{trigger.record_updated.record_id}}",
            "data": {
              "status": "Released"
            }
          }
        }
      ],
      "Fail": [
        {
          "id": "quarantine_lot",
          "type": "tool_call",
          "description": "Quarantine the failed lot",
          "tool_name": "update_record",
          "input": {
            "table_id": "material_lots",
            "record_id": "{{trigger.record_updated.record_id}}",
            "data": {
              "status": "Quarantined"
            }
          }
        },
        {
          "id": "create_ncr",
          "type": "tool_call",
          "description": "Create a non-conformance report for the failed lot",
          "tool_name": "create_record",
          "input": {
            "table_id": "non_conformance_reports",
            "data": {
              "lot_id": "{{get_lot.output.data.lot_id}}",
              "inspection_id": "{{create_inspection.output.record_id}}",
              "description": "Lot failed incoming inspection. Inspector notes: {{wait_inspection.output.next_data.notes}}",
              "disposition": "Pending Review",
              "status": "Open"
            }
          }
        },
        {
          "id": "lock_lot",
          "type": "tool_call",
          "description": "Lock the quarantined lot record to prevent unauthorized changes",
          "tool_name": "lock_record",
          "input": {
            "table_id": "material_lots",
            "record_id": "{{trigger.record_updated.record_id}}"
          }
        },
        {
          "id": "notify_qa_manager",
          "type": "tool_call",
          "description": "Alert QA manager about the failed inspection",
          "tool_name": "send_notification",
          "input": {
            "title": "Inspection FAIL: Lot {{get_lot.output.data.lot_id}}",
            "message": "Component {{get_lot.output.data.component}} from {{get_lot.output.data.supplier_id}} failed incoming inspection. NCR created. Lot quarantined and locked."
          }
        }
      ]
    }
  }
]
```

## Results

| Metric | Before | After |
|--------|--------|-------|
| Missing inspection forms | 10-15% | 0% |
| Uninspected lots reaching production | 2-3/month | 0 |
| Inspection cycle time | 1-2 days | 2-4 hours |

-> [Set up this workflow on Lotics](https://lotics.ai)

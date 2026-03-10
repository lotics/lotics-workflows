# Repair Estimate Generation and Line Approval

## The problem

After inspection, damaged containers need repair estimates sent to the shipping line for approval before work can begin. Estimators manually calculate labor and material costs using IICL (Institute of International Container Lessors) repair codes, type up the estimate, and email it. The shipping line takes 1-3 days to respond, and follow-up is manual. Containers sit idle in the yard at $5-10/day storage while waiting for approval, and estimators spend 30+ minutes per estimate on repetitive calculations.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Containers | container_number, owner_line, current_status | Container master |
| Damage Items | inspection_id, damage_type, location_on_container, severity, repair_needed | Damage details |
| Repair Estimates | container_id, total_labor, total_material, total_cost, status, submitted_at, approved_at | Estimate records |
| Shipping Lines | name, email, approval_contact | Line owner contacts |

## Example prompts

- "When an inspector clicks 'Generate Estimate', calculate repair costs from damage items and email the estimate to the shipping line for approval."
- "Automatically build a repair estimate from inspection damage and send it to the container owner."

## Workflow

**Trigger:** Button pressed on a container record to generate repair estimate.

```json
[
  {
    "id": "get_container",
    "type": "tool_call",
    "description": "Fetch the container record",
    "tool_name": "get_record",
    "input": {
      "table_id": "containers",
      "record_id": "{{trigger.button_pressed.record_id}}"
    }
  },
  {
    "id": "get_damage_items",
    "type": "tool_call",
    "description": "Query all damage items that need repair for this container's latest inspection",
    "tool_name": "query_records",
    "input": {
      "table_id": "damage_items",
      "filter": {
        "and": [
          { "field": "container_id", "operator": "equals", "value": "{{trigger.button_pressed.record_id}}" },
          { "field": "repair_needed", "operator": "equals", "value": true }
        ]
      }
    }
  },
  {
    "id": "calculate_estimate",
    "type": "tool_call",
    "description": "Use LLM to calculate IICL-standard repair costs from damage items",
    "tool_name": "llm_generate_text",
    "input": {
      "prompt": "You are a container repair estimator. Calculate repair costs using standard IICL repair codes. Damage items: {{JSON.stringify(get_damage_items.output.data)}}. For each item, provide IICL repair code, labor hours, labor cost at $45/hr, material cost, and line total. Return JSON: { \"line_items\": [{ \"damage_type\": string, \"iicl_code\": string, \"labor_hours\": number, \"labor_cost\": number, \"material_cost\": number, \"line_total\": number }], \"total_labor\": number, \"total_material\": number, \"total_cost\": number }. Only return JSON."
    }
  },
  {
    "id": "create_estimate",
    "type": "tool_call",
    "description": "Create the repair estimate record",
    "tool_name": "create_record",
    "input": {
      "table_id": "repair_estimates",
      "data": {
        "container_id": "{{get_container.output.data.id}}",
        "total_labor": "{{JSON.parse(calculate_estimate.output.text).total_labor}}",
        "total_material": "{{JSON.parse(calculate_estimate.output.text).total_material}}",
        "total_cost": "{{JSON.parse(calculate_estimate.output.text).total_cost}}",
        "status": "pending_approval",
        "submitted_at": "{{NOW()}}"
      }
    }
  },
  {
    "id": "generate_estimate_pdf",
    "type": "tool_call",
    "description": "Generate a formal repair estimate PDF",
    "tool_name": "generate_pdf_from_html_template",
    "input": {
      "template_id": "repair_estimate_template",
      "data": {
        "container_number": "{{get_container.output.data.container_number}}",
        "owner_line": "{{get_container.output.data.owner_line}}",
        "estimate_data": "{{calculate_estimate.output.text}}"
      }
    }
  },
  {
    "id": "get_shipping_line",
    "type": "tool_call",
    "description": "Fetch the shipping line contact for approval",
    "tool_name": "query_records",
    "input": {
      "table_id": "shipping_lines",
      "filter": {
        "field": "name",
        "operator": "equals",
        "value": "{{get_container.output.data.owner_line}}"
      }
    }
  },
  {
    "id": "email_estimate",
    "type": "tool_call",
    "description": "Email the repair estimate to the shipping line for approval",
    "tool_name": "gmail_send_email",
    "input": {
      "to": "{{get_shipping_line.output.data[0].approval_contact}}",
      "subject": "Repair Estimate — Container {{get_container.output.data.container_number}} — ${{JSON.parse(calculate_estimate.output.text).total_cost}}",
      "body": "Please find attached the repair estimate for container {{get_container.output.data.container_number}}.\n\nTotal: ${{JSON.parse(calculate_estimate.output.text).total_cost}} (Labor: ${{JSON.parse(calculate_estimate.output.text).total_labor}}, Material: ${{JSON.parse(calculate_estimate.output.text).total_material}})\n\nPlease reply to approve or request revisions.",
      "attachments": ["{{generate_estimate_pdf.output.file_id}}"]
    }
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| Time to produce a repair estimate | 30-45 minutes | Under 3 minutes |
| Average container idle days awaiting estimate | 2-3 days | Same day |
| Storage cost from estimate delays | $2,000-4,000/month | Under $500/month |

→ [Set up this workflow on Lotics](https://lotics.ai)

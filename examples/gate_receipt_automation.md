# Gate Receipt Automation

Automate gate-in/gate-out documentation at container depots.

## The problem

500 containers/month means 1,000+ paper forms, 15-20 minutes of paperwork per container, 5-10% error rate, 250+ hours/month on documentation.

## Data model

| Table | Key fields | Purpose |
|-------|-----------|---------|
| Containers | container_number (ISO 6346), size, status, seal_number, condition, location | Inventory |
| Gate Transactions | container (link), type (In/Out), timestamp, driver, truck_plate, damage_notes, photos | Every gate event |
| Customers | company_name, contact_email | For auto-sending receipts |

## Example prompts

Say this to the [Lotics](https://lotics.ai) AI assistant:

> Create tables for container depot management: a Containers table with container number, size, status, seal number, condition, and yard location. A Gate Transactions table linked to containers with gate type, timestamp, driver name, truck plate, damage notes, and photos.

> Set up a workflow that triggers when a new gate transaction is created. Update the container status, generate a gate receipt PDF, and email it to the customer.

## Workflow

**Trigger:** `record_updated` on the Gate Transactions table (fires on new records and updates).

**Steps:**

```json
[
  {
    "id": "update_status",
    "type": "tool_call",
    "description": "Update container status based on gate type",
    "tool_name": "update_record",
    "input": {
      "table_id": "{{trigger.record_updated.table_id}}",
      "record_id": "{{trigger.record_updated.next_data.container}}",
      "data": {
        "status": "{{trigger.record_updated.next_data.type === 'Gate In' ? 'In Yard' : 'Gate Out'}}"
      }
    }
  },
  {
    "id": "check_damage",
    "type": "if",
    "description": "Check if damage was reported",
    "condition": "{{trigger.record_updated.next_data.damage_notes !== null && trigger.record_updated.next_data.damage_notes !== ''}}",
    "then": [
      {
        "id": "create_repair",
        "type": "tool_call",
        "description": "Create repair record for damaged container",
        "tool_name": "create_record",
        "input": {
          "table_id": "repairs_table_id",
          "data": {
            "container": "{{trigger.record_updated.next_data.container}}",
            "reported_damage": "{{trigger.record_updated.next_data.damage_notes}}"
          }
        }
      }
    ]
  },
  {
    "id": "generate_receipt",
    "type": "tool_call",
    "description": "Generate gate receipt PDF",
    "tool_name": "generate_pdf_from_html_template",
    "input": {
      "template_id": "gate_receipt_template_id",
      "record_id": "{{trigger.record_updated.record_id}}",
      "table_id": "{{trigger.record_updated.table_id}}"
    }
  },
  {
    "id": "email_receipt",
    "type": "tool_call",
    "description": "Email receipt to customer",
    "tool_name": "gmail_send_email",
    "input": {
      "to": "{{trigger.record_updated.display.container.customer.contact_email}}",
      "subject": "Gate {{trigger.record_updated.next_data.type}} — {{trigger.record_updated.display.container.container_number}}",
      "body": "Attached is your gate receipt.",
      "attachments": ["{{generate_receipt.output.file_url}}"]
    }
  }
]
```

**Expression syntax:** `{{ }}` wraps JavaScript expressions evaluated against the workflow execution context. `trigger.record_updated.*` accesses the trigger payload. `{step_id}.output.*` accesses results from previous steps.

**Step types used:**
- `tool_call` — executes a registered tool (`update_record`, `create_record`, `generate_pdf_from_html_template`, `gmail_send_email`)
- `if` — conditional branching with `condition`, `then`, and optional `else`

Other available step types: `switch` (multi-way branch), `foreach` (loop over arrays), `wait` (delay), `wait_for_event` (pause for external event like payment), `return` (exit early).

## Results

30 team members, ~500 containers/month:

| Metric | Before | After |
|--------|--------|-------|
| Time per gate transaction | 15-20 min | 2-3 min |
| Monthly paperwork hours | 250+ | ~50 |
| Data entry error rate | 5-10% | <1% |

→ [Set up this workflow on Lotics](https://lotics.ai)

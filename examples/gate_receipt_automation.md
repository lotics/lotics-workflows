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

```json
{
  "name": "Gate Receipt Automation",
  "trigger": { "type": "record_created", "table": "Gate Transactions" },
  "steps": [
    {
      "name": "Update container status",
      "type": "update_record",
      "table": "Containers",
      "record": "{{container}}",
      "fields": {
        "status": "{{if type == 'Gate In' then 'In Yard' else 'Gate Out'}}"
      }
    },
    {
      "name": "Check for damage",
      "type": "condition",
      "condition": "damage_notes is not empty",
      "on_true": [
        {
          "type": "create_record",
          "table": "Repairs",
          "fields": { "container": "{{container}}", "reported_damage": "{{damage_notes}}" }
        }
      ]
    },
    {
      "name": "Generate gate receipt",
      "type": "generate_document",
      "template": "gate_receipt",
      "output_field": "receipt_pdf"
    },
    {
      "name": "Email receipt",
      "type": "send_email",
      "to": "{{container.customer.contact_email}}",
      "subject": "Gate {{type}} — {{container.container_number}}",
      "attachments": ["{{receipt_pdf}}"]
    }
  ]
}
```

## Results

30 team members, ~500 containers/month:

| Metric | Before | After |
|--------|--------|-------|
| Time per gate transaction | 15-20 min | 2-3 min |
| Monthly paperwork hours | 250+ | ~50 |
| Data entry error rate | 5-10% | <1% |

→ [Set up this workflow on Lotics](https://lotics.ai)

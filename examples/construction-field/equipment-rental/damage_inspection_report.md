# Damage Inspection Report

## The problem

When rented equipment is returned, yard staff perform a visual inspection and note any damage. This information is scrawled on a paper form, filed in a folder, and only entered into the system later -- if at all. Disputed damage charges are the number one source of customer complaints, and without timestamped documentation at the point of return, the company loses 40% of damage claim disputes, writing off $15,000-25,000 per year.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Equipment Returns | rental_contract (link), equipment (link), return_date, inspected_by, damage_found, damage_description, damage_cost_estimate, photos | Return inspection records |
| Rental Contracts | contract_number, customer (link), equipment (link), status | Rental agreements |
| Customers | company_name, contact_name, email | Customer directory |

## Example prompts

- "When a return inspection form is submitted and damage is found, generate a damage report PDF with photos and inspection notes, email it to the customer, and create a billing record for the repair cost."
- "On equipment return with damage, auto-generate a damage report, notify the customer with documentation, and flag the contract for damage billing."

## Workflow

**Trigger:** When the return inspection form is submitted on the Equipment Returns table

```json
[
  {
    "id": "check_damage",
    "type": "if",
    "description": "Only proceed if damage was found during inspection",
    "condition": "{{trigger.form_submitted.data.damage_found === true}}",
    "then": [
      {
        "id": "get_contract",
        "type": "tool_call",
        "description": "Fetch the rental contract for customer info",
        "tool_name": "get_record",
        "input": {
          "table_name": "Rental Contracts",
          "record_id": "{{trigger.form_submitted.data.rental_contract}}"
        }
      },
      {
        "id": "get_customer",
        "type": "tool_call",
        "description": "Fetch customer details",
        "tool_name": "get_record",
        "input": {
          "table_name": "Customers",
          "record_id": "{{get_contract.output.record.customer}}"
        }
      },
      {
        "id": "get_equipment",
        "type": "tool_call",
        "description": "Fetch equipment details for the report",
        "tool_name": "get_record",
        "input": {
          "table_name": "Equipment",
          "record_id": "{{trigger.form_submitted.data.equipment}}"
        }
      },
      {
        "id": "generate_damage_report",
        "type": "tool_call",
        "description": "Generate a PDF damage inspection report",
        "tool_name": "generate_pdf_from_html_template",
        "input": {
          "template_name": "damage_inspection_report",
          "data": {
            "contract_number": "{{get_contract.output.record.contract_number}}",
            "customer_name": "{{get_customer.output.record.company_name}}",
            "equipment_name": "{{get_equipment.output.record.name}}",
            "serial_number": "{{get_equipment.output.record.serial_number}}",
            "return_date": "{{trigger.form_submitted.data.return_date}}",
            "inspected_by": "{{trigger.form_submitted.data.inspected_by}}",
            "damage_description": "{{trigger.form_submitted.data.damage_description}}",
            "damage_cost_estimate": "{{trigger.form_submitted.data.damage_cost_estimate}}"
          }
        }
      },
      {
        "id": "attach_report_to_return",
        "type": "tool_call",
        "description": "Attach the damage report PDF to the return record",
        "tool_name": "add_files_to_record",
        "input": {
          "table_name": "Equipment Returns",
          "record_id": "{{trigger.form_submitted.record_id}}",
          "files": ["{{generate_damage_report.output.file_url}}"]
        }
      },
      {
        "id": "email_customer_damage",
        "type": "tool_call",
        "description": "Email the customer with the damage report and estimated charges",
        "tool_name": "gmail_send_email",
        "input": {
          "to": "{{get_customer.output.record.email}}",
          "subject": "Equipment Return — Damage Report for Contract {{get_contract.output.record.contract_number}}",
          "body": "Hi {{get_customer.output.record.contact_name}},\n\nDuring the return inspection of {{get_equipment.output.record.name}} (S/N: {{get_equipment.output.record.serial_number}}) on {{trigger.form_submitted.data.return_date}}, damage was noted:\n\n{{trigger.form_submitted.data.damage_description}}\n\nEstimated repair cost: ${{trigger.form_submitted.data.damage_cost_estimate}}\n\nPlease see the attached damage inspection report with full details. If you have questions or wish to dispute any findings, please respond within 5 business days.\n\nThank you.",
          "attachments": ["{{generate_damage_report.output.file_url}}"]
        }
      },
      {
        "id": "update_contract_status",
        "type": "tool_call",
        "description": "Flag the contract for damage billing",
        "tool_name": "update_record",
        "input": {
          "table_name": "Rental Contracts",
          "record_id": "{{trigger.form_submitted.data.rental_contract}}",
          "data": {
            "status": "Damage Billing"
          }
        }
      }
    ],
    "else": [
      {
        "id": "close_clean_return",
        "type": "tool_call",
        "description": "Mark contract as returned with no damage",
        "tool_name": "update_record",
        "input": {
          "table_name": "Rental Contracts",
          "record_id": "{{trigger.form_submitted.data.rental_contract}}",
          "data": {
            "status": "Returned"
          }
        }
      }
    ]
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| Damage claim disputes lost | 40% | Under 5% (timestamped documentation) |
| Annual damage write-offs | $15,000-25,000 | Under $3,000 |
| Time from return to customer notification | 3-5 days | Under 10 minutes |

-> [Set up this workflow on Lotics](https://lotics.ai)

# Service Completion Invoice

## The problem

After an HVAC tech completes a service call, the office must generate an invoice based on labor hours, parts used, and contract terms (T&M vs. flat rate vs. covered under maintenance contract). Billing staff process 20-30 invoices per day, cross-referencing parts prices, labor rates, and contract coverage. Invoices go out 5-7 days after service on average, and 15% contain errors (wrong labor rate, missed parts, incorrect contract discount) that require correction and re-sending.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Service Tickets | customer (link), assigned_tech (link), status, labor_hours, parts_used, completion_notes, issue_type | Completed service records |
| Customers | company_name, contact_name, email, contract_tier, billing_address | Customer billing info |
| Parts | part_name, part_number, unit_cost | Parts catalog for pricing |
| Invoices | customer (link), service_ticket (link), line_items, total_amount, status, sent_date | Generated invoices |

## Example prompts

- "When a service ticket is marked Complete, look up the customer's contract tier, calculate the invoice based on labor hours and parts used, generate a PDF invoice, and email it to the customer the same day."
- "Auto-generate invoices on ticket completion: pull labor rates from the contract tier, price out parts from the catalog, create the invoice record with line items, and send the PDF to the customer."

## Workflow

**Trigger:** When a Service Tickets record is updated

```json
[
  {
    "id": "check_completion",
    "type": "if",
    "description": "Only proceed if ticket just changed to Complete",
    "condition": "{{trigger.record_updated.next_data.status === 'Complete' && trigger.record_updated.prev_data.status !== 'Complete'}}",
    "then": [
      {
        "id": "get_customer",
        "type": "tool_call",
        "description": "Fetch customer details and contract tier for billing rates",
        "tool_name": "get_record",
        "input": {
          "table_name": "Customers",
          "record_id": "{{trigger.record_updated.next_data.customer}}"
        }
      },
      {
        "id": "get_parts_pricing",
        "type": "tool_call",
        "description": "Query the parts catalog for pricing",
        "tool_name": "query_records",
        "input": {
          "table_name": "Parts"
        }
      },
      {
        "id": "calculate_invoice",
        "type": "tool_call",
        "description": "Use LLM to calculate invoice line items based on labor, parts, and contract tier",
        "tool_name": "llm_generate_text",
        "input": {
          "prompt": "Calculate an HVAC service invoice as JSON with the following fields: line_items (array of {description, quantity, unit_price, total}), subtotal, tax (8.25%), total.\n\nLabor: {{trigger.record_updated.next_data.labor_hours}} hours\nContract tier: {{get_customer.output.record.contract_tier}}\nLabor rates: Standard=$125/hr, Premium=$110/hr (10% discount), Enterprise=$100/hr (20% discount)\nParts used: {{trigger.record_updated.next_data.parts_used}}\nParts catalog: {{JSON.stringify(get_parts_pricing.output.records.map(p => ({name: p.part_name, cost: p.unit_cost})))}}\n\nIf contract tier is 'Maintenance Contract', labor is covered — only bill parts.",
          "response_format": "json"
        }
      },
      {
        "id": "create_invoice_record",
        "type": "tool_call",
        "description": "Create the invoice record",
        "tool_name": "create_record",
        "input": {
          "table_name": "Invoices",
          "data": {
            "customer": "{{trigger.record_updated.next_data.customer}}",
            "service_ticket": "{{trigger.record_updated.record_id}}",
            "line_items": "{{calculate_invoice.output.text}}",
            "total_amount": "{{JSON.parse(calculate_invoice.output.text).total}}",
            "status": "Sent",
            "sent_date": "{{new Date().toISOString().split('T')[0]}}"
          }
        }
      },
      {
        "id": "generate_invoice_pdf",
        "type": "tool_call",
        "description": "Generate a PDF invoice from the template",
        "tool_name": "generate_pdf_from_html_template",
        "input": {
          "template_name": "service_invoice",
          "data": {
            "customer_name": "{{get_customer.output.record.company_name}}",
            "billing_address": "{{get_customer.output.record.billing_address}}",
            "invoice_date": "{{new Date().toISOString().split('T')[0]}}",
            "line_items": "{{calculate_invoice.output.text}}",
            "total": "{{JSON.parse(calculate_invoice.output.text).total}}",
            "service_description": "{{trigger.record_updated.next_data.issue_type}}",
            "completion_notes": "{{trigger.record_updated.next_data.completion_notes}}"
          }
        }
      },
      {
        "id": "email_invoice",
        "type": "tool_call",
        "description": "Email the invoice to the customer",
        "tool_name": "outlook_send_email",
        "input": {
          "to": "{{get_customer.output.record.email}}",
          "subject": "Invoice for HVAC Service — {{trigger.record_updated.next_data.issue_type}}",
          "body": "Hi {{get_customer.output.record.contact_name}},\n\nPlease find attached the invoice for the {{trigger.record_updated.next_data.issue_type}} service completed on {{new Date().toISOString().split('T')[0]}}.\n\nTotal due: ${{JSON.parse(calculate_invoice.output.text).total}}\n\nPayment is due within 30 days. If you have any questions about this invoice, please reply to this email.\n\nThank you for choosing us for your HVAC needs.",
          "attachments": ["{{generate_invoice_pdf.output.file_url}}"]
        }
      }
    ],
    "else": []
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| Invoice delivery time | 5-7 days after service | Same day |
| Invoice error rate | 15% | Under 2% (auto-calculated from catalog) |
| Billing staff time per invoice | 15-20 minutes | 0 minutes (automated) |

-> [Set up this workflow on Lotics](https://lotics.ai)

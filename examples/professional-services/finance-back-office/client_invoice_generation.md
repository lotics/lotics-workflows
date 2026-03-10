# Client Invoice Generation

## The problem

A professional services firm bills clients monthly based on completed milestones and time entries. The finance team manually cross-references timesheets with project milestones, calculates totals, populates a PDF template, and emails it to the client. For 25 active clients, this consumes 3 full days at the end of every month. Invoices frequently contain errors — wrong hours, missing line items, outdated rates — leading to disputes that delay payment by 15-30 days.

## Data model

| Table | Key fields | Purpose |
|-------|-----------|---------|
| Clients | client_name, billing_contact_email, payment_terms, billing_rate | Client master data |
| Time Entries | client (link), consultant, hours, date, description, billable, invoiced | Logged hours against client engagements |
| Invoices | client (link), invoice_number, period_start, period_end, total_amount, status, pdf_file, sent_at | Generated invoices with attached PDFs |

## Example prompts

- "On the first of every month, pull all uninvoiced billable time entries for each client, generate a PDF invoice from our template, and email it to their billing contact."
- "Automate monthly invoicing: aggregate time entries by client, calculate totals at the client's billing rate, create the invoice PDF, and send it."

## Workflow

**Trigger:** `recurring_schedule` — 1st of every month at 09:00.

```json
[
  {
    "id": "get_clients",
    "type": "tool_call",
    "description": "Fetch all active clients",
    "tool_name": "query_records",
    "input": {
      "table_name": "Clients",
      "filters": {
        "status": "Active"
      }
    }
  },
  {
    "id": "process_each_client",
    "type": "foreach",
    "description": "Generate and send an invoice for each active client",
    "items": "{{get_clients.output.records}}",
    "steps": [
      {
        "id": "get_unbilled_time",
        "type": "tool_call",
        "description": "Query uninvoiced billable time entries for this client",
        "tool_name": "query_records",
        "input": {
          "table_name": "Time Entries",
          "filters": {
            "client": "{{item.id}}",
            "billable": true,
            "invoiced": false
          }
        }
      },
      {
        "id": "check_has_entries",
        "type": "if",
        "description": "Only generate invoice if there are billable entries",
        "condition": "{{get_unbilled_time.output.total > 0}}",
        "then": [
          {
            "id": "calculate_totals",
            "type": "tool_call",
            "description": "Aggregate total billable hours for this client",
            "tool_name": "aggregate_records",
            "input": {
              "table_name": "Time Entries",
              "filters": {
                "client": "{{item.id}}",
                "billable": true,
                "invoiced": false
              },
              "aggregations": [
                {
                  "field": "hours",
                  "function": "sum"
                }
              ]
            }
          },
          {
            "id": "generate_invoice_pdf",
            "type": "tool_call",
            "description": "Generate the invoice PDF from the HTML template",
            "tool_name": "generate_pdf_from_html_template",
            "input": {
              "template_name": "client_invoice",
              "data": {
                "client_name": "{{item.client_name}}",
                "invoice_date": "{{today()}}",
                "period_start": "{{startOfMonth(addMonths(today(), -1))}}",
                "period_end": "{{endOfMonth(addMonths(today(), -1))}}",
                "line_items": "{{get_unbilled_time.output.records}}",
                "total_hours": "{{calculate_totals.output.results[0].value}}",
                "billing_rate": "{{item.billing_rate}}",
                "total_amount": "{{calculate_totals.output.results[0].value * item.billing_rate}}"
              }
            }
          },
          {
            "id": "create_invoice_record",
            "type": "tool_call",
            "description": "Create the invoice record in the system",
            "tool_name": "create_record",
            "input": {
              "table_name": "Invoices",
              "data": {
                "client": "{{item.id}}",
                "period_start": "{{startOfMonth(addMonths(today(), -1))}}",
                "period_end": "{{endOfMonth(addMonths(today(), -1))}}",
                "total_amount": "{{calculate_totals.output.results[0].value * item.billing_rate}}",
                "status": "Sent"
              }
            }
          },
          {
            "id": "attach_pdf",
            "type": "tool_call",
            "description": "Attach the generated PDF to the invoice record",
            "tool_name": "add_files_to_record",
            "input": {
              "table_name": "Invoices",
              "record_id": "{{create_invoice_record.output.record_id}}",
              "field_name": "pdf_file",
              "file_ids": ["{{generate_invoice_pdf.output.file_id}}"]
            }
          },
          {
            "id": "email_invoice",
            "type": "tool_call",
            "description": "Send the invoice to the client's billing contact",
            "tool_name": "gmail_send_email",
            "input": {
              "to": "{{item.billing_contact_email}}",
              "subject": "Invoice from {{item.client_name}} — {{startOfMonth(addMonths(today(), -1))}} to {{endOfMonth(addMonths(today(), -1))}}",
              "body": "Please find attached your invoice for the previous billing period.\n\nTotal: ${{calculate_totals.output.results[0].value * item.billing_rate}}\nPayment terms: {{item.payment_terms}}\n\nThank you for your continued partnership."
            }
          }
        ],
        "else": []
      }
    ]
  }
]
```

## Results

| Metric | Before | After |
|--------|--------|-------|
| Time spent on monthly invoicing | 3 days | 0 (automated) |
| Invoice errors per month | 4-6 | 0 |
| Avg. days to payment | 42 days | 28 days |

-> [Set up this workflow on Lotics](https://lotics.ai)

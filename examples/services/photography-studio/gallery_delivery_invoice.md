# Gallery Delivery & Invoice Generation

## The problem

When a photography project is finalized, the studio must send the client a gallery link, generate an invoice PDF, and mark the project as delivered. Currently, the studio manager does this manually: uploads photos to the gallery platform, copies the link into an email, opens the invoice template, fills in line items, exports to PDF, attaches it, and sends. This takes 20-30 minutes per project. During busy season with 15+ deliveries per week, invoices go out days late, delaying payment collection.

## Data model

| Table | Key fields | Purpose |
|-------|-----------|---------|
| Projects | client_name, client_email, shoot_type, shoot_date, gallery_url, total_amount, status, invoice_number | Project records with delivery info |
| Invoice Line Items | project_id, description, quantity, unit_price, line_total | Breakdown of charges |

## Example prompts

- "When a project status changes to 'gallery_ready', generate an invoice PDF from the template, email the client with the gallery link and invoice attached, and mark the project as delivered."
- "Automatically send clients their photo gallery and invoice when the editor marks the project as complete."

## Workflow

**Trigger:** When a project record is updated and status becomes "gallery_ready"

```json
[
  {
    "id": "get_line_items",
    "type": "tool_call",
    "description": "Fetch invoice line items for this project",
    "tool_name": "query_records",
    "input": {
      "table_name": "Invoice Line Items",
      "filters": {
        "project_id": "{{trigger.record_updated.next_data.id}}"
      }
    }
  },
  {
    "id": "generate_invoice",
    "type": "tool_call",
    "description": "Generate the invoice PDF from the HTML template",
    "tool_name": "generate_pdf_from_html_template",
    "input": {
      "template_name": "photography_invoice",
      "data": {
        "invoice_number": "{{trigger.record_updated.next_data.invoice_number}}",
        "client_name": "{{trigger.record_updated.next_data.client_name}}",
        "shoot_type": "{{trigger.record_updated.next_data.shoot_type}}",
        "shoot_date": "{{trigger.record_updated.next_data.shoot_date}}",
        "line_items": "{{get_line_items.output.records}}",
        "total_amount": "{{trigger.record_updated.next_data.total_amount}}"
      }
    }
  },
  {
    "id": "attach_invoice",
    "type": "tool_call",
    "description": "Attach the generated invoice PDF to the project record",
    "tool_name": "add_files_to_record",
    "input": {
      "table_name": "Projects",
      "record_id": "{{trigger.record_updated.next_data.id}}",
      "files": ["{{generate_invoice.output.file_url}}"]
    }
  },
  {
    "id": "send_delivery_email",
    "type": "tool_call",
    "description": "Email the client with gallery link and invoice",
    "tool_name": "gmail_send_email",
    "input": {
      "to": "{{trigger.record_updated.next_data.client_email}}",
      "subject": "Your photos are ready! - {{trigger.record_updated.next_data.shoot_type}} Session",
      "body": "Hi {{trigger.record_updated.next_data.client_name}},\n\nYour photos are ready for viewing and download!\n\nGallery Link: {{trigger.record_updated.next_data.gallery_url}}\n\nYour invoice (Invoice #{{trigger.record_updated.next_data.invoice_number}}) for ${{trigger.record_updated.next_data.total_amount}} is attached to this email.\n\nThe gallery link is active for 30 days. Please download your favorites during this period.\n\nThank you for choosing us. We'd love a review if you're happy with the results!"
    }
  },
  {
    "id": "mark_delivered",
    "type": "tool_call",
    "description": "Update project status to delivered",
    "tool_name": "update_record",
    "input": {
      "table_name": "Projects",
      "record_id": "{{trigger.record_updated.next_data.id}}",
      "data": {
        "status": "delivered"
      }
    }
  }
]
```

## Results

| Metric | Before | After |
|--------|--------|-------|
| Time per delivery + invoice | 25 min | 0 min (automated) |
| Invoice delivery delay | 2-5 days after gallery ready | Instant |
| Average payment collection time | 14 days | 7 days |

-> [Set up this workflow on Lotics](https://lotics.ai)

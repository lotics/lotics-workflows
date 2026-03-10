# Service Report Generation

## The problem

After completing a job, technicians fill out a brief completion form on their phone -- parts used, labor hours, notes, and photos. The office then spends 20 minutes per job reformatting this into a branded service report PDF that gets emailed to the customer. With 25-40 completed jobs per day, the admin team is 2-3 days behind on report delivery. Customers call asking for reports, tying up phone lines.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Work Orders | title, status, customer (link), assigned_tech (link), site_address, completion_notes, parts_used, labor_hours, photos | Completed service jobs |
| Customers | name, email, company, billing_address | Customer directory |
| Technicians | name, phone, license_number | Tech details for report |

## Example prompts

- "When a technician submits the completion form on a work order, generate a branded PDF service report with the job details, parts, labor, and tech info, then email it to the customer automatically."
- "After a work order is marked complete via form submission, pull the customer and tech details, generate the service report PDF, attach it to the work order record, and send it to the customer."

## Workflow

**Trigger:** When the completion form is submitted on a Work Order

```json
[
  {
    "id": "get_work_order",
    "type": "tool_call",
    "description": "Fetch the full work order record",
    "tool_name": "get_record",
    "input": {
      "table_name": "Work Orders",
      "record_id": "{{trigger.form_submitted.record_id}}"
    }
  },
  {
    "id": "get_customer",
    "type": "tool_call",
    "description": "Fetch customer details for the report",
    "tool_name": "get_record",
    "input": {
      "table_name": "Customers",
      "record_id": "{{trigger.form_submitted.data.customer}}"
    }
  },
  {
    "id": "get_tech",
    "type": "tool_call",
    "description": "Fetch technician details for the report",
    "tool_name": "get_record",
    "input": {
      "table_name": "Technicians",
      "record_id": "{{trigger.form_submitted.data.assigned_tech}}"
    }
  },
  {
    "id": "generate_report_pdf",
    "type": "tool_call",
    "description": "Generate a branded PDF service report",
    "tool_name": "generate_pdf_from_html_template",
    "input": {
      "template_name": "service_report",
      "data": {
        "work_order_id": "{{trigger.form_submitted.record_id}}",
        "title": "{{get_work_order.output.record.title}}",
        "site_address": "{{get_work_order.output.record.site_address}}",
        "completion_notes": "{{trigger.form_submitted.data.completion_notes}}",
        "parts_used": "{{trigger.form_submitted.data.parts_used}}",
        "labor_hours": "{{trigger.form_submitted.data.labor_hours}}",
        "customer_name": "{{get_customer.output.record.name}}",
        "customer_company": "{{get_customer.output.record.company}}",
        "tech_name": "{{get_tech.output.record.name}}",
        "tech_license": "{{get_tech.output.record.license_number}}",
        "date": "{{new Date().toISOString().split('T')[0]}}"
      }
    }
  },
  {
    "id": "attach_report",
    "type": "tool_call",
    "description": "Attach the generated PDF to the work order record",
    "tool_name": "add_files_to_record",
    "input": {
      "table_name": "Work Orders",
      "record_id": "{{trigger.form_submitted.record_id}}",
      "files": ["{{generate_report_pdf.output.file_url}}"]
    }
  },
  {
    "id": "update_status",
    "type": "tool_call",
    "description": "Mark work order as Report Sent",
    "tool_name": "update_record",
    "input": {
      "table_name": "Work Orders",
      "record_id": "{{trigger.form_submitted.record_id}}",
      "data": {
        "status": "Report Sent"
      }
    }
  },
  {
    "id": "email_customer",
    "type": "tool_call",
    "description": "Email the service report to the customer",
    "tool_name": "gmail_send_email",
    "input": {
      "to": "{{get_customer.output.record.email}}",
      "subject": "Service Report — {{get_work_order.output.record.title}}",
      "body": "Hi {{get_customer.output.record.name}},\n\nPlease find attached the service report for the work completed at {{get_work_order.output.record.site_address}}.\n\nSummary:\n- Technician: {{get_tech.output.record.name}}\n- Labor hours: {{trigger.form_submitted.data.labor_hours}}\n- Parts used: {{trigger.form_submitted.data.parts_used}}\n\nPlease let us know if you have any questions.\n\nThank you for your business.",
      "attachments": ["{{generate_report_pdf.output.file_url}}"]
    }
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| Report delivery time | 2-3 days | Under 5 minutes |
| Admin time per report | 20 minutes | 0 minutes (automated) |
| Customer follow-up calls about reports | 10-15/week | Near zero |

-> [Set up this workflow on Lotics](https://lotics.ai)

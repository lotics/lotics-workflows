# Vendor Bill Processing

## The problem

A mid-size professional services firm receives 40-60 vendor bills per month via email — software subscriptions, subcontractor invoices, office supplies, travel bookings. An office manager manually downloads attachments, enters data into a spreadsheet, routes bills for department head approval, and then forwards to accounting. Bills get buried in inboxes, data entry errors cause payment disputes, and late payments damage vendor relationships and trigger penalty fees averaging $1,200/quarter.

## Data model

| Table | Key fields | Purpose |
|-------|-----------|---------|
| Vendor Bills | vendor_name, amount, due_date, category, department, status, bill_file, extracted_data, approved_by | Incoming bills from vendors |
| Vendors | vendor_name, contact_email, payment_method, default_category, department_owner | Vendor master data |
| Payment Queue | vendor_bill (link), payment_amount, scheduled_date, payment_status | Bills approved and scheduled for payment |

## Example prompts

- "When a vendor bill arrives by email, extract the amount and due date from the attachment, match it to a vendor, route it for approval, and queue it for payment once approved."
- "Automate our vendor bill intake: pull data from PDF invoices, create records, get department head sign-off, and schedule payments."

## Workflow

**Trigger:** `receive_gmail_email` — emails matching vendor bill patterns arrive in a monitored inbox.

```json
[
  {
    "id": "extract_bill_data",
    "type": "tool_call",
    "description": "Extract structured data from the email attachment (vendor invoice PDF)",
    "tool_name": "extract_file_data",
    "input": {
      "file_id": "{{trigger.receive_gmail_email.attachment_ids[0]}}",
      "extraction_prompt": "Extract the following from this vendor invoice: vendor_name, invoice_number, amount, due_date, line_item_descriptions. Return as JSON."
    }
  },
  {
    "id": "match_vendor",
    "type": "tool_call",
    "description": "Look up the vendor in the vendor master table",
    "tool_name": "query_records",
    "input": {
      "table_name": "Vendors",
      "filters": {
        "vendor_name": "{{extract_bill_data.output.extracted.vendor_name}}"
      }
    }
  },
  {
    "id": "create_bill_record",
    "type": "tool_call",
    "description": "Create the vendor bill record with extracted data",
    "tool_name": "create_record",
    "input": {
      "table_name": "Vendor Bills",
      "data": {
        "vendor_name": "{{extract_bill_data.output.extracted.vendor_name}}",
        "amount": "{{extract_bill_data.output.extracted.amount}}",
        "due_date": "{{extract_bill_data.output.extracted.due_date}}",
        "category": "{{match_vendor.output.records[0].default_category}}",
        "department": "{{match_vendor.output.records[0].department_owner}}",
        "status": "Pending Approval",
        "extracted_data": "{{JSON.stringify(extract_bill_data.output.extracted)}}"
      }
    }
  },
  {
    "id": "attach_bill_file",
    "type": "tool_call",
    "description": "Attach the original invoice PDF to the bill record",
    "tool_name": "add_files_to_record",
    "input": {
      "table_name": "Vendor Bills",
      "record_id": "{{create_bill_record.output.record_id}}",
      "field_name": "bill_file",
      "file_ids": "{{trigger.receive_gmail_email.attachment_ids}}"
    }
  },
  {
    "id": "get_department_members",
    "type": "tool_call",
    "description": "Find the department head who needs to approve this bill",
    "tool_name": "query_members",
    "input": {
      "filters": {
        "role": "Department Head",
        "department": "{{match_vendor.output.records[0].department_owner}}"
      }
    }
  },
  {
    "id": "request_approval",
    "type": "tool_call",
    "description": "Notify the department head to review and approve the bill",
    "tool_name": "send_notification",
    "input": {
      "member_id": "{{get_department_members.output.members[0].id}}",
      "title": "Vendor bill needs approval — ${{extract_bill_data.output.extracted.amount}}",
      "message": "A bill from {{extract_bill_data.output.extracted.vendor_name}} for ${{extract_bill_data.output.extracted.amount}} (due {{extract_bill_data.output.extracted.due_date}}) needs your approval."
    }
  },
  {
    "id": "wait_for_approval",
    "type": "wait_for_event",
    "description": "Wait for the department head to approve or reject the bill",
    "event": "record_updated",
    "condition": "{{trigger.record_updated.next_data.status == 'Approved' || trigger.record_updated.next_data.status == 'Rejected'}}"
  },
  {
    "id": "check_approved",
    "type": "if",
    "description": "If approved, queue for payment; if rejected, notify accounting",
    "condition": "{{wait_for_approval.output.next_data.status == 'Approved'}}",
    "then": [
      {
        "id": "queue_payment",
        "type": "tool_call",
        "description": "Create a payment queue entry scheduled before the due date",
        "tool_name": "create_record",
        "input": {
          "table_name": "Payment Queue",
          "data": {
            "vendor_bill": "{{create_bill_record.output.record_id}}",
            "payment_amount": "{{extract_bill_data.output.extracted.amount}}",
            "scheduled_date": "{{extract_bill_data.output.extracted.due_date}}",
            "payment_status": "Scheduled"
          }
        }
      },
      {
        "id": "confirm_to_sender",
        "type": "tool_call",
        "description": "Reply to the original vendor email confirming receipt and scheduled payment",
        "tool_name": "gmail_reply_email",
        "input": {
          "email_id": "{{trigger.receive_gmail_email.email_id}}",
          "body": "Thank you. Your invoice #{{extract_bill_data.output.extracted.invoice_number}} for ${{extract_bill_data.output.extracted.amount}} has been approved and scheduled for payment by {{extract_bill_data.output.extracted.due_date}}."
        }
      }
    ],
    "else": [
      {
        "id": "notify_rejection",
        "type": "tool_call",
        "description": "Reply to vendor that the bill is being reviewed further",
        "tool_name": "gmail_reply_email",
        "input": {
          "email_id": "{{trigger.receive_gmail_email.email_id}}",
          "body": "We have received your invoice #{{extract_bill_data.output.extracted.invoice_number}}. Our team is reviewing it and will follow up with any questions."
        }
      }
    ]
  }
]
```

## Results

| Metric | Before | After |
|--------|--------|-------|
| Bill processing time | 20 min per bill | < 1 min (automated) |
| Late payment penalties per quarter | $1,200 | $0 |
| Data entry errors per month | 8-10 | 0 |

-> [Set up this workflow on Lotics](https://lotics.ai)

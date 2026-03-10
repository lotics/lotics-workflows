# Vendor Bid Collection and Comparison

## The problem

Before breaking ground, a development team solicits bids from 5-10 subcontractors per trade (concrete, steel, electrical, plumbing, HVAC). For a mid-size project, this means collecting and comparing 30-60 bids across 6-8 trades. Bid documents arrive as email attachments in inconsistent formats -- some as PDFs, others as Excel sheets. A project estimator spends 15-20 hours manually extracting line items, normalizing units, and building comparison spreadsheets. Bids that arrive late or with missing scope items are often overlooked, leading to change orders that inflate costs by 5-10%.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Bid Packages | package_id, project_id, trade, scope_description, due_date, status | Groups of bids solicited for a specific trade |
| Bids | bid_id, package_id, vendor_name, vendor_email, total_amount, submitted_at, file_id, extraction_status | Individual vendor bid submissions |
| Bid Line Items | item_id, bid_id, description, quantity, unit, unit_price, total | Extracted line items from each bid for comparison |

## Example prompts

- "When a bid email comes in from a vendor, extract the attachment, create a bid record, pull out line items using AI, and notify the estimator that a new bid is ready for review."
- "Automate bid intake: capture vendor bid emails, extract the PDF data into structured line items, save everything to the Bids table, and alert the team."

## Workflow

**Trigger:** An email is received at the project bids inbox matching the bid package subject line.

```json
[
  {
    "id": "read_bid_email",
    "type": "tool_call",
    "description": "Read the full email content and metadata",
    "tool_name": "gmail_read_email",
    "input": {
      "email_id": "{{trigger.receive_gmail_email.email_id}}"
    }
  },
  {
    "id": "find_package",
    "type": "tool_call",
    "description": "Match the email to an open bid package based on the subject line",
    "tool_name": "query_records",
    "input": {
      "table_name": "Bid Packages",
      "filters": {
        "status": "Open"
      }
    }
  },
  {
    "id": "create_bid_record",
    "type": "tool_call",
    "description": "Create a bid record for this vendor submission",
    "tool_name": "create_record",
    "input": {
      "table_name": "Bids",
      "data": {
        "package_id": "{{find_package.output.records[0].package_id}}",
        "vendor_name": "{{read_bid_email.output.from_name}}",
        "vendor_email": "{{read_bid_email.output.from_address}}",
        "submitted_at": "{{NOW()}}",
        "extraction_status": "Processing"
      }
    }
  },
  {
    "id": "attach_bid_file",
    "type": "tool_call",
    "description": "Attach the bid document from the email to the bid record",
    "tool_name": "add_files_to_record",
    "input": {
      "table_name": "Bids",
      "record_id": "{{create_bid_record.output.record_id}}",
      "files": ["{{read_bid_email.output.attachments[0].url}}"]
    }
  },
  {
    "id": "extract_bid_data",
    "type": "tool_call",
    "description": "Extract structured data from the bid document using AI",
    "tool_name": "extract_file_data",
    "input": {
      "file_url": "{{read_bid_email.output.attachments[0].url}}",
      "extraction_schema": {
        "total_amount": "number",
        "line_items": [
          {
            "description": "string",
            "quantity": "number",
            "unit": "string",
            "unit_price": "number",
            "total": "number"
          }
        ]
      }
    }
  },
  {
    "id": "update_bid_total",
    "type": "tool_call",
    "description": "Update the bid record with the extracted total amount",
    "tool_name": "update_record",
    "input": {
      "table_name": "Bids",
      "record_id": "{{create_bid_record.output.record_id}}",
      "data": {
        "total_amount": "{{extract_bid_data.output.total_amount}}",
        "extraction_status": "Complete"
      }
    }
  },
  {
    "id": "create_line_items",
    "type": "foreach",
    "description": "Create a record for each extracted line item",
    "items": "{{extract_bid_data.output.line_items}}",
    "steps": [
      {
        "id": "create_item",
        "type": "tool_call",
        "description": "Create a bid line item record",
        "tool_name": "create_record",
        "input": {
          "table_name": "Bid Line Items",
          "data": {
            "bid_id": "{{create_bid_record.output.record_id}}",
            "description": "{{item.description}}",
            "quantity": "{{item.quantity}}",
            "unit": "{{item.unit}}",
            "unit_price": "{{item.unit_price}}",
            "total": "{{item.total}}"
          }
        }
      }
    ]
  },
  {
    "id": "notify_estimator",
    "type": "tool_call",
    "description": "Notify the estimator that a new bid has been processed",
    "tool_name": "send_notification",
    "input": {
      "recipient_id": "estimator",
      "message": "New bid received from {{read_bid_email.output.from_name}} for {{find_package.output.records[0].trade}} (Bid Package {{find_package.output.records[0].package_id}}). Total: ${{extract_bid_data.output.total_amount}}. {{extract_bid_data.output.line_items.length}} line items extracted and ready for comparison."
    }
  },
  {
    "id": "acknowledge_vendor",
    "type": "tool_call",
    "description": "Send the vendor an acknowledgment reply",
    "tool_name": "gmail_reply_email",
    "input": {
      "email_id": "{{trigger.receive_gmail_email.email_id}}",
      "body": "Thank you for your bid submission. We have received and logged your proposal. Our team will review all bids and follow up by the package due date.\n\nBest regards,\nProject Estimating Team"
    }
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| Hours to process all bids per trade | 3-4 hours | Under 10 minutes |
| Total estimator hours per project bid cycle | 15-20 hours | 2-3 hours (review only) |
| Bids with missed line items | 10-15% | 0% (AI extraction) |
| Vendor acknowledgment time | 1-2 business days | Immediate |

-> [Set up this workflow on Lotics](https://lotics.ai)

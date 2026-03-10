# Supplier Quote Collection and Comparison

## The problem

A contract manufacturer sources 200+ unique components from a pool of 45 suppliers. For each new project, procurement analysts email 3-5 suppliers for quotes, then manually compile responses into a comparison spreadsheet. The process takes 4-6 days per RFQ cycle. Quotes arrive in different formats -- some as PDF attachments, some as inline email text -- and analysts spend 2 hours per RFQ just normalizing the data. Late quotes are frequently missed because there is no systematic follow-up.

## Data model

| Table | Key fields | Purpose |
|-------|-----------|---------|
| RFQ Requests | rfq_id, component, quantity, target_price, deadline, status | Request for quote initiated by procurement |
| Supplier Quotes | quote_id, rfq_id, supplier_id, unit_price, lead_time_days, moq, valid_until, source_email_id | Individual supplier responses |
| Suppliers | supplier_id, name, email, category, rating, payment_terms | Approved supplier directory |

## Example prompts

- "When a supplier replies to an RFQ email, extract the quoted price, lead time, and MOQ from the email body or attachment, and add it to the quote comparison table."
- "Automatically capture supplier quote details from incoming emails and build a side-by-side comparison for each open RFQ."

## Workflow

**Trigger:** Incoming Gmail email matching the RFQ subject line pattern

```json
[
  {
    "id": "read_email",
    "type": "tool_call",
    "description": "Read the full email content including attachments",
    "tool_name": "gmail_read_email",
    "input": {
      "email_id": "{{trigger.receive_gmail_email.email_id}}"
    }
  },
  {
    "id": "extract_quote",
    "type": "tool_call",
    "description": "Use LLM to extract structured quote data from the email",
    "tool_name": "llm_generate_text",
    "input": {
      "prompt": "Extract the following from this supplier quote email. Return valid JSON with keys: rfq_id, unit_price (number), lead_time_days (number), moq (number), valid_until (YYYY-MM-DD). If any field is missing, use null.\n\nFrom: {{read_email.output.from}}\nSubject: {{read_email.output.subject}}\nBody: {{read_email.output.body}}"
    }
  },
  {
    "id": "find_rfq",
    "type": "tool_call",
    "description": "Look up the matching RFQ record",
    "tool_name": "query_records",
    "input": {
      "table_id": "rfq_requests",
      "filters": {
        "rfq_id": "{{extract_quote.output.text.rfq_id}}",
        "status": "Open"
      }
    }
  },
  {
    "id": "check_rfq_exists",
    "type": "if",
    "description": "Verify the RFQ exists and is still open",
    "condition": "{{find_rfq.output.records.length > 0}}",
    "then": [
      {
        "id": "find_supplier",
        "type": "tool_call",
        "description": "Match the sender email to a supplier record",
        "tool_name": "query_records",
        "input": {
          "table_id": "suppliers",
          "filters": {
            "email": "{{read_email.output.from}}"
          }
        }
      },
      {
        "id": "save_quote",
        "type": "tool_call",
        "description": "Create the supplier quote record with extracted data",
        "tool_name": "create_record",
        "input": {
          "table_id": "supplier_quotes",
          "data": {
            "rfq_id": "{{extract_quote.output.text.rfq_id}}",
            "supplier_id": "{{find_supplier.output.records[0].supplier_id}}",
            "unit_price": "{{extract_quote.output.text.unit_price}}",
            "lead_time_days": "{{extract_quote.output.text.lead_time_days}}",
            "moq": "{{extract_quote.output.text.moq}}",
            "valid_until": "{{extract_quote.output.text.valid_until}}",
            "source_email_id": "{{trigger.receive_gmail_email.email_id}}"
          }
        }
      },
      {
        "id": "acknowledge_supplier",
        "type": "tool_call",
        "description": "Send an automated acknowledgment to the supplier",
        "tool_name": "gmail_reply_email",
        "input": {
          "email_id": "{{trigger.receive_gmail_email.email_id}}",
          "body": "Thank you for your quote on RFQ {{extract_quote.output.text.rfq_id}}. We have received your submission and will follow up shortly."
        }
      },
      {
        "id": "notify_analyst",
        "type": "tool_call",
        "description": "Notify the procurement analyst that a new quote arrived",
        "tool_name": "send_notification",
        "input": {
          "title": "New quote received for RFQ {{extract_quote.output.text.rfq_id}}",
          "message": "{{find_supplier.output.records[0].name}} quoted ${{extract_quote.output.text.unit_price}}/unit with {{extract_quote.output.text.lead_time_days}} day lead time."
        }
      }
    ],
    "else": [
      {
        "id": "notify_no_match",
        "type": "tool_call",
        "description": "Alert procurement that a quote could not be matched to an open RFQ",
        "tool_name": "send_notification",
        "input": {
          "title": "Unmatched supplier quote received",
          "message": "An email from {{read_email.output.from}} appears to be a quote but could not be matched to an open RFQ. Subject: {{read_email.output.subject}}"
        }
      }
    ]
  }
]
```

## Results

| Metric | Before | After |
|--------|--------|-------|
| Quote processing time | 2 hours per RFQ | < 2 minutes |
| RFQ cycle time | 4-6 days | 1-2 days |
| Missed late quotes | 20% | 0% |

-> [Set up this workflow on Lotics](https://lotics.ai)

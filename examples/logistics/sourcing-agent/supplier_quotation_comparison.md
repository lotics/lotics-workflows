# Supplier Quotation Comparison

## The problem

Sourcing agents receive quotations from 5-15 suppliers per product inquiry, arriving as PDF attachments, email bodies, and spreadsheet files in varying formats. Comparing quotes requires manually extracting unit price, MOQ, lead time, payment terms, and shipping terms from each document, then building a comparison matrix. This takes 2-4 hours per inquiry and is error-prone — a missed decimal point or overlooked payment term can cost the buyer thousands. During peak season, agents handle 30+ inquiries per week and comparison backlogs delay purchasing decisions by days.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Inquiries | inquiry_ref, product_description, target_price, target_qty, buyer_id, status | Buyer inquiry master |
| Quotations | inquiry_id, supplier_id, unit_price, currency, moq, lead_time_days, payment_terms, incoterm, validity_date, status | Supplier quotes |
| Suppliers | name, email, country, rating | Supplier directory |
| Comparison Reports | inquiry_id, recommended_supplier_id, report_summary, file_id, created_at | Final comparison output |

## Example prompts

- "When a supplier sends a quotation by email, extract the pricing details and add it to the inquiry. When all quotes are in, generate a comparison report."
- "Automatically process incoming quotation emails and build a side-by-side comparison for the buyer."

## Workflow

**Trigger:** Gmail email received matching label "Supplier Quotations".

```json
[
  {
    "id": "read_quote_email",
    "type": "tool_call",
    "description": "Read the incoming quotation email",
    "tool_name": "gmail_read_email",
    "input": {
      "email_id": "{{trigger.receive_gmail_email.email_id}}"
    }
  },
  {
    "id": "extract_quote_data",
    "type": "tool_call",
    "description": "Extract quotation details from the email attachment",
    "tool_name": "extract_file_data",
    "input": {
      "file_id": "{{read_quote_email.output.attachments[0].file_id}}",
      "extraction_prompt": "Extract quotation details: supplier name, product description, unit price, currency, minimum order quantity (MOQ), lead time in days, payment terms, incoterm (FOB/CIF/EXW etc.), quotation validity date. Also extract any referenced inquiry or RFQ number. Return JSON with these fields."
    }
  },
  {
    "id": "find_inquiry",
    "type": "tool_call",
    "description": "Match the quotation to an open inquiry",
    "tool_name": "query_records",
    "input": {
      "table_id": "inquiries",
      "filter": {
        "and": [
          { "field": "inquiry_ref", "operator": "contains", "value": "{{JSON.parse(extract_quote_data.output.data).inquiry_ref}}" },
          { "field": "status", "operator": "equals", "value": "quoting" }
        ]
      }
    }
  },
  {
    "id": "find_supplier",
    "type": "tool_call",
    "description": "Match the supplier from the quotation",
    "tool_name": "query_records",
    "input": {
      "table_id": "suppliers",
      "filter": {
        "field": "name",
        "operator": "contains",
        "value": "{{JSON.parse(extract_quote_data.output.data).supplier_name}}"
      }
    }
  },
  {
    "id": "create_quotation",
    "type": "tool_call",
    "description": "Create a quotation record from the extracted data",
    "tool_name": "create_record",
    "input": {
      "table_id": "quotations",
      "data": {
        "inquiry_id": "{{find_inquiry.output.data[0].id}}",
        "supplier_id": "{{find_supplier.output.data[0].id}}",
        "unit_price": "{{JSON.parse(extract_quote_data.output.data).unit_price}}",
        "currency": "{{JSON.parse(extract_quote_data.output.data).currency}}",
        "moq": "{{JSON.parse(extract_quote_data.output.data).moq}}",
        "lead_time_days": "{{JSON.parse(extract_quote_data.output.data).lead_time_days}}",
        "payment_terms": "{{JSON.parse(extract_quote_data.output.data).payment_terms}}",
        "incoterm": "{{JSON.parse(extract_quote_data.output.data).incoterm}}",
        "validity_date": "{{JSON.parse(extract_quote_data.output.data).validity_date}}",
        "status": "received"
      }
    }
  },
  {
    "id": "attach_source",
    "type": "tool_call",
    "description": "Attach the original quotation file to the record",
    "tool_name": "add_files_to_record",
    "input": {
      "table_id": "quotations",
      "record_id": "{{create_quotation.output.data.id}}",
      "file_ids": ["{{read_quote_email.output.attachments[0].file_id}}"]
    }
  },
  {
    "id": "notify_agent",
    "type": "tool_call",
    "description": "Notify the sourcing agent that a new quotation has been processed",
    "tool_name": "send_notification",
    "input": {
      "message": "New quotation from {{find_supplier.output.data[0].name}} for inquiry {{find_inquiry.output.data[0].inquiry_ref}}: ${{JSON.parse(extract_quote_data.output.data).unit_price}}/unit, MOQ {{JSON.parse(extract_quote_data.output.data).moq}}, {{JSON.parse(extract_quote_data.output.data).lead_time_days}} days lead time.",
      "record_id": "{{create_quotation.output.data.id}}"
    }
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| Time to process one quotation | 20-30 minutes | Under 2 minutes |
| Data extraction errors | 5-8% of quotes | Under 1% |
| Inquiry-to-decision time | 5-7 days | 2-3 days |

→ [Set up this workflow on Lotics](https://lotics.ai)

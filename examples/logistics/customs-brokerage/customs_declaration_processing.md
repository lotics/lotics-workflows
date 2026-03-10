# Customs Declaration Document Processing

## The problem

Customs brokers receive import documents — commercial invoices, packing lists, bills of lading — via email from importers and freight forwarders. A broker handling 20+ declarations per day must manually open each email, download attachments, extract commodity descriptions, HS codes, values, and weights, then key them into the declaration system. This data entry takes 15-25 minutes per declaration and is error-prone: a wrong HS code means incorrect duty calculation, and a wrong value triggers customs audit flags.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Declarations | declaration_number, importer_id, status, total_value, total_duty, submitted_at | Customs declaration master |
| Declaration Lines | declaration_id, hs_code, description, qty, unit_value, total_value, origin_country, duty_rate | Line items per declaration |
| Importers | name, email, tin_number | Importer details |
| Source Documents | declaration_id, doc_type, file_id, extracted_data | Uploaded source docs |

## Example prompts

- "When I receive an email with import documents, extract the data from the attachments and create a draft customs declaration with all line items."
- "Automatically process incoming import document emails into customs declaration drafts."

## Workflow

**Trigger:** Gmail email received matching label "Import Documents".

```json
[
  {
    "id": "read_email",
    "type": "tool_call",
    "description": "Read the incoming email with import documents",
    "tool_name": "gmail_read_email",
    "input": {
      "email_id": "{{trigger.receive_gmail_email.email_id}}"
    }
  },
  {
    "id": "extract_invoice_data",
    "type": "tool_call",
    "description": "Extract commodity and value data from the commercial invoice attachment",
    "tool_name": "extract_file_data",
    "input": {
      "file_id": "{{read_email.output.attachments[0].file_id}}",
      "extraction_prompt": "Extract from this commercial invoice: importer name, invoice number, invoice date, currency, and for each line item: description of goods, quantity, unit price, total value, country of origin, and suggested HS code (6-digit). Return JSON: { \"importer_name\": string, \"invoice_number\": string, \"invoice_date\": string, \"currency\": string, \"line_items\": [{ \"description\": string, \"qty\": number, \"unit_price\": number, \"total_value\": number, \"origin_country\": string, \"hs_code\": string }], \"grand_total\": number }"
    }
  },
  {
    "id": "find_importer",
    "type": "tool_call",
    "description": "Look up the importer in our records",
    "tool_name": "query_records",
    "input": {
      "table_id": "importers",
      "filter": {
        "field": "name",
        "operator": "contains",
        "value": "{{JSON.parse(extract_invoice_data.output.data).importer_name}}"
      }
    }
  },
  {
    "id": "classify_hs_codes",
    "type": "tool_call",
    "description": "Validate and refine HS codes with duty rates using LLM knowledge",
    "tool_name": "llm_generate_text",
    "input": {
      "prompt": "Review these line items and validate/correct the HS codes. Assign the most accurate 8-digit HS code and the applicable import duty rate percentage. Items: {{JSON.stringify(JSON.parse(extract_invoice_data.output.data).line_items)}}. Return JSON array: [{ \"description\": string, \"hs_code\": string, \"duty_rate\": number, \"qty\": number, \"unit_price\": number, \"total_value\": number, \"origin_country\": string }]. Only return JSON."
    }
  },
  {
    "id": "create_declaration",
    "type": "tool_call",
    "description": "Create the draft customs declaration",
    "tool_name": "create_record",
    "input": {
      "table_id": "declarations",
      "data": {
        "importer_id": "{{find_importer.output.data[0].id}}",
        "status": "draft",
        "total_value": "{{JSON.parse(extract_invoice_data.output.data).grand_total}}",
        "source_email_subject": "{{read_email.output.subject}}"
      }
    }
  },
  {
    "id": "create_line_items",
    "type": "foreach",
    "description": "Create declaration line items with validated HS codes",
    "items": "{{JSON.parse(classify_hs_codes.output.text)}}",
    "steps": [
      {
        "id": "create_line",
        "type": "tool_call",
        "description": "Create a declaration line item",
        "tool_name": "create_record",
        "input": {
          "table_id": "declaration_lines",
          "data": {
            "declaration_id": "{{create_declaration.output.data.id}}",
            "hs_code": "{{create_line_items.item.hs_code}}",
            "description": "{{create_line_items.item.description}}",
            "qty": "{{create_line_items.item.qty}}",
            "unit_value": "{{create_line_items.item.unit_price}}",
            "total_value": "{{create_line_items.item.total_value}}",
            "origin_country": "{{create_line_items.item.origin_country}}",
            "duty_rate": "{{create_line_items.item.duty_rate}}"
          }
        }
      }
    ]
  },
  {
    "id": "store_source_doc",
    "type": "tool_call",
    "description": "Link the source document to the declaration",
    "tool_name": "create_record",
    "input": {
      "table_id": "source_documents",
      "data": {
        "declaration_id": "{{create_declaration.output.data.id}}",
        "doc_type": "commercial_invoice",
        "file_id": "{{read_email.output.attachments[0].file_id}}",
        "extracted_data": "{{extract_invoice_data.output.data}}"
      }
    }
  },
  {
    "id": "notify_broker",
    "type": "tool_call",
    "description": "Notify the customs broker that a draft declaration is ready for review",
    "tool_name": "send_notification",
    "input": {
      "message": "Draft declaration created from email: {{read_email.output.subject}}. {{JSON.parse(classify_hs_codes.output.text).length}} line items extracted. Please review HS codes and duty rates before submission.",
      "record_id": "{{create_declaration.output.data.id}}"
    }
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| Declaration data entry time | 15-25 min each | Under 2 min review |
| HS code classification errors | 8-12% | Under 2% |
| Declarations processed per broker per day | 15-20 | 35-45 |

→ [Set up this workflow on Lotics](https://lotics.ai)

# Duty Payment Tracking and Client Billing

## The problem

Customs brokers pay import duties on behalf of their clients and must recover these advances promptly. A broker processing 300+ declarations per month advances $200,000-500,000 in duties, and when clients are slow to reimburse, cash flow suffers. The accounting team manually reconciles duty payments against client invoices in spreadsheets, and overdue advances often go unnoticed for weeks. Brokers with 60+ day outstanding advances risk financial exposure and strained client relationships.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Declarations | declaration_number, importer_id, total_duty, duty_paid_at, status | Declaration with duty amounts |
| Duty Advances | declaration_id, importer_id, amount, paid_date, reimbursed, reimbursed_date | Duty payment tracking |
| Importers | name, email, billing_email, credit_limit | Client billing info |

## Example prompts

- "Every Friday, find duty advances older than 30 days that haven't been reimbursed and send a statement to each client."
- "Weekly check for overdue duty reimbursements and email clients their outstanding balance."

## Workflow

**Trigger:** Recurring weekly schedule every Friday at 10:00 UTC.

```json
[
  {
    "id": "find_overdue_advances",
    "type": "tool_call",
    "description": "Query duty advances not reimbursed and older than 30 days",
    "tool_name": "query_records",
    "input": {
      "table_id": "duty_advances",
      "filter": {
        "and": [
          { "field": "reimbursed", "operator": "equals", "value": false },
          { "field": "paid_date", "operator": "less_than", "value": "{{DATE_ADD(NOW(), -30, 'days')}}" }
        ]
      }
    }
  },
  {
    "id": "check_any_overdue",
    "type": "if",
    "description": "Only proceed if there are overdue advances",
    "condition": "{{find_overdue_advances.output.data.length > 0}}",
    "then": [
      {
        "id": "group_by_client",
        "type": "tool_call",
        "description": "Group overdue advances by importer for consolidated statements",
        "tool_name": "llm_generate_text",
        "input": {
          "prompt": "Group these overdue duty advances by importer_id. Advances: {{JSON.stringify(find_overdue_advances.output.data)}}. Return JSON: { \"importers\": [{ \"importer_id\": string, \"advances\": [{ \"declaration_id\": string, \"amount\": number, \"paid_date\": string }], \"total_outstanding\": number }] }. Only return JSON."
        }
      },
      {
        "id": "send_statements",
        "type": "foreach",
        "description": "Send an outstanding balance statement to each client",
        "items": "{{JSON.parse(group_by_client.output.text).importers}}",
        "steps": [
          {
            "id": "get_importer",
            "type": "tool_call",
            "description": "Fetch importer billing details",
            "tool_name": "get_record",
            "input": {
              "table_id": "importers",
              "record_id": "{{send_statements.item.importer_id}}"
            }
          },
          {
            "id": "compose_statement",
            "type": "tool_call",
            "description": "Generate a professional outstanding balance statement email",
            "tool_name": "llm_generate_text",
            "input": {
              "prompt": "Write a professional but firm outstanding duty reimbursement email. Client: {{get_importer.output.data.name}}. Outstanding advances: {{JSON.stringify(send_statements.item.advances)}}. Total outstanding: ${{send_statements.item.total_outstanding}}. All items are 30+ days past due. Request immediate payment. Keep it under 150 words."
            }
          },
          {
            "id": "email_statement",
            "type": "tool_call",
            "description": "Email the overdue statement to the client's billing contact",
            "tool_name": "gmail_send_email",
            "input": {
              "to": "{{get_importer.output.data.billing_email}}",
              "subject": "Outstanding Duty Reimbursement — ${{send_statements.item.total_outstanding}} overdue",
              "body": "{{compose_statement.output.text}}"
            }
          }
        ]
      }
    ],
    "else": []
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| Average days to collect duty reimbursement | 45-60 days | 25-30 days |
| Outstanding advances over 60 days | $50,000-100,000 | Under $15,000 |
| Hours spent on manual reconciliation | 6 hrs/week | 1 hr/week |

→ [Set up this workflow on Lotics](https://lotics.ai)

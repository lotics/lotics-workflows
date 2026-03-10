# Lease Expiry Notification and Renewal Pipeline

## The problem

A portfolio of 300 residential units has leases expiring on rolling dates throughout the year. Property managers must send renewal offers 90 days before expiry, follow up at 60 and 30 days, and prepare vacancy marketing if the tenant declines. When this is tracked manually in spreadsheets, 10-15% of renewals are initiated late, leading to month-to-month holdovers at below-market rates, unplanned vacancies, and $2,000-5,000 in lost revenue per missed renewal.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Leases | lease_id, unit_id, tenant_name, tenant_email, start_date, end_date, monthly_rent, renewal_status | Active and historical lease agreements |
| Units | unit_id, building, address, bedrooms, status | Property unit inventory |
| Renewal Offers | offer_id, lease_id, proposed_rent, sent_at, response, response_date | Track outbound renewal offers and tenant responses |

## Example prompts

- "Every day at 8am, check for leases expiring in exactly 90 days, generate a renewal offer with a 3% rent increase, and email the tenant a PDF letter."
- "Build a daily workflow that finds leases hitting the 90-day-out mark, creates a renewal offer record, generates the offer letter as a PDF, and sends it to the tenant."

## Workflow

**Trigger:** Recurring daily schedule at 8:00 AM.

```json
[
  {
    "id": "find_expiring_leases",
    "type": "tool_call",
    "description": "Query leases expiring in exactly 90 days with no renewal initiated",
    "tool_name": "query_records",
    "input": {
      "table_name": "Leases",
      "filters": {
        "end_date": "{{DATE_ADD(NOW(), 90, 'day')}}",
        "renewal_status": "Not Started"
      }
    }
  },
  {
    "id": "process_each_lease",
    "type": "foreach",
    "description": "Process each expiring lease: create offer, generate PDF, email tenant",
    "items": "{{find_expiring_leases.output.records}}",
    "steps": [
      {
        "id": "create_offer",
        "type": "tool_call",
        "description": "Create a renewal offer record with a 3% rent increase",
        "tool_name": "create_record",
        "input": {
          "table_name": "Renewal Offers",
          "data": {
            "lease_id": "{{item.lease_id}}",
            "proposed_rent": "{{item.monthly_rent * 1.03}}",
            "sent_at": "{{NOW()}}",
            "response": "Pending"
          }
        }
      },
      {
        "id": "generate_letter",
        "type": "tool_call",
        "description": "Generate a PDF renewal offer letter from template",
        "tool_name": "generate_pdf_from_html_template",
        "input": {
          "template_name": "Renewal Offer Letter",
          "data": {
            "tenant_name": "{{item.tenant_name}}",
            "unit_address": "{{item.unit_id}}",
            "current_rent": "{{item.monthly_rent}}",
            "proposed_rent": "{{item.monthly_rent * 1.03}}",
            "lease_end_date": "{{item.end_date}}",
            "response_deadline": "{{DATE_ADD(NOW(), 30, 'day')}}"
          }
        }
      },
      {
        "id": "attach_letter",
        "type": "tool_call",
        "description": "Attach the generated PDF to the renewal offer record",
        "tool_name": "add_files_to_record",
        "input": {
          "table_name": "Renewal Offers",
          "record_id": "{{create_offer.output.record_id}}",
          "files": ["{{generate_letter.output.file_url}}"]
        }
      },
      {
        "id": "email_tenant",
        "type": "tool_call",
        "description": "Email the renewal offer letter to the tenant",
        "tool_name": "gmail_send_email",
        "input": {
          "to": "{{item.tenant_email}}",
          "subject": "Your Lease Renewal Offer - Action Required by {{DATE_ADD(NOW(), 30, 'day')}}",
          "body": "Dear {{item.tenant_name}},\n\nYour current lease expires on {{item.end_date}}. We are pleased to offer you a renewal at ${{item.monthly_rent * 1.03}}/month.\n\nPlease review the attached offer letter and respond within 30 days.\n\nBest regards,\nProperty Management",
          "attachments": ["{{generate_letter.output.file_url}}"]
        }
      },
      {
        "id": "mark_initiated",
        "type": "tool_call",
        "description": "Update the lease renewal status to Offer Sent",
        "tool_name": "update_record",
        "input": {
          "table_name": "Leases",
          "record_id": "{{item.lease_id}}",
          "data": {
            "renewal_status": "Offer Sent"
          }
        }
      }
    ]
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| Renewals initiated on time | 85% | 100% |
| Month-to-month holdovers | 12-18 per year | 0 |
| Revenue lost to late renewals | $30,000-60,000/year | Near zero |
| Staff hours on renewal tracking | 15 hours/month | 1 hour/month (review only) |

-> [Set up this workflow on Lotics](https://lotics.ai)

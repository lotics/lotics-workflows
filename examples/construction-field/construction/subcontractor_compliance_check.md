# Subcontractor Compliance Check

## The problem

General contractors must verify that every subcontractor on site has current insurance, licenses, and safety certifications before they can work. On a mid-size commercial project with 15-25 subs, the project coordinator spends 6-8 hours per week chasing expired documents. If an uninsured sub is found on site during an audit, the GC faces fines of $5,000-25,000 and potential project shutdown.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Subcontractors | company_name, contact_email, phone, trade | Sub directory |
| Compliance Documents | subcontractor (link), document_type, expiration_date, status, file | Insurance, license, and cert records |
| Projects | project_name, start_date, project_coordinator_email | Active projects |

## Example prompts

- "Every Monday, check all subcontractor compliance documents. If anything expires within 30 days, email the subcontractor asking them to upload a renewed document. If anything is already expired, lock the subcontractor record and notify the project coordinator."
- "Weekly, scan compliance documents for upcoming expirations. Send reminder emails 30 days before expiry and lock out any sub with expired docs."

## Workflow

**Trigger:** Recurring schedule, weekly on Monday at 8:00 AM

```json
[
  {
    "id": "get_all_docs",
    "type": "tool_call",
    "description": "Query all compliance documents",
    "tool_name": "query_records",
    "input": {
      "table_name": "Compliance Documents"
    }
  },
  {
    "id": "check_each_doc",
    "type": "foreach",
    "description": "Evaluate each compliance document for expiration",
    "items": "{{get_all_docs.output.records}}",
    "steps": [
      {
        "id": "get_sub",
        "type": "tool_call",
        "description": "Fetch the subcontractor for this document",
        "tool_name": "get_record",
        "input": {
          "table_name": "Subcontractors",
          "record_id": "{{check_each_doc.item.subcontractor}}"
        }
      },
      {
        "id": "check_expiry",
        "type": "if",
        "description": "Check if document is already expired",
        "condition": "{{new Date(check_each_doc.item.expiration_date) < new Date()}}",
        "then": [
          {
            "id": "mark_expired",
            "type": "tool_call",
            "description": "Update document status to Expired",
            "tool_name": "update_record",
            "input": {
              "table_name": "Compliance Documents",
              "record_id": "{{check_each_doc.item.id}}",
              "data": {
                "status": "Expired"
              }
            }
          },
          {
            "id": "lock_sub",
            "type": "tool_call",
            "description": "Lock the subcontractor record to prevent site access",
            "tool_name": "lock_record",
            "input": {
              "table_name": "Subcontractors",
              "record_id": "{{check_each_doc.item.subcontractor}}"
            }
          },
          {
            "id": "notify_coordinator",
            "type": "tool_call",
            "description": "Alert the project coordinator about the expired document",
            "tool_name": "send_notification",
            "input": {
              "message": "COMPLIANCE ALERT: {{get_sub.output.record.company_name}} has an expired {{check_each_doc.item.document_type}} (expired {{check_each_doc.item.expiration_date}}). Subcontractor has been locked and cannot be assigned to work until renewed.",
              "record_id": "{{check_each_doc.item.subcontractor}}",
              "table_name": "Subcontractors"
            }
          },
          {
            "id": "email_sub_expired",
            "type": "tool_call",
            "description": "Email the subcontractor about the expired document",
            "tool_name": "gmail_send_email",
            "input": {
              "to": "{{get_sub.output.record.contact_email}}",
              "subject": "URGENT: Expired {{check_each_doc.item.document_type}} — Action Required",
              "body": "Hi {{get_sub.output.record.company_name}},\n\nOur records show that your {{check_each_doc.item.document_type}} expired on {{check_each_doc.item.expiration_date}}.\n\nYour company has been temporarily suspended from all active job sites until a current document is provided. Please upload your renewed {{check_each_doc.item.document_type}} as soon as possible to restore access.\n\nThank you."
            }
          }
        ],
        "else": [
          {
            "id": "check_expiring_soon",
            "type": "if",
            "description": "Check if document expires within 30 days",
            "condition": "{{(new Date(check_each_doc.item.expiration_date) - new Date()) < 2592000000}}",
            "then": [
              {
                "id": "email_sub_reminder",
                "type": "tool_call",
                "description": "Send renewal reminder to subcontractor",
                "tool_name": "gmail_send_email",
                "input": {
                  "to": "{{get_sub.output.record.contact_email}}",
                  "subject": "Reminder: {{check_each_doc.item.document_type}} expiring on {{check_each_doc.item.expiration_date}}",
                  "body": "Hi {{get_sub.output.record.company_name}},\n\nThis is a reminder that your {{check_each_doc.item.document_type}} expires on {{check_each_doc.item.expiration_date}}. To avoid any interruption to your work on our projects, please submit your renewed document before the expiration date.\n\nThank you for your prompt attention."
                }
              }
            ],
            "else": []
          }
        ]
      }
    ]
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| Coordinator time on compliance | 6-8 hours/week | Under 1 hour/week |
| Expired documents found on audit | 3-5 per audit | 0 (auto-locked on expiry) |
| Renewal reminders sent | Inconsistent, often late | 30 days before every expiration |

-> [Set up this workflow on Lotics](https://lotics.ai)

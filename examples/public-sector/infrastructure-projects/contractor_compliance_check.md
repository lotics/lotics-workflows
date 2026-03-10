# Contractor Compliance Check

## The problem

A county transportation department manages 35+ active infrastructure contracts at any time. Each contractor must maintain current insurance certificates, bonding, DBE (Disadvantaged Business Enterprise) certifications, and prevailing wage affidavits. Compliance officers manually check expiration dates in shared drives and call contractors when documents lapse. Last fiscal year, 3 contractors worked for an average of 18 days with expired insurance before anyone noticed — creating significant liability exposure for the county.

## Data model

| Table | Key fields | Purpose |
|-------|-----------|---------|
| Contractors | company_name, contact_email, contact_phone, dbe_certified, active | Master list of approved contractors |
| Compliance Documents | contractor_id, document_type, expiration_date, status, file, verified_by | Track insurance, bonding, and certification documents |
| Projects | project_name, contractor_id, status, start_date, end_date | Active infrastructure projects linked to contractors |

## Example prompts

- "Every Monday, check all compliance documents expiring in the next 30 days. Email the contractor to upload renewed documents and flag the project manager if anything is already expired."
- "Set up a weekly compliance sweep that finds contractors with lapsing insurance or bonding, notifies them, and locks their project records if documents are overdue."

## Workflow

**Trigger:** `recurring_schedule` — runs every Monday at 07:00 AM.

```json
[
  {
    "id": "find_expiring_docs",
    "type": "tool_call",
    "description": "Query compliance documents expiring within the next 30 days or already expired",
    "tool_name": "query_records",
    "input": {
      "table_id": "compliance_documents",
      "filters": {
        "status": "Active"
      }
    }
  },
  {
    "id": "check_each_doc",
    "type": "foreach",
    "description": "Evaluate each compliance document for expiration risk",
    "items": "{{find_expiring_docs.output.records}}",
    "steps": [
      {
        "id": "get_contractor",
        "type": "tool_call",
        "description": "Look up the contractor details for this document",
        "tool_name": "get_record",
        "input": {
          "table_id": "contractors",
          "record_id": "{{check_each_doc.item.contractor_id}}"
        }
      },
      {
        "id": "check_expired",
        "type": "if",
        "description": "Check if the document is already expired",
        "condition": "{{new Date(check_each_doc.item.expiration_date) < new Date()}}",
        "then": [
          {
            "id": "mark_expired",
            "type": "tool_call",
            "description": "Update the compliance document status to Expired",
            "tool_name": "update_record",
            "input": {
              "table_id": "compliance_documents",
              "record_id": "{{check_each_doc.item.id}}",
              "data": {
                "status": "Expired"
              }
            }
          },
          {
            "id": "find_affected_projects",
            "type": "tool_call",
            "description": "Find active projects tied to this contractor",
            "tool_name": "query_records",
            "input": {
              "table_id": "projects",
              "filters": {
                "contractor_id": "{{check_each_doc.item.contractor_id}}",
                "status": "Active"
              }
            }
          },
          {
            "id": "lock_projects",
            "type": "foreach",
            "description": "Lock each affected project record to prevent new activity until compliance is restored",
            "items": "{{find_affected_projects.output.records}}",
            "steps": [
              {
                "id": "lock_project",
                "type": "tool_call",
                "description": "Lock the project record",
                "tool_name": "lock_record",
                "input": {
                  "table_id": "projects",
                  "record_id": "{{lock_projects.item.id}}"
                }
              }
            ]
          },
          {
            "id": "urgent_contractor_notice",
            "type": "tool_call",
            "description": "Send urgent email to contractor about expired documents",
            "tool_name": "gmail_send_email",
            "input": {
              "to": "{{get_contractor.output.contact_email}}",
              "subject": "URGENT: Expired {{check_each_doc.item.document_type}} - Project Work Suspended",
              "body": "{{get_contractor.output.company_name}},\n\nYour {{check_each_doc.item.document_type}} expired on {{check_each_doc.item.expiration_date}}. Per county policy, all project activity under your contracts has been suspended until a current document is submitted.\n\nPlease upload your renewed {{check_each_doc.item.document_type}} immediately.\n\nCounty Transportation Department - Compliance Division"
            }
          }
        ],
        "else": [
          {
            "id": "check_expiring_soon",
            "type": "if",
            "description": "Check if the document expires within 30 days",
            "condition": "{{new Date(check_each_doc.item.expiration_date) < new Date(Date.now() + 30 * 86400000)}}",
            "then": [
              {
                "id": "reminder_email",
                "type": "tool_call",
                "description": "Send a courtesy reminder to the contractor about upcoming expiration",
                "tool_name": "gmail_send_email",
                "input": {
                  "to": "{{get_contractor.output.contact_email}}",
                  "subject": "Reminder: {{check_each_doc.item.document_type}} expires {{check_each_doc.item.expiration_date}}",
                  "body": "{{get_contractor.output.company_name}},\n\nThis is a courtesy reminder that your {{check_each_doc.item.document_type}} expires on {{check_each_doc.item.expiration_date}}. Please submit your renewed document before the expiration date to avoid work suspension.\n\nCounty Transportation Department - Compliance Division"
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
|--------|--------|-------|
| Days working with expired insurance | Up to 18 days | 0 (same-day detection) |
| Compliance check staff hours per week | 8 hours | 30 minutes (review exceptions) |
| Contractor advance notice before expiration | Inconsistent | 30 days, every time |
| Liability incidents from lapsed coverage | 3 per year | 0 |

-> [Set up this workflow on Lotics](https://lotics.ai)

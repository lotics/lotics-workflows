# Document Intake Routing

## The problem

Municipal clerks manually sort and route 200+ incoming documents per week — permit applications, FOIA requests, zoning appeals, business license renewals — to the correct department. Misrouted documents add 3-5 days of delay, and 15% of submissions sit unassigned for over a week because the intake desk cannot keep up during peak periods.

## Data model

| Table | Key fields | Purpose |
|-------|-----------|---------|
| Incoming Documents | document_type, submitter_name, submitter_email, department, status, assigned_to, received_date, attachments | Track every document from receipt to resolution |
| Departments | name, intake_email, director, active_staff_count | Route documents to the right team |
| Routing Rules | document_type, target_department, sla_days, priority | Map document categories to departments and deadlines |

## Example prompts

- "When a new document is submitted through the intake form, classify it by type, look up the routing rule, assign it to the correct department, and email the department director."
- "Set up a workflow so every incoming document gets auto-routed based on its type, with a notification to the submitter confirming receipt and expected response time."

## Workflow

**Trigger:** `form_submitted` — a resident or staff member submits a document through the public intake form.

```json
[
  {
    "id": "extract_doc_info",
    "type": "tool_call",
    "description": "Extract structured data from the uploaded document attachments",
    "tool_name": "extract_file_data",
    "input": {
      "record_id": "{{trigger.form_submitted.record_id}}",
      "table_id": "{{trigger.form_submitted.table_id}}"
    }
  },
  {
    "id": "classify_document",
    "type": "tool_call",
    "description": "Use LLM to classify the document type from extracted content",
    "tool_name": "llm_generate_text",
    "input": {
      "prompt": "Classify this government document into exactly one category: permit_application, foia_request, zoning_appeal, business_license_renewal, complaint, or other. Document content: {{extract_doc_info.output.extracted_text}}. Respond with only the category slug."
    }
  },
  {
    "id": "lookup_routing",
    "type": "tool_call",
    "description": "Find the routing rule for this document type",
    "tool_name": "query_records",
    "input": {
      "table_id": "routing_rules",
      "filters": {
        "document_type": "{{classify_document.output.text}}"
      }
    }
  },
  {
    "id": "get_department",
    "type": "tool_call",
    "description": "Look up the target department details",
    "tool_name": "query_records",
    "input": {
      "table_id": "departments",
      "filters": {
        "name": "{{lookup_routing.output.records[0].target_department}}"
      }
    }
  },
  {
    "id": "update_document_record",
    "type": "tool_call",
    "description": "Tag the document with its classification, department, and SLA deadline",
    "tool_name": "update_record",
    "input": {
      "table_id": "{{trigger.form_submitted.table_id}}",
      "record_id": "{{trigger.form_submitted.record_id}}",
      "data": {
        "document_type": "{{classify_document.output.text}}",
        "department": "{{lookup_routing.output.records[0].target_department}}",
        "status": "Routed",
        "priority": "{{lookup_routing.output.records[0].priority}}"
      }
    }
  },
  {
    "id": "notify_department",
    "type": "tool_call",
    "description": "Email the department director about the new intake",
    "tool_name": "gmail_send_email",
    "input": {
      "to": "{{get_department.output.records[0].intake_email}}",
      "subject": "New {{classify_document.output.text}} intake requires assignment",
      "body": "A new {{classify_document.output.text}} document has been routed to your department. Submitter: {{trigger.form_submitted.data.submitter_name}}. SLA: {{lookup_routing.output.records[0].sla_days}} days. Please assign a staff member."
    }
  },
  {
    "id": "confirm_to_submitter",
    "type": "tool_call",
    "description": "Send the submitter a confirmation email with expected timeline",
    "tool_name": "gmail_send_email",
    "input": {
      "to": "{{trigger.form_submitted.data.submitter_email}}",
      "subject": "Your document submission has been received",
      "body": "Thank you, {{trigger.form_submitted.data.submitter_name}}. Your {{classify_document.output.text}} submission has been received and routed to the {{lookup_routing.output.records[0].target_department}} department. Expected response time: {{lookup_routing.output.records[0].sla_days}} business days."
    }
  }
]
```

## Results

| Metric | Before | After |
|--------|--------|-------|
| Average routing time | 2 days | Under 5 minutes |
| Misrouted documents | 15% | Under 2% |
| Submitter confirmation | Manual, inconsistent | Immediate, every submission |
| Weekly clerk hours on sorting | 12 hours | 1 hour (exceptions only) |

-> [Set up this workflow on Lotics](https://lotics.ai)

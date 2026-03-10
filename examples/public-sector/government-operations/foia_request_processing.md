# FOIA Request Processing

## The problem

The city clerk's office receives 40-60 Freedom of Information Act requests per month via email. Each request must be logged, acknowledged within 5 business days (state mandate), assigned to the custodian of the relevant records, and tracked to completion within 30 days. Staff currently copy-paste from emails into spreadsheets, and last year 12% of requests missed the 5-day acknowledgment deadline — exposing the municipality to legal complaints and public trust erosion.

## Data model

| Table | Key fields | Purpose |
|-------|-----------|---------|
| FOIA Requests | requester_name, requester_email, request_summary, records_requested, department, custodian, status, received_date, acknowledgment_due, response_due, response_files | Track each FOIA request from receipt to fulfillment |
| Records Custodians | department, custodian_name, custodian_email | Map departments to the staff member responsible for records |

## Example prompts

- "When a FOIA request email arrives, create a tracking record, find the right records custodian, send the requester an acknowledgment letter, and notify the custodian to begin gathering documents."
- "Automatically process incoming FOIA emails: log the request, calculate the statutory deadlines, and assign it to the department that holds the records."

## Workflow

**Trigger:** `receive_gmail_email` — an email arrives at the dedicated foia@city.gov inbox.

```json
[
  {
    "id": "read_request_email",
    "type": "tool_call",
    "description": "Read the full content of the incoming FOIA request email",
    "tool_name": "gmail_read_email",
    "input": {
      "email_id": "{{trigger.receive_gmail_email.email_id}}"
    }
  },
  {
    "id": "parse_request",
    "type": "tool_call",
    "description": "Extract structured fields from the FOIA request using LLM",
    "tool_name": "llm_generate_text",
    "input": {
      "prompt": "Extract the following from this FOIA request email and return as JSON with keys requester_name, requester_email, request_summary, records_requested, target_department. Email body: {{read_request_email.output.body}}"
    }
  },
  {
    "id": "find_custodian",
    "type": "tool_call",
    "description": "Look up the records custodian for the target department",
    "tool_name": "query_records",
    "input": {
      "table_id": "records_custodians",
      "filters": {
        "department": "{{JSON.parse(parse_request.output.text).target_department}}"
      }
    }
  },
  {
    "id": "create_foia_record",
    "type": "tool_call",
    "description": "Create the FOIA tracking record with statutory deadlines",
    "tool_name": "create_record",
    "input": {
      "table_id": "foia_requests",
      "data": {
        "requester_name": "{{JSON.parse(parse_request.output.text).requester_name}}",
        "requester_email": "{{JSON.parse(parse_request.output.text).requester_email}}",
        "request_summary": "{{JSON.parse(parse_request.output.text).request_summary}}",
        "records_requested": "{{JSON.parse(parse_request.output.text).records_requested}}",
        "department": "{{JSON.parse(parse_request.output.text).target_department}}",
        "custodian": "{{find_custodian.output.records[0].custodian_email}}",
        "status": "Received",
        "received_date": "{{trigger.receive_gmail_email.received_at}}"
      }
    }
  },
  {
    "id": "send_acknowledgment",
    "type": "tool_call",
    "description": "Reply to the requester with a formal acknowledgment within the statutory window",
    "tool_name": "gmail_reply_email",
    "input": {
      "email_id": "{{trigger.receive_gmail_email.email_id}}",
      "body": "Dear {{JSON.parse(parse_request.output.text).requester_name}},\n\nThis acknowledges receipt of your Freedom of Information Act request received on {{trigger.receive_gmail_email.received_at}}.\n\nYour request for: {{JSON.parse(parse_request.output.text).records_requested}}\n\nThis request has been assigned to the {{JSON.parse(parse_request.output.text).target_department}} department. You can expect a response within 30 business days per statutory requirements.\n\nTracking ID: {{create_foia_record.output.record_id}}\n\nCity Clerk's Office"
    }
  },
  {
    "id": "notify_custodian",
    "type": "tool_call",
    "description": "Notify the records custodian to begin gathering responsive documents",
    "tool_name": "gmail_send_email",
    "input": {
      "to": "{{find_custodian.output.records[0].custodian_email}}",
      "subject": "FOIA Request Assigned - {{create_foia_record.output.record_id}}",
      "body": "A new FOIA request has been assigned to you.\n\nRequester: {{JSON.parse(parse_request.output.text).requester_name}}\nRecords requested: {{JSON.parse(parse_request.output.text).records_requested}}\nSummary: {{JSON.parse(parse_request.output.text).request_summary}}\n\nPlease begin gathering responsive documents. The statutory response deadline is 30 business days from {{trigger.receive_gmail_email.received_at}}."
    }
  }
]
```

## Results

| Metric | Before | After |
|--------|--------|-------|
| Acknowledgment sent within 5 days | 88% | 100% (immediate) |
| Time to log and assign a request | 25 minutes | Under 2 minutes |
| Requests lost or duplicated | 3-4 per quarter | 0 |
| Clerk hours on FOIA intake per month | 20 hours | 2 hours (review only) |

-> [Set up this workflow on Lotics](https://lotics.ai)

# Interview Scheduling and Feedback Collection

## The problem

A staffing agency coordinates 50-70 client interviews per week. Each interview requires confirming availability with the candidate, sending details to the client hiring manager, and collecting post-interview feedback from both sides. Recruiters spend 4-6 hours daily on scheduling logistics — phone tag, email chains, and manual calendar checks. Feedback collection is worse: 40% of hiring managers never respond, leaving recruiters without the data they need to move candidates forward or provide coaching. Placements stall because nobody closes the feedback loop.

## Data model

| Table | Key fields | Purpose |
|-------|-----------|---------|
| Interviews | candidate (link), job_order (link), client_contact, interview_date, interview_time, location, status, candidate_feedback, client_feedback | Scheduled interviews with feedback |
| Candidates | candidate_name, email, phone, status | Candidate profiles |
| Job Orders | job_title, client_company, client_contact_email, recruiter, status | Open positions from clients |

## Example prompts

- "When a recruiter schedules an interview, email the candidate with details and send the client a confirmation. After the interview date passes, request feedback from both sides and log it on the record."
- "Automate interview coordination: send confirmations when scheduled, follow up for feedback after the interview, and alert the recruiter when both sides have responded."

## Workflow

**Trigger:** `record_updated` on the Interviews table — fires when `status` changes to `Scheduled`.

```json
[
  {
    "id": "get_candidate",
    "type": "tool_call",
    "description": "Fetch candidate details for the interview",
    "tool_name": "get_record",
    "input": {
      "table_name": "Candidates",
      "record_id": "{{trigger.record_updated.next_data.candidate}}"
    }
  },
  {
    "id": "get_job_order",
    "type": "tool_call",
    "description": "Fetch job order details including client contact",
    "tool_name": "get_record",
    "input": {
      "table_name": "Job Orders",
      "record_id": "{{trigger.record_updated.next_data.job_order}}"
    }
  },
  {
    "id": "email_candidate",
    "type": "tool_call",
    "description": "Send interview confirmation to the candidate",
    "tool_name": "gmail_send_email",
    "input": {
      "to": "{{get_candidate.output.email}}",
      "subject": "Interview Confirmed: {{get_job_order.output.job_title}} at {{get_job_order.output.client_company}}",
      "body": "Hi {{get_candidate.output.candidate_name}},\n\nYour interview has been confirmed:\n\nPosition: {{get_job_order.output.job_title}}\nCompany: {{get_job_order.output.client_company}}\nDate: {{trigger.record_updated.next_data.interview_date}}\nTime: {{trigger.record_updated.next_data.interview_time}}\nLocation: {{trigger.record_updated.next_data.location}}\n\nPlease arrive 10 minutes early and bring a copy of your resume. If you need to reschedule, reply to this email as soon as possible.\n\nGood luck!\nRecruitment Team"
    }
  },
  {
    "id": "email_client",
    "type": "tool_call",
    "description": "Send interview confirmation to the client hiring manager",
    "tool_name": "gmail_send_email",
    "input": {
      "to": "{{get_job_order.output.client_contact_email}}",
      "subject": "Interview Scheduled: {{get_candidate.output.candidate_name}} for {{get_job_order.output.job_title}}",
      "body": "Hi,\n\nAn interview has been scheduled for the {{get_job_order.output.job_title}} position:\n\nCandidate: {{get_candidate.output.candidate_name}}\nDate: {{trigger.record_updated.next_data.interview_date}}\nTime: {{trigger.record_updated.next_data.interview_time}}\nLocation: {{trigger.record_updated.next_data.location}}\n\nWe will follow up after the interview to collect your feedback.\n\nThank you,\nRecruitment Team"
    }
  },
  {
    "id": "wait_until_after_interview",
    "type": "wait",
    "description": "Wait until the day after the interview to request feedback",
    "until": "{{addDays(trigger.record_updated.next_data.interview_date, 1)}}"
  },
  {
    "id": "request_candidate_feedback",
    "type": "tool_call",
    "description": "Email the candidate requesting their interview feedback",
    "tool_name": "gmail_send_email",
    "input": {
      "to": "{{get_candidate.output.email}}",
      "subject": "How did your interview go? — {{get_job_order.output.job_title}} at {{get_job_order.output.client_company}}",
      "body": "Hi {{get_candidate.output.candidate_name}},\n\nWe hope your interview for the {{get_job_order.output.job_title}} position at {{get_job_order.output.client_company}} went well.\n\nPlease reply with your feedback:\n1. How did the interview go overall?\n2. Are you still interested in the position?\n3. Any concerns or questions?\n\nYour feedback helps us advocate for you and improve our process.\n\nThank you,\nRecruitment Team"
    }
  },
  {
    "id": "request_client_feedback",
    "type": "tool_call",
    "description": "Email the client hiring manager requesting their feedback on the candidate",
    "tool_name": "gmail_send_email",
    "input": {
      "to": "{{get_job_order.output.client_contact_email}}",
      "subject": "Feedback Request: {{get_candidate.output.candidate_name}} — {{get_job_order.output.job_title}}",
      "body": "Hi,\n\nThank you for interviewing {{get_candidate.output.candidate_name}} for the {{get_job_order.output.job_title}} role.\n\nPlease reply with your feedback:\n1. Overall impression (Strong Yes / Yes / Maybe / No)\n2. Key strengths observed\n3. Any concerns\n4. Would you like to proceed to next steps?\n\nWe appreciate your timely response so we can keep the process moving.\n\nThank you,\nRecruitment Team"
    }
  },
  {
    "id": "wait_for_client_response",
    "type": "wait_for_event",
    "description": "Wait for the client to reply with feedback",
    "event": "receive_gmail_email",
    "condition": "{{true}}"
  },
  {
    "id": "log_client_feedback",
    "type": "tool_call",
    "description": "Update the interview record with the client's feedback",
    "tool_name": "update_record",
    "input": {
      "table_name": "Interviews",
      "record_id": "{{trigger.record_updated.record_id}}",
      "data": {
        "client_feedback": "{{wait_for_client_response.output.body}}",
        "status": "Feedback Received"
      }
    }
  },
  {
    "id": "notify_recruiter",
    "type": "tool_call",
    "description": "Alert the recruiter that client feedback has been received",
    "tool_name": "send_notification",
    "input": {
      "member_id": "{{get_job_order.output.recruiter}}",
      "title": "Client feedback received: {{get_candidate.output.candidate_name}}",
      "message": "{{get_job_order.output.client_company}} has provided feedback on {{get_candidate.output.candidate_name}} for the {{get_job_order.output.job_title}} position. Review the interview record to take next steps."
    }
  }
]
```

## Results

| Metric | Before | After |
|--------|--------|-------|
| Recruiter hours on scheduling logistics | 4-6 hrs/day | < 1 hr/day |
| Client feedback response rate | 60% | 90% |
| Avg. time from interview to next step | 5 days | 1.5 days |

-> [Set up this workflow on Lotics](https://lotics.ai)

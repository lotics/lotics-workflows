# Post-Event Feedback Collection

## The problem

After an event concludes, the coordinator intends to collect feedback from the client and attendees but it rarely happens. There's no systematic process, so follow-up emails go out 2-3 weeks late or not at all. By that time, the experience is stale and response rates are below 10%. Without structured feedback, the company can't identify which vendors performed well, what went wrong, or build testimonials for sales pitches. Repeat business suffers because clients feel the relationship ends at the event.

## Data model

| Table | Key fields | Purpose |
|-------|-----------|---------|
| Events | event_name, event_date, client_name, client_email, status, coordinator_id, feedback_status | Event records |
| Attendees | event_id, full_name, email, role | Event attendee list |
| Feedback | event_id, respondent_email, rating, comments, submitted_at | Collected feedback responses |

## Example prompts

- "One day after an event ends, email the client and all attendees a feedback request. After 5 days, compile any responses and send a summary to the coordinator."
- "Automate post-event feedback collection: send surveys the day after, then summarize responses for the team."

## Workflow

**Trigger:** When an event record is updated and status becomes "completed"

```json
[
  {
    "id": "wait_one_day",
    "type": "wait",
    "description": "Wait one day after event completion for feedback timing",
    "duration": "P1D"
  },
  {
    "id": "get_attendees",
    "type": "tool_call",
    "description": "Fetch all attendees for the event",
    "tool_name": "query_records",
    "input": {
      "table_name": "Attendees",
      "filters": {
        "event_id": "{{trigger.record_updated.next_data.id}}"
      }
    }
  },
  {
    "id": "email_client",
    "type": "tool_call",
    "description": "Send feedback request to the primary client",
    "tool_name": "gmail_send_email",
    "input": {
      "to": "{{trigger.record_updated.next_data.client_email}}",
      "subject": "How was {{trigger.record_updated.next_data.event_name}}? We'd love your feedback",
      "body": "Hi {{trigger.record_updated.next_data.client_name}},\n\nThank you for trusting us with {{trigger.record_updated.next_data.event_name}}! We hope everything exceeded your expectations.\n\nWe'd greatly appreciate your feedback to help us improve:\n\n1. Overall satisfaction (1-10)?\n2. What went well?\n3. What could we improve?\n4. Would you recommend us to others?\n\nPlease reply to this email with your thoughts. It only takes 2 minutes.\n\nThank you!"
    }
  },
  {
    "id": "email_attendees",
    "type": "foreach",
    "description": "Send feedback request to each attendee",
    "items": "{{get_attendees.output.records}}",
    "steps": [
      {
        "id": "send_attendee_survey",
        "type": "tool_call",
        "description": "Email feedback request to each attendee",
        "tool_name": "gmail_send_email",
        "input": {
          "to": "{{item.email}}",
          "subject": "Quick feedback on {{trigger.record_updated.next_data.event_name}}",
          "body": "Hi {{item.full_name}},\n\nThank you for attending {{trigger.record_updated.next_data.event_name}} on {{trigger.record_updated.next_data.event_date}}.\n\nWe'd love to hear your thoughts:\n- How would you rate the event overall (1-10)?\n- Any highlights or suggestions?\n\nJust reply to this email. Thank you!"
        }
      }
    ]
  },
  {
    "id": "update_feedback_status",
    "type": "tool_call",
    "description": "Mark the event as having feedback requests sent",
    "tool_name": "update_record",
    "input": {
      "table_name": "Events",
      "record_id": "{{trigger.record_updated.next_data.id}}",
      "data": {
        "feedback_status": "surveys_sent"
      }
    }
  },
  {
    "id": "wait_for_responses",
    "type": "wait",
    "description": "Wait 5 days for responses to come in",
    "duration": "P5D"
  },
  {
    "id": "collect_feedback",
    "type": "tool_call",
    "description": "Query all feedback responses received for this event",
    "tool_name": "query_records",
    "input": {
      "table_name": "Feedback",
      "filters": {
        "event_id": "{{trigger.record_updated.next_data.id}}"
      }
    }
  },
  {
    "id": "summarize_feedback",
    "type": "tool_call",
    "description": "Use AI to compile a feedback summary",
    "tool_name": "llm_generate_text",
    "input": {
      "prompt": "Summarize the following event feedback responses for {{trigger.record_updated.next_data.event_name}}. Calculate the average rating, list common themes in positive and negative feedback, and provide 3 actionable recommendations. Feedback data: {{collect_feedback.output.records}}"
    }
  },
  {
    "id": "send_summary",
    "type": "tool_call",
    "description": "Send the compiled feedback summary to the coordinator",
    "tool_name": "send_notification",
    "input": {
      "message": "Feedback Summary for {{trigger.record_updated.next_data.event_name}}:\n\nResponses received: {{collect_feedback.output.records.length}}\n\n{{summarize_feedback.output.text}}",
      "member_id": "{{trigger.record_updated.next_data.coordinator_id}}"
    }
  }
]
```

## Results

| Metric | Before | After |
|--------|--------|-------|
| Events with feedback collected | 30% | 100% |
| Feedback response rate | Under 10% | 35-45% |
| Time from event to feedback summary | 3+ weeks (if at all) | 6 days (automated) |

-> [Set up this workflow on Lotics](https://lotics.ai)

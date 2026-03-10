# Personal Training Session Summary

## The problem

Personal trainers log workout details on paper or in personal notes apps after each session. Clients never receive a written record of what they did, making it hard to track progress or continue exercises independently. When a trainer is absent, the substitute has no context on what the client has been working on. Gyms charging $60-$80/session struggle to justify the premium when clients can't see tangible value beyond the hour itself.

## Data model

| Table | Key fields | Purpose |
|-------|-----------|---------|
| PT Sessions | client_id, trainer_id, session_date, exercises_json, notes, session_number, package_id | Individual training session logs |
| Members | full_name, email, fitness_goals | Client profiles |
| PT Packages | client_id, total_sessions, used_sessions, status | Purchased training packages |

## Example prompts

- "After a trainer submits a PT session form, use AI to generate a friendly session summary from the exercise notes and email it to the client with their remaining sessions count."
- "When a PT session is logged, create a summary email for the client covering exercises performed, trainer notes, and how many sessions they have left."

## Workflow

**Trigger:** Form submitted (PT Session Log form)

```json
[
  {
    "id": "get_member",
    "type": "tool_call",
    "description": "Fetch the client's profile for email and fitness goals",
    "tool_name": "get_record",
    "input": {
      "table_name": "Members",
      "record_id": "{{trigger.form_submitted.data.client_id}}"
    }
  },
  {
    "id": "get_package",
    "type": "tool_call",
    "description": "Fetch the PT package to check remaining sessions",
    "tool_name": "get_record",
    "input": {
      "table_name": "PT Packages",
      "record_id": "{{trigger.form_submitted.data.package_id}}"
    }
  },
  {
    "id": "generate_summary",
    "type": "tool_call",
    "description": "Use AI to create a readable session summary from exercise data",
    "tool_name": "llm_generate_text",
    "input": {
      "prompt": "Write a friendly, concise workout summary email for a personal training client. Their fitness goals are: {{get_member.output.fitness_goals}}. Today's exercises: {{trigger.form_submitted.data.exercises_json}}. Trainer notes: {{trigger.form_submitted.data.notes}}. Session number: {{trigger.form_submitted.data.session_number}}. Keep it encouraging and under 200 words. Do not include a subject line."
    }
  },
  {
    "id": "update_package",
    "type": "tool_call",
    "description": "Increment the used sessions count on the package",
    "tool_name": "update_record",
    "input": {
      "table_name": "PT Packages",
      "record_id": "{{get_package.output.id}}",
      "data": {
        "used_sessions": "{{get_package.output.used_sessions + 1}}"
      }
    }
  },
  {
    "id": "email_summary",
    "type": "tool_call",
    "description": "Send the AI-generated session summary to the client",
    "tool_name": "gmail_send_email",
    "input": {
      "to": "{{get_member.output.email}}",
      "subject": "Your PT Session #{{trigger.form_submitted.data.session_number}} Recap",
      "body": "Hi {{get_member.output.full_name}},\n\n{{generate_summary.output.text}}\n\nSessions used: {{update_package.output.used_sessions}} of {{get_package.output.total_sessions}}\n\nKeep up the great work!"
    }
  }
]
```

## Results

| Metric | Before | After |
|--------|--------|-------|
| Clients receiving session recaps | 0% | 100% |
| Trainer time on post-session admin | 10 min/session | 2 min (form only) |
| Client retention on PT packages | 70% rebook | 85% rebook |

-> [Set up this workflow on Lotics](https://lotics.ai)

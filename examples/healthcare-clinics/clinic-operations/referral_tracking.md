# Specialist Referral Tracking

## The problem

Primary care clinics generate 30-50 referrals per week to specialists, but 40% of referrals are never completed by patients. Coordinators manually track referral status by calling specialist offices, and patients fall through the cracks when no one follows up. Incomplete referrals lead to delayed diagnoses and liability exposure.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Referrals | patient_name, patient_email, referring_provider, specialist, specialty, reason, status, date_created, days_open | Tracks outbound referrals |
| Patients | full_name, email, phone, primary_provider | Patient directory |
| Providers | name, specialty, fax, email | Internal and external provider contacts |

## Example prompts

- "Every Monday morning, check for referrals that have been open for more than 14 days. Email the patient a reminder and notify the care coordinator if any referral has been open for more than 30 days."
- "Build a weekly workflow that finds stale referrals and follows up with patients automatically, escalating old ones to the coordinator."

## Workflow

**Trigger:** `recurring_schedule` — Runs every Monday at 8:00 AM.

```json
[
  {
    "id": "find_stale_referrals",
    "type": "tool_call",
    "description": "Query all referrals that are still open and older than 14 days",
    "tool_name": "query_records",
    "input": {
      "table_name": "Referrals",
      "filters": {
        "status": "Open"
      }
    }
  },
  {
    "id": "process_referrals",
    "type": "foreach",
    "description": "Follow up on each stale referral based on how long it has been open",
    "items": "{{find_stale_referrals.output.records.filter(r => r.days_open > 14)}}",
    "steps": [
      {
        "id": "check_escalation",
        "type": "if",
        "description": "Escalate referrals open longer than 30 days to the care coordinator",
        "condition": "{{item.days_open > 30}}",
        "then": [
          {
            "id": "escalate_to_coordinator",
            "type": "tool_call",
            "description": "Notify care coordinator about a critically overdue referral",
            "tool_name": "send_notification",
            "input": {
              "message": "OVERDUE REFERRAL: {{item.patient_name}} was referred to {{item.specialist}} ({{item.specialty}}) {{item.days_open}} days ago. Reason: {{item.reason}}. This referral needs immediate follow-up.",
              "channel": "care-coordination"
            }
          },
          {
            "id": "mark_escalated",
            "type": "tool_call",
            "description": "Update referral status to escalated",
            "tool_name": "update_record",
            "input": {
              "table_name": "Referrals",
              "record_id": "{{item.id}}",
              "data": {
                "status": "Escalated"
              }
            }
          }
        ],
        "else": [
          {
            "id": "send_patient_reminder",
            "type": "tool_call",
            "description": "Email patient a reminder to schedule their specialist appointment",
            "tool_name": "gmail_send_email",
            "input": {
              "to": "{{item.patient_email}}",
              "subject": "Reminder: Please schedule your {{item.specialty}} appointment",
              "body": "Hi {{item.patient_name}},\n\nOur records show that Dr. {{item.referring_provider}} referred you to {{item.specialist}} for {{item.reason}} on {{item.date_created}}.\n\nWe haven't received confirmation that this appointment has been scheduled. Please call {{item.specialist}}'s office to book your visit.\n\nIf you've already scheduled or completed this appointment, please let us know so we can update our records.\n\nThank you,\nCare Coordination Team"
            }
          }
        ]
      }
    ]
  },
  {
    "id": "generate_weekly_summary",
    "type": "tool_call",
    "description": "Use LLM to generate a summary of this week's referral status",
    "tool_name": "llm_generate_text",
    "input": {
      "prompt": "Summarize the following referral data into a brief weekly report. Total open referrals: {{find_stale_referrals.output.records.length}}. Referrals over 14 days: {{find_stale_referrals.output.records.filter(r => r.days_open > 14).length}}. Referrals over 30 days: {{find_stale_referrals.output.records.filter(r => r.days_open > 30).length}}. Format as a short paragraph suitable for a Monday morning team briefing."
    }
  },
  {
    "id": "post_summary",
    "type": "tool_call",
    "description": "Send the weekly referral summary to the care coordination channel",
    "tool_name": "send_notification",
    "input": {
      "message": "Weekly Referral Report:\n{{generate_weekly_summary.output.text}}",
      "channel": "care-coordination"
    }
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| Referrals completed within 30 days | 60% | 85% |
| Coordinator hours on follow-up calls | 6 hrs/week | 1.5 hrs/week |
| Referrals lost to no follow-up | 12/month | 2/month |
| Average days to referral completion | 34 days | 19 days |

-> [Set up this workflow on Lotics](https://lotics.ai)

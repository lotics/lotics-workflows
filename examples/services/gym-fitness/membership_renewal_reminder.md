# Membership Renewal Reminder

## The problem

A gym with 800 active members loses 12-15 memberships per month simply because renewal emails go out too late or not at all. The front desk runs a spreadsheet export every Monday to check who expires that week, but members whose renewal falls mid-week often get missed. By the time someone notices, the member has already signed up at a competitor. Each lost membership costs $600-$1,200/year in recurring revenue.

## Data model

| Table | Key fields | Purpose |
|-------|-----------|---------|
| Members | full_name, email, phone, membership_type, expiry_date, status, renewal_count | Active gym membership records |
| Membership Types | type_name, monthly_price, annual_price, benefits | Plan catalog |

## Example prompts

- "Every morning at 8 AM, find members whose membership expires in exactly 7 days and send them a renewal reminder email with their plan details and price."
- "Send automated renewal reminders to members 7 days before expiry. Include their membership type and renewal price."

## Workflow

**Trigger:** Recurring schedule, daily at 08:00

```json
[
  {
    "id": "find_expiring",
    "type": "tool_call",
    "description": "Query members expiring in exactly 7 days",
    "tool_name": "query_records",
    "input": {
      "table_name": "Members",
      "filters": {
        "status": "active",
        "expiry_date": "{{trigger.recurring_schedule.scheduled_at_date_plus_7d}}"
      }
    }
  },
  {
    "id": "send_reminders",
    "type": "foreach",
    "description": "Send renewal reminder to each expiring member",
    "items": "{{find_expiring.output.records}}",
    "steps": [
      {
        "id": "get_plan",
        "type": "tool_call",
        "description": "Fetch the member's current membership type details",
        "tool_name": "get_record",
        "input": {
          "table_name": "Membership Types",
          "record_id": "{{item.membership_type_id}}"
        }
      },
      {
        "id": "send_email",
        "type": "tool_call",
        "description": "Email the renewal reminder with plan and pricing info",
        "tool_name": "gmail_send_email",
        "input": {
          "to": "{{item.email}}",
          "subject": "Your {{get_plan.output.type_name}} membership expires in 7 days",
          "body": "Hi {{item.full_name}},\n\nYour {{get_plan.output.type_name}} membership expires on {{item.expiry_date}}.\n\nRenewal pricing:\n- Monthly: ${{get_plan.output.monthly_price}}/month\n- Annual: ${{get_plan.output.annual_price}}/year\n\nBenefits included: {{get_plan.output.benefits}}\n\nVisit the front desk or reply to this email to renew. Don't lose access to your fitness routine!\n\nSee you at the gym."
        }
      }
    ]
  }
]
```

## Results

| Metric | Before | After |
|--------|--------|-------|
| Missed renewal notifications | 12-15/month | 0/month |
| Membership churn from missed renewals | 8% | 2% |
| Staff time on renewal tracking | 3 hrs/week | 0 hrs/week |

-> [Set up this workflow on Lotics](https://lotics.ai)

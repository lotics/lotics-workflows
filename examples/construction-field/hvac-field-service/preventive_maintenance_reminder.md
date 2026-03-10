# Preventive Maintenance Reminder

## The problem

HVAC companies sell annual maintenance contracts covering 2-4 visits per year (spring startup, fall shutdown, filter changes). With 300-800 active maintenance contracts, scheduling these visits is a massive coordination effort. Missed visits erode customer trust and void manufacturer warranties. Companies typically miss 20-25% of scheduled PM visits, leading to $40,000-80,000 in lost contract revenue annually and a 30% higher rate of emergency calls from neglected systems.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Maintenance Contracts | customer (link), system_type, visit_frequency, next_visit_due, last_visit_date, status | Recurring service agreements |
| Customers | company_name, contact_name, email, phone, address | Customer directory |
| Service Tickets | customer (link), issue_type, status, assigned_tech, scheduled_time | Created for each PM visit |

## Example prompts

- "Every Monday, check all maintenance contracts for visits due in the next 14 days. For each one, create a service ticket, email the customer to schedule, and notify the dispatch team."
- "Weekly, scan PM contracts coming due and auto-generate service tickets. Reach out to customers to confirm scheduling and flag any overdue contracts."

## Workflow

**Trigger:** Recurring schedule, weekly on Monday at 7:00 AM

```json
[
  {
    "id": "get_due_contracts",
    "type": "tool_call",
    "description": "Query maintenance contracts with visits due in the next 14 days",
    "tool_name": "query_records",
    "input": {
      "table_name": "Maintenance Contracts",
      "filters": {
        "status": "Active"
      }
    }
  },
  {
    "id": "process_due",
    "type": "foreach",
    "description": "Process each contract due for a visit",
    "items": "{{get_due_contracts.output.records.filter(c => new Date(c.next_visit_due) <= new Date(Date.now() + 1209600000))}}",
    "steps": [
      {
        "id": "get_customer",
        "type": "tool_call",
        "description": "Fetch customer details",
        "tool_name": "get_record",
        "input": {
          "table_name": "Customers",
          "record_id": "{{process_due.item.customer}}"
        }
      },
      {
        "id": "check_overdue",
        "type": "if",
        "description": "Check if the visit is already overdue",
        "condition": "{{new Date(process_due.item.next_visit_due) < new Date()}}",
        "then": [
          {
            "id": "create_urgent_ticket",
            "type": "tool_call",
            "description": "Create an urgent service ticket for the overdue PM visit",
            "tool_name": "create_record",
            "input": {
              "table_name": "Service Tickets",
              "data": {
                "customer": "{{process_due.item.customer}}",
                "issue_type": "Preventive Maintenance — {{process_due.item.system_type}} (OVERDUE)",
                "status": "Urgent",
                "scheduled_time": ""
              }
            }
          },
          {
            "id": "email_overdue_customer",
            "type": "tool_call",
            "description": "Email the customer about the overdue maintenance visit",
            "tool_name": "outlook_send_email",
            "input": {
              "to": "{{get_customer.output.record.email}}",
              "subject": "Overdue Maintenance Visit — {{process_due.item.system_type}} Service",
              "body": "Hi {{get_customer.output.record.contact_name}},\n\nYour scheduled {{process_due.item.system_type}} maintenance visit was due on {{process_due.item.next_visit_due}} and has not yet been completed.\n\nRegular maintenance is important to keep your system running efficiently and maintain your manufacturer warranty coverage. We'd like to schedule this visit as soon as possible.\n\nPlease reply with 2-3 dates and times that work for you, or call us at your convenience.\n\nThank you."
            }
          }
        ],
        "else": [
          {
            "id": "create_pm_ticket",
            "type": "tool_call",
            "description": "Create a standard service ticket for the upcoming PM visit",
            "tool_name": "create_record",
            "input": {
              "table_name": "Service Tickets",
              "data": {
                "customer": "{{process_due.item.customer}}",
                "issue_type": "Preventive Maintenance — {{process_due.item.system_type}}",
                "status": "Scheduling",
                "scheduled_time": ""
              }
            }
          },
          {
            "id": "email_schedule_customer",
            "type": "tool_call",
            "description": "Email the customer to schedule their maintenance visit",
            "tool_name": "outlook_send_email",
            "input": {
              "to": "{{get_customer.output.record.email}}",
              "subject": "Time to Schedule Your {{process_due.item.system_type}} Maintenance",
              "body": "Hi {{get_customer.output.record.contact_name}},\n\nYour {{process_due.item.system_type}} preventive maintenance visit is coming up (due by {{process_due.item.next_visit_due}}).\n\nThis visit includes a full system inspection, filter replacement, and performance check to keep your equipment running at peak efficiency.\n\nPlease reply with 2-3 dates and times that work for you and we'll get you scheduled.\n\nThank you."
            }
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
| Missed PM visits | 20-25% of scheduled visits | Under 3% |
| Lost contract revenue from missed visits | $40,000-80,000/year | Under $5,000/year |
| Emergency calls from neglected systems | 30% higher rate | Baseline rate |

-> [Set up this workflow on Lotics](https://lotics.ai)

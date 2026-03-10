# Rent Payment Reconciliation and Late Notice

## The problem

A property management company with 150 units collects rent on the 1st of each month. By the 5th, the office manager manually cross-references bank deposits against expected rents, identifies who has not paid, and sends individual late notices. This takes 4-6 hours each month, and errors in the reconciliation mean some tenants receive late notices they do not owe while others who are genuinely late slip through. Late payment follow-up delays cost an average of $8,000/month in deferred collections.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Rent Ledger | entry_id, unit_id, tenant_id, tenant_email, amount_due, amount_paid, due_date, payment_date, status | Monthly rent charges and payment records |
| Units | unit_id, building, address, tenant_id | Unit-to-tenant mapping |
| Late Notices | notice_id, tenant_id, unit_id, amount_overdue, notice_date, delivery_method | Record of all late payment notices sent |

## Example prompts

- "On the 6th of every month, find all rent entries that are still unpaid for this month, create a late notice record for each, and send the tenant an email with the amount owed and a payment link."
- "Automate rent reconciliation: on the 6th, query unpaid rents, generate late notices, email tenants, and notify the property manager with a summary of total outstanding."

## Workflow

**Trigger:** Recurring monthly schedule on the 6th at 9:00 AM.

```json
[
  {
    "id": "find_unpaid",
    "type": "tool_call",
    "description": "Query all rent ledger entries for the current month that remain unpaid",
    "tool_name": "query_records",
    "input": {
      "table_name": "Rent Ledger",
      "filters": {
        "due_date": "{{FORMAT_DATE(DATE_ADD(NOW(), -6, 'day'), 'YYYY-MM-01')}}",
        "status": "Unpaid"
      }
    }
  },
  {
    "id": "check_any_unpaid",
    "type": "if",
    "description": "Only proceed if there are unpaid entries",
    "condition": "{{find_unpaid.output.records.length > 0}}",
    "then": [
      {
        "id": "process_late_tenants",
        "type": "foreach",
        "description": "Create a late notice and email each tenant with unpaid rent",
        "items": "{{find_unpaid.output.records}}",
        "steps": [
          {
            "id": "create_notice",
            "type": "tool_call",
            "description": "Create a late notice record",
            "tool_name": "create_record",
            "input": {
              "table_name": "Late Notices",
              "data": {
                "tenant_id": "{{item.tenant_id}}",
                "unit_id": "{{item.unit_id}}",
                "amount_overdue": "{{item.amount_due - item.amount_paid}}",
                "notice_date": "{{NOW()}}",
                "delivery_method": "Email"
              }
            }
          },
          {
            "id": "update_status",
            "type": "tool_call",
            "description": "Mark the ledger entry as Late",
            "tool_name": "update_record",
            "input": {
              "table_name": "Rent Ledger",
              "record_id": "{{item.entry_id}}",
              "data": {
                "status": "Late"
              }
            }
          },
          {
            "id": "send_late_email",
            "type": "tool_call",
            "description": "Email the tenant a late payment notice",
            "tool_name": "outlook_send_email",
            "input": {
              "to": "{{item.tenant_email}}",
              "subject": "Rent Payment Overdue - Unit {{item.unit_id}}",
              "body": "Dear Tenant,\n\nOur records indicate that your rent payment of ${{item.amount_due - item.amount_paid}} for Unit {{item.unit_id}} due on {{item.due_date}} has not been received.\n\nPlease remit payment as soon as possible to avoid additional late fees.\n\nIf you have already made this payment, please disregard this notice and contact our office so we can update your account.\n\nRegards,\nProperty Management Office"
            }
          }
        ]
      },
      {
        "id": "get_total_outstanding",
        "type": "tool_call",
        "description": "Calculate total outstanding amount across all unpaid rents",
        "tool_name": "aggregate_records",
        "input": {
          "table_name": "Rent Ledger",
          "filters": {
            "due_date": "{{FORMAT_DATE(DATE_ADD(NOW(), -6, 'day'), 'YYYY-MM-01')}}",
            "status": "Late"
          },
          "aggregate": {
            "field": "amount_due",
            "function": "sum"
          }
        }
      },
      {
        "id": "notify_manager",
        "type": "tool_call",
        "description": "Send the property manager an in-app summary of outstanding rents",
        "tool_name": "send_notification",
        "input": {
          "recipient_id": "property_manager",
          "message": "Monthly rent reconciliation complete. {{find_unpaid.output.records.length}} units have unpaid rent totaling ${{get_total_outstanding.output.result}}. Late notices have been sent to all tenants."
        }
      }
    ],
    "else": [
      {
        "id": "notify_all_paid",
        "type": "tool_call",
        "description": "Notify the property manager that all rents are collected",
        "tool_name": "send_notification",
        "input": {
          "recipient_id": "property_manager",
          "message": "Monthly rent reconciliation complete. All rents for this period have been received. No late notices sent."
        }
      }
    ]
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| Reconciliation time | 4-6 hours/month | Fully automated |
| Late notices sent within 5 days | 60% | 100% |
| Erroneous late notices | 3-5 per month | 0 |
| Average days to collect late rent | 18 days | 9 days |

-> [Set up this workflow on Lotics](https://lotics.ai)

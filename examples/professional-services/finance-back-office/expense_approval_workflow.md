# Expense Approval Workflow

## The problem

At a 50-person consulting firm, employees submit 80-120 expense reports per month. Each report must be approved by a manager, then by finance if it exceeds $500. The process runs on email forwards and a shared spreadsheet — reports get lost, approvals take 7-10 days, and finance spends 6+ hours a week chasing managers for sign-off. Employees are frustrated by reimbursement delays and lack of visibility.

## Data model

| Table | Key fields | Purpose |
|-------|-----------|---------|
| Expense Reports | submitter, amount, category, description, receipt_file, status, submitted_at, manager, approved_by | Individual expense submissions with receipts |
| Approval Policies | category, threshold, requires_finance_review | Rules governing which expenses need extra approval |
| Reimbursements | expense_report (link), payment_amount, payment_date, payment_method, status | Tracks payment back to the employee |

## Example prompts

- "When an employee submits an expense report, route it to their manager for approval. If it's over $500, also require finance sign-off. Then create a reimbursement record automatically."
- "Set up an expense approval flow where receipts are validated, managers approve, finance reviews large amounts, and reimbursements are queued."

## Workflow

**Trigger:** `form_submitted` — employee submits an expense report via form.

```json
[
  {
    "id": "create_expense",
    "type": "tool_call",
    "description": "Create the expense report record from the form submission",
    "tool_name": "create_record",
    "input": {
      "table_name": "Expense Reports",
      "data": {
        "submitter": "{{trigger.form_submitted.submitted_by}}",
        "amount": "{{trigger.form_submitted.data.amount}}",
        "category": "{{trigger.form_submitted.data.category}}",
        "description": "{{trigger.form_submitted.data.description}}",
        "status": "Pending Manager Approval",
        "submitted_at": "{{now()}}",
        "manager": "{{trigger.form_submitted.data.manager}}"
      }
    }
  },
  {
    "id": "attach_receipt",
    "type": "tool_call",
    "description": "Attach the uploaded receipt file to the expense record",
    "tool_name": "add_files_to_record",
    "input": {
      "table_name": "Expense Reports",
      "record_id": "{{create_expense.output.record_id}}",
      "field_name": "receipt_file",
      "file_ids": "{{trigger.form_submitted.data.receipt_file_ids}}"
    }
  },
  {
    "id": "notify_manager",
    "type": "tool_call",
    "description": "Send approval notification to the submitter's manager",
    "tool_name": "send_notification",
    "input": {
      "member_id": "{{trigger.form_submitted.data.manager}}",
      "title": "Expense approval needed — ${{trigger.form_submitted.data.amount}}",
      "message": "{{trigger.form_submitted.submitted_by_name}} submitted a ${{trigger.form_submitted.data.amount}} expense for {{trigger.form_submitted.data.category}}. Please review and approve or reject."
    }
  },
  {
    "id": "wait_for_manager",
    "type": "wait_for_event",
    "description": "Wait for the manager to update the expense status",
    "event": "record_updated",
    "condition": "{{trigger.record_updated.next_data.status == 'Manager Approved' || trigger.record_updated.next_data.status == 'Rejected'}}"
  },
  {
    "id": "check_manager_decision",
    "type": "if",
    "description": "Check if the manager approved or rejected",
    "condition": "{{wait_for_manager.output.next_data.status == 'Manager Approved'}}",
    "then": [
      {
        "id": "check_threshold",
        "type": "if",
        "description": "Check if the amount exceeds $500 and needs finance review",
        "condition": "{{trigger.form_submitted.data.amount > 500}}",
        "then": [
          {
            "id": "update_status_finance",
            "type": "tool_call",
            "description": "Update status to pending finance review",
            "tool_name": "update_record",
            "input": {
              "table_name": "Expense Reports",
              "record_id": "{{create_expense.output.record_id}}",
              "data": {
                "status": "Pending Finance Review"
              }
            }
          },
          {
            "id": "notify_finance",
            "type": "tool_call",
            "description": "Notify the finance team about the high-value expense",
            "tool_name": "gmail_send_email",
            "input": {
              "to": "finance@company.com",
              "subject": "Finance Review Required: ${{trigger.form_submitted.data.amount}} expense from {{trigger.form_submitted.submitted_by_name}}",
              "body": "A manager-approved expense of ${{trigger.form_submitted.data.amount}} ({{trigger.form_submitted.data.category}}) requires finance review.\n\nDescription: {{trigger.form_submitted.data.description}}\n\nPlease review in Lotics and approve or reject."
            }
          },
          {
            "id": "wait_for_finance",
            "type": "wait_for_event",
            "description": "Wait for finance to approve or reject",
            "event": "record_updated",
            "condition": "{{trigger.record_updated.next_data.status == 'Approved' || trigger.record_updated.next_data.status == 'Rejected'}}"
          }
        ],
        "else": [
          {
            "id": "auto_approve",
            "type": "tool_call",
            "description": "Auto-approve expenses under threshold after manager sign-off",
            "tool_name": "update_record",
            "input": {
              "table_name": "Expense Reports",
              "record_id": "{{create_expense.output.record_id}}",
              "data": {
                "status": "Approved"
              }
            }
          }
        ]
      },
      {
        "id": "create_reimbursement",
        "type": "tool_call",
        "description": "Create a reimbursement record for the approved expense",
        "tool_name": "create_record",
        "input": {
          "table_name": "Reimbursements",
          "data": {
            "expense_report": "{{create_expense.output.record_id}}",
            "payment_amount": "{{trigger.form_submitted.data.amount}}",
            "payment_method": "Bank Transfer",
            "status": "Queued"
          }
        }
      }
    ],
    "else": [
      {
        "id": "notify_rejection",
        "type": "tool_call",
        "description": "Notify the employee their expense was rejected",
        "tool_name": "send_notification",
        "input": {
          "member_id": "{{trigger.form_submitted.submitted_by}}",
          "title": "Expense rejected",
          "message": "Your ${{trigger.form_submitted.data.amount}} expense for {{trigger.form_submitted.data.category}} was rejected by your manager. Please check the record for details."
        }
      }
    ]
  }
]
```

## Results

| Metric | Before | After |
|--------|--------|-------|
| Avg. approval turnaround | 7-10 days | 1-2 days |
| Finance hours on expense chasing | 6+ hrs/week | < 1 hr/week |
| Lost or duplicate expense reports | 5-8/month | 0 |

-> [Set up this workflow on Lotics](https://lotics.ai)

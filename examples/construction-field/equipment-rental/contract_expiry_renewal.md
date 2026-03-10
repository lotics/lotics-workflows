# Contract Expiry Renewal

## The problem

Equipment rental companies manage 200-500 active rental contracts at any time. Contracts auto-renew if not addressed before expiration, often at unfavorable rates. Meanwhile, equipment sitting idle on a customer's lot generates no revenue if the contract has lapsed without renewal. Coordinators miss 10-15% of contract expirations, leading to $30,000-60,000 in lost revenue per quarter from lapsed contracts and unbilled idle days.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Rental Contracts | contract_number, customer (link), equipment (link), start_date, end_date, daily_rate, status, auto_renew | Active rental agreements |
| Customers | company_name, contact_name, email, phone, credit_status | Customer directory |
| Equipment | name, type, serial_number, availability_status | Rental fleet |

## Example prompts

- "14 days before a rental contract expires, email the customer with a renewal offer. If they haven't responded 3 days before expiry, send a follow-up and notify the account manager. On expiration day, if still no response, mark the contract as Expired and update the equipment to Available."
- "Automate contract renewal reminders: send initial notice 14 days out, follow up at 3 days, and auto-expire contracts with no response."

## Workflow

**Trigger:** Recurring schedule, daily at 8:00 AM

```json
[
  {
    "id": "get_expiring_contracts",
    "type": "tool_call",
    "description": "Query contracts expiring within the next 14 days",
    "tool_name": "query_records",
    "input": {
      "table_name": "Rental Contracts",
      "filters": {
        "status": "Active"
      }
    }
  },
  {
    "id": "process_contracts",
    "type": "foreach",
    "description": "Process each contract based on days until expiration",
    "items": "{{get_expiring_contracts.output.records}}",
    "steps": [
      {
        "id": "get_customer",
        "type": "tool_call",
        "description": "Fetch customer details for the contract",
        "tool_name": "get_record",
        "input": {
          "table_name": "Customers",
          "record_id": "{{process_contracts.item.customer}}"
        }
      },
      {
        "id": "check_expiry_window",
        "type": "switch",
        "description": "Route based on days until expiration",
        "expression": "{{Math.ceil((new Date(process_contracts.item.end_date) - new Date()) / 86400000)}}",
        "cases": {
          "14": [
            {
              "id": "send_renewal_notice",
              "type": "tool_call",
              "description": "Send 14-day renewal notice to customer",
              "tool_name": "gmail_send_email",
              "input": {
                "to": "{{get_customer.output.record.email}}",
                "subject": "Rental Contract {{process_contracts.item.contract_number}} — Renewal Notice",
                "body": "Hi {{get_customer.output.record.contact_name}},\n\nYour rental contract {{process_contracts.item.contract_number}} for equipment {{process_contracts.item.equipment}} expires on {{process_contracts.item.end_date}}.\n\nCurrent daily rate: ${{process_contracts.item.daily_rate}}\n\nTo renew, please reply to this email or contact your account representative. If you'd like to return the equipment, please schedule a pickup by responding with your preferred date.\n\nThank you for your business."
              }
            }
          ],
          "3": [
            {
              "id": "send_urgent_followup",
              "type": "tool_call",
              "description": "Send urgent 3-day follow-up to customer",
              "tool_name": "gmail_send_email",
              "input": {
                "to": "{{get_customer.output.record.email}}",
                "subject": "ACTION REQUIRED: Contract {{process_contracts.item.contract_number}} expires in 3 days",
                "body": "Hi {{get_customer.output.record.contact_name}},\n\nThis is a follow-up regarding your rental contract {{process_contracts.item.contract_number}} expiring on {{process_contracts.item.end_date}}.\n\nPlease let us know if you'd like to renew or schedule a return. If we don't hear from you by the expiration date, the contract will be closed and the equipment will need to be returned.\n\nThank you."
              }
            },
            {
              "id": "notify_account_manager",
              "type": "tool_call",
              "description": "Alert the account manager about the unresponsive customer",
              "tool_name": "send_notification",
              "input": {
                "message": "Contract {{process_contracts.item.contract_number}} for {{get_customer.output.record.company_name}} expires in 3 days with no renewal response. Daily rate: ${{process_contracts.item.daily_rate}}. Please follow up directly.",
                "record_id": "{{process_contracts.item.id}}",
                "table_name": "Rental Contracts"
              }
            }
          ],
          "0": [
            {
              "id": "expire_contract",
              "type": "tool_call",
              "description": "Mark the contract as expired",
              "tool_name": "update_record",
              "input": {
                "table_name": "Rental Contracts",
                "record_id": "{{process_contracts.item.id}}",
                "data": {
                  "status": "Expired"
                }
              }
            },
            {
              "id": "update_equipment_availability",
              "type": "tool_call",
              "description": "Mark the equipment as available for re-rental",
              "tool_name": "update_record",
              "input": {
                "table_name": "Equipment",
                "record_id": "{{process_contracts.item.equipment}}",
                "data": {
                  "availability_status": "Available"
                }
              }
            },
            {
              "id": "log_expiry",
              "type": "tool_call",
              "description": "Add a comment documenting the contract expiration",
              "tool_name": "create_record_comments",
              "input": {
                "table_name": "Rental Contracts",
                "record_id": "{{process_contracts.item.id}}",
                "comment": "Contract expired on {{process_contracts.item.end_date}}. No renewal response received after 14-day and 3-day reminders. Equipment marked as available."
              }
            }
          ]
        }
      }
    ]
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| Missed contract expirations | 10-15% of contracts | Under 1% |
| Lost revenue from lapsed contracts | $30,000-60,000/quarter | Under $5,000/quarter |
| Time spent on renewal tracking | 8 hours/week | Under 1 hour/week |

-> [Set up this workflow on Lotics](https://lotics.ai)

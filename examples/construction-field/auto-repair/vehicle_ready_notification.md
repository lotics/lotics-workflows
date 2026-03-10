# Vehicle Ready Notification

## The problem

When a repair is finished, the technician marks the job complete and moves to the next vehicle. The service advisor must notice the status change, review the final invoice, and call the customer. During busy days with 15-20 completions, advisors fall behind -- customers aren't notified for 2-4 hours, vehicles sit in finished bays blocking new work, and customers call repeatedly asking "is my car done yet?" This ties up phone lines and creates a poor experience. Shops that notify customers within 15 minutes of completion see 25% higher review scores.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Repair Orders | vehicle (link), customer (link), status, assigned_tech (link), diagnosis, final_cost, labor_hours, parts_cost | Repair job records |
| Vehicles | make, model, year, license_plate | Vehicle details |
| Customers | name, phone, email, preferred_contact_method | Customer contacts |

## Example prompts

- "When a repair order status changes to Complete, generate the final invoice PDF, email the customer that their vehicle is ready with the total cost and payment instructions, and notify the front desk."
- "Instantly notify customers when their car is done: send an email with the final cost breakdown and mark the vehicle as ready for pickup."

## Workflow

**Trigger:** When a Repair Orders record is updated

```json
[
  {
    "id": "check_completion",
    "type": "if",
    "description": "Only fire when status changes to Complete",
    "condition": "{{trigger.record_updated.next_data.status === 'Complete' && trigger.record_updated.prev_data.status !== 'Complete'}}",
    "then": [
      {
        "id": "get_vehicle",
        "type": "tool_call",
        "description": "Fetch vehicle details for the notification",
        "tool_name": "get_record",
        "input": {
          "table_name": "Vehicles",
          "record_id": "{{trigger.record_updated.next_data.vehicle}}"
        }
      },
      {
        "id": "get_customer",
        "type": "tool_call",
        "description": "Fetch customer contact info",
        "tool_name": "get_record",
        "input": {
          "table_name": "Customers",
          "record_id": "{{trigger.record_updated.next_data.customer}}"
        }
      },
      {
        "id": "generate_invoice",
        "type": "tool_call",
        "description": "Generate a PDF invoice for the completed repair",
        "tool_name": "generate_pdf_from_html_template",
        "input": {
          "template_name": "repair_invoice",
          "data": {
            "repair_order_id": "{{trigger.record_updated.record_id}}",
            "vehicle": "{{get_vehicle.output.record.year}} {{get_vehicle.output.record.make}} {{get_vehicle.output.record.model}}",
            "license_plate": "{{get_vehicle.output.record.license_plate}}",
            "diagnosis": "{{trigger.record_updated.next_data.diagnosis}}",
            "labor_hours": "{{trigger.record_updated.next_data.labor_hours}}",
            "parts_cost": "{{trigger.record_updated.next_data.parts_cost}}",
            "final_cost": "{{trigger.record_updated.next_data.final_cost}}",
            "date": "{{new Date().toISOString().split('T')[0]}}"
          }
        }
      },
      {
        "id": "attach_invoice",
        "type": "tool_call",
        "description": "Attach the invoice PDF to the repair order",
        "tool_name": "add_files_to_record",
        "input": {
          "table_name": "Repair Orders",
          "record_id": "{{trigger.record_updated.record_id}}",
          "files": ["{{generate_invoice.output.file_url}}"]
        }
      },
      {
        "id": "update_status_ready",
        "type": "tool_call",
        "description": "Update repair order to Ready for Pickup",
        "tool_name": "update_record",
        "input": {
          "table_name": "Repair Orders",
          "record_id": "{{trigger.record_updated.record_id}}",
          "data": {
            "status": "Ready for Pickup"
          }
        }
      },
      {
        "id": "email_customer",
        "type": "tool_call",
        "description": "Email the customer that their vehicle is ready",
        "tool_name": "gmail_send_email",
        "input": {
          "to": "{{get_customer.output.record.email}}",
          "subject": "Your {{get_vehicle.output.record.year}} {{get_vehicle.output.record.make}} {{get_vehicle.output.record.model}} is Ready for Pickup",
          "body": "Hi {{get_customer.output.record.name}},\n\nGreat news — your {{get_vehicle.output.record.year}} {{get_vehicle.output.record.make}} {{get_vehicle.output.record.model}} ({{get_vehicle.output.record.license_plate}}) is ready for pickup.\n\nWork performed:\n{{trigger.record_updated.next_data.diagnosis}}\n\nTotal: ${{trigger.record_updated.next_data.final_cost}}\n\nWe accept cash, check, and all major credit cards. Your detailed invoice is attached.\n\nOur hours today are 7 AM — 6 PM. Please bring your repair order number ({{trigger.record_updated.record_id}}) when you arrive.\n\nThank you for your business.",
          "attachments": ["{{generate_invoice.output.file_url}}"]
        }
      },
      {
        "id": "notify_front_desk",
        "type": "tool_call",
        "description": "Notify front desk that the vehicle is ready for customer pickup",
        "tool_name": "send_notification",
        "input": {
          "message": "Vehicle ready: {{get_vehicle.output.record.year}} {{get_vehicle.output.record.make}} {{get_vehicle.output.record.model}} ({{get_vehicle.output.record.license_plate}}) — {{get_customer.output.record.name}}. Total: ${{trigger.record_updated.next_data.final_cost}}. Customer has been emailed.",
          "record_id": "{{trigger.record_updated.record_id}}",
          "table_name": "Repair Orders"
        }
      }
    ],
    "else": []
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| Customer notification time | 2-4 hours after completion | Under 1 minute |
| "Is my car done?" phone calls | 10-15/day | 1-2/day |
| Bay turnaround (completion to next vehicle) | 2-4 hours | Under 30 minutes |

-> [Set up this workflow on Lotics](https://lotics.ai)

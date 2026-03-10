# Repair Order Intake

## The problem

Auto repair shops receive vehicles with customer complaints ranging from "it makes a weird noise" to detailed descriptions of symptoms. Service advisors spend 10-15 minutes per vehicle creating a repair order, looking up vehicle history, checking warranty status, and generating a preliminary estimate. With 15-25 vehicles per day, intake alone consumes 3-5 hours of service advisor time. Incomplete intake leads to missed upsell opportunities -- 30% of vehicles have overdue maintenance that isn't flagged during check-in.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Vehicles | vin, make, model, year, mileage, customer (link) | Vehicle directory |
| Customers | name, phone, email, preferred_contact_method | Customer contacts |
| Repair Orders | vehicle (link), customer (link), complaint, diagnosis, estimated_cost, status, assigned_tech (link) | Active repair jobs |
| Service History | vehicle (link), service_type, date, mileage_at_service, cost | Past service records |

## Example prompts

- "When a new repair order form is submitted, look up the vehicle's service history, check for any overdue maintenance based on mileage, generate a preliminary diagnosis using AI, and email the customer a summary with the estimated timeline."
- "On vehicle check-in, auto-pull service history, flag overdue maintenance items, create the repair order with AI-assisted diagnosis notes, and send the customer a confirmation."

## Workflow

**Trigger:** When the intake form is submitted on the Repair Orders table

```json
[
  {
    "id": "get_vehicle",
    "type": "tool_call",
    "description": "Fetch the vehicle record for make, model, and mileage",
    "tool_name": "get_record",
    "input": {
      "table_name": "Vehicles",
      "record_id": "{{trigger.form_submitted.data.vehicle}}"
    }
  },
  {
    "id": "get_customer",
    "type": "tool_call",
    "description": "Fetch customer contact details",
    "tool_name": "get_record",
    "input": {
      "table_name": "Customers",
      "record_id": "{{trigger.form_submitted.data.customer}}"
    }
  },
  {
    "id": "get_service_history",
    "type": "tool_call",
    "description": "Pull the vehicle's complete service history",
    "tool_name": "query_records",
    "input": {
      "table_name": "Service History",
      "filters": {
        "vehicle": "{{trigger.form_submitted.data.vehicle}}"
      }
    }
  },
  {
    "id": "analyze_vehicle",
    "type": "tool_call",
    "description": "Use AI to generate preliminary diagnosis and flag overdue maintenance",
    "tool_name": "llm_generate_text",
    "input": {
      "prompt": "You are an experienced auto repair service advisor. Based on the following information, provide:\n1. A preliminary diagnosis of the customer complaint (2-3 likely causes)\n2. Recommended inspection areas\n3. Any overdue maintenance items based on mileage and service history\n\nVehicle: {{get_vehicle.output.record.year}} {{get_vehicle.output.record.make}} {{get_vehicle.output.record.model}}\nCurrent mileage: {{get_vehicle.output.record.mileage}}\nCustomer complaint: {{trigger.form_submitted.data.complaint}}\n\nService history (most recent first):\n{{JSON.stringify(get_service_history.output.records.map(s => ({service: s.service_type, date: s.date, mileage: s.mileage_at_service})))}}\n\nCommon maintenance intervals: oil change every 5,000 miles, brake inspection every 15,000 miles, transmission service every 30,000 miles, coolant flush every 30,000 miles, spark plugs every 60,000 miles, timing belt every 90,000 miles."
    }
  },
  {
    "id": "update_repair_order",
    "type": "tool_call",
    "description": "Update the repair order with AI-generated diagnosis notes",
    "tool_name": "update_record",
    "input": {
      "table_name": "Repair Orders",
      "record_id": "{{trigger.form_submitted.record_id}}",
      "data": {
        "diagnosis": "{{analyze_vehicle.output.text}}",
        "status": "Pending Inspection"
      }
    }
  },
  {
    "id": "email_customer",
    "type": "tool_call",
    "description": "Send the customer a check-in confirmation with preliminary info",
    "tool_name": "gmail_send_email",
    "input": {
      "to": "{{get_customer.output.record.email}}",
      "subject": "Vehicle Check-In Confirmation — {{get_vehicle.output.record.year}} {{get_vehicle.output.record.make}} {{get_vehicle.output.record.model}}",
      "body": "Hi {{get_customer.output.record.name}},\n\nYour {{get_vehicle.output.record.year}} {{get_vehicle.output.record.make}} {{get_vehicle.output.record.model}} has been checked in for service.\n\nYour concern: {{trigger.form_submitted.data.complaint}}\n\nOur technician will perform a thorough inspection and we'll contact you with findings and a detailed estimate before proceeding with any repairs.\n\nRepair order #{{trigger.form_submitted.record_id}}\n\nWe'll keep you updated. Thank you for choosing us."
    }
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| Intake time per vehicle | 10-15 minutes | 2 minutes (form submission) |
| Overdue maintenance items flagged | 70% missed | 100% flagged automatically |
| Customer confirmation time | End of day or next morning | Immediate |

-> [Set up this workflow on Lotics](https://lotics.ai)

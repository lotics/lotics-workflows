# Order Status Update Notification

## The problem

A laundry and dry cleaning shop processes 80-120 orders daily across wash, dry clean, and press services. Customers call 10-15 times per day asking "Is my order ready?" because they have no visibility into processing status. Each call takes 2-3 minutes and interrupts the workflow of front-counter staff. Customers who don't call sometimes show up too early, wait, and leave frustrated.

## Data model

| Table | Key fields | Purpose |
|-------|-----------|---------|
| Orders | customer_id, order_number, items_description, service_type, status, received_at, promised_at, total_price | Individual laundry/dry cleaning orders |
| Customers | full_name, phone, email, address | Customer contact information |

## Example prompts

- "When an order status changes to 'ready_for_pickup', email the customer that their order is ready with the order number and total price."
- "Notify customers automatically whenever their laundry order moves to a new stage: processing, ready for pickup, or delivered."

## Workflow

**Trigger:** When an order record is updated (status field changes)

```json
[
  {
    "id": "get_customer",
    "type": "tool_call",
    "description": "Fetch customer contact details",
    "tool_name": "get_record",
    "input": {
      "table_name": "Customers",
      "record_id": "{{trigger.record_updated.next_data.customer_id}}"
    }
  },
  {
    "id": "route_notification",
    "type": "switch",
    "description": "Send different notifications based on the new order status",
    "expression": "{{trigger.record_updated.next_data.status}}",
    "cases": {
      "processing": [
        {
          "id": "notify_processing",
          "type": "tool_call",
          "description": "Email customer that their order is being processed",
          "tool_name": "gmail_send_email",
          "input": {
            "to": "{{get_customer.output.email}}",
            "subject": "Order #{{trigger.record_updated.next_data.order_number}} is being processed",
            "body": "Hi {{get_customer.output.full_name}},\n\nYour order #{{trigger.record_updated.next_data.order_number}} ({{trigger.record_updated.next_data.service_type}}) is now being processed.\n\nItems: {{trigger.record_updated.next_data.items_description}}\nEstimated ready: {{trigger.record_updated.next_data.promised_at}}\n\nWe'll notify you as soon as it's ready for pickup."
          }
        }
      ],
      "ready_for_pickup": [
        {
          "id": "notify_ready",
          "type": "tool_call",
          "description": "Email customer that their order is ready",
          "tool_name": "gmail_send_email",
          "input": {
            "to": "{{get_customer.output.email}}",
            "subject": "Order #{{trigger.record_updated.next_data.order_number}} is ready for pickup!",
            "body": "Hi {{get_customer.output.full_name}},\n\nYour order is ready!\n\nOrder #: {{trigger.record_updated.next_data.order_number}}\nItems: {{trigger.record_updated.next_data.items_description}}\nTotal: ${{trigger.record_updated.next_data.total_price}}\n\nPickup hours: Mon-Sat 8AM-8PM, Sun 9AM-5PM\n\nPlease pick up within 7 days. Thank you for choosing us!"
          }
        }
      ]
    }
  }
]
```

## Results

| Metric | Before | After |
|--------|--------|-------|
| "Is my order ready?" calls per day | 10-15 | 1-2 |
| Staff time on status inquiries | 30-45 min/day | 5 min/day |
| Customer satisfaction (pickup experience) | Frequent complaints | Positive feedback on notifications |

-> [Set up this workflow on Lotics](https://lotics.ai)

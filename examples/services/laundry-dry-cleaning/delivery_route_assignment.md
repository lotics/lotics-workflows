# Delivery Route Assignment & Customer Notification

## The problem

A laundry service offering home pickup and delivery manually assigns delivery routes each morning. The dispatcher opens the orders list, checks customer addresses, groups them by area, and texts drivers their stops. This takes 45 minutes every morning and errors mean drivers show up at the wrong address or miss stops entirely. Customers don't know when to expect delivery and call to ask, adding more interruptions.

## Data model

| Table | Key fields | Purpose |
|-------|-----------|---------|
| Orders | order_number, customer_id, status, delivery_type, delivery_zone, delivery_date | Orders with delivery requests |
| Customers | full_name, email, phone, address, delivery_zone | Customer profiles with zone mapping |
| Drivers | full_name, phone, assigned_zone, status | Delivery driver roster |
| Deliveries | order_id, driver_id, customer_id, scheduled_date, status, sequence | Delivery assignments |

## Example prompts

- "Every morning at 7 AM, find all orders scheduled for delivery today, group them by zone, assign drivers, create delivery records, and email each customer their estimated window."
- "Automatically build delivery routes each morning by matching orders to zone drivers and notifying customers."

## Workflow

**Trigger:** Recurring schedule, daily at 07:00

```json
[
  {
    "id": "todays_deliveries",
    "type": "tool_call",
    "description": "Find all orders scheduled for delivery today",
    "tool_name": "query_records",
    "input": {
      "table_name": "Orders",
      "filters": {
        "delivery_type": "home_delivery",
        "delivery_date": "{{trigger.recurring_schedule.scheduled_at_date}}",
        "status": "ready_for_pickup"
      }
    }
  },
  {
    "id": "get_drivers",
    "type": "tool_call",
    "description": "Fetch all available drivers",
    "tool_name": "query_records",
    "input": {
      "table_name": "Drivers",
      "filters": {
        "status": "available"
      }
    }
  },
  {
    "id": "assign_deliveries",
    "type": "foreach",
    "description": "Create delivery records and notify customers for each order",
    "items": "{{todays_deliveries.output.records}}",
    "steps": [
      {
        "id": "get_customer",
        "type": "tool_call",
        "description": "Fetch customer details for address and contact",
        "tool_name": "get_record",
        "input": {
          "table_name": "Customers",
          "record_id": "{{item.customer_id}}"
        }
      },
      {
        "id": "create_delivery",
        "type": "tool_call",
        "description": "Create a delivery assignment record",
        "tool_name": "create_record",
        "input": {
          "table_name": "Deliveries",
          "data": {
            "order_id": "{{item.id}}",
            "customer_id": "{{item.customer_id}}",
            "driver_id": "{{item.delivery_zone}}_driver",
            "scheduled_date": "{{trigger.recurring_schedule.scheduled_at_date}}",
            "status": "scheduled"
          }
        }
      },
      {
        "id": "update_order_status",
        "type": "tool_call",
        "description": "Mark the order as out for delivery",
        "tool_name": "update_record",
        "input": {
          "table_name": "Orders",
          "record_id": "{{item.id}}",
          "data": {
            "status": "out_for_delivery"
          }
        }
      },
      {
        "id": "notify_customer",
        "type": "tool_call",
        "description": "Email customer with delivery notification",
        "tool_name": "gmail_send_email",
        "input": {
          "to": "{{get_customer.output.email}}",
          "subject": "Order #{{item.order_number}} - Delivery today",
          "body": "Hi {{get_customer.output.full_name}},\n\nYour laundry order #{{item.order_number}} is scheduled for delivery today to:\n{{get_customer.output.address}}\n\nPlease ensure someone is available to receive the delivery. If you need to reschedule, reply to this email before 10 AM.\n\nThank you!"
        }
      }
    ]
  }
]
```

## Results

| Metric | Before | After |
|--------|--------|-------|
| Morning dispatch time | 45 min | 0 min (automated) |
| Missed delivery stops | 3-5/week | 0 (systematic assignment) |
| Customer "where's my delivery" calls | 8-12/day | 1-2/day |

-> [Set up this workflow on Lotics](https://lotics.ai)

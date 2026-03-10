# Delivery Schedule Notification

## The problem

Event rental companies coordinate deliveries and pickups across 10-25 events per week. Drivers need load lists, site addresses, contact info, and timing. Customers need delivery windows and setup instructions. Coordinators spend 2-3 hours daily building delivery schedules, calling customers to confirm, and briefing drivers. Last-minute changes (venue access times, gate codes, layout modifications) get lost in text threads, causing failed deliveries and setup delays at 15-20% of events.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Rental Orders | customer (link), event_date, delivery_date, delivery_time_window, site_address, access_notes, setup_instructions, status | Confirmed rental orders |
| Order Items | rental_order (link), equipment (link), quantity_confirmed | Items to deliver |
| Customers | company_name, contact_name, phone, email | Customer contacts |
| Delivery Routes | route_date, driver_name, driver_email, stops | Daily delivery schedules |

## Example prompts

- "Every evening at 6 PM, compile tomorrow's deliveries into a route schedule grouped by driver. Email each driver their load list with addresses, customer contacts, and access notes. Also email each customer their delivery window and what to expect."
- "Night before each delivery day, auto-generate driver route sheets and customer delivery confirmations with all the details they need."

## Workflow

**Trigger:** Recurring schedule, daily at 6:00 PM

```json
[
  {
    "id": "get_tomorrow_orders",
    "type": "tool_call",
    "description": "Query all confirmed orders with delivery scheduled for tomorrow",
    "tool_name": "query_records",
    "input": {
      "table_name": "Rental Orders",
      "filters": {
        "delivery_date": "{{new Date(Date.now() + 86400000).toISOString().split('T')[0]}}",
        "status": "Confirmed"
      }
    }
  },
  {
    "id": "notify_customers",
    "type": "foreach",
    "description": "Send delivery confirmation to each customer",
    "items": "{{get_tomorrow_orders.output.records}}",
    "steps": [
      {
        "id": "get_customer",
        "type": "tool_call",
        "description": "Fetch customer contact info",
        "tool_name": "get_record",
        "input": {
          "table_name": "Customers",
          "record_id": "{{notify_customers.item.customer}}"
        }
      },
      {
        "id": "get_items",
        "type": "tool_call",
        "description": "Get the items being delivered for this order",
        "tool_name": "query_records",
        "input": {
          "table_name": "Order Items",
          "filters": {
            "rental_order": "{{notify_customers.item.id}}"
          }
        }
      },
      {
        "id": "email_customer",
        "type": "tool_call",
        "description": "Email the customer their delivery confirmation with details",
        "tool_name": "gmail_send_email",
        "input": {
          "to": "{{get_customer.output.record.email}}",
          "subject": "Delivery Tomorrow — {{notify_customers.item.delivery_time_window}} at {{notify_customers.item.site_address}}",
          "body": "Hi {{get_customer.output.record.contact_name}},\n\nThis is a confirmation that your rental delivery is scheduled for tomorrow.\n\nDelivery window: {{notify_customers.item.delivery_time_window}}\nLocation: {{notify_customers.item.site_address}}\n\nItems being delivered:\n{{get_items.output.records.map(i => '- ' + i.equipment + ' (qty: ' + i.quantity_confirmed + ')').join('\\n')}}\n\nSetup instructions on file: {{notify_customers.item.setup_instructions}}\n\nIf there are any changes to venue access, timing, or layout, please reply to this email or call us before 7 AM tomorrow.\n\nThank you."
        }
      }
    ]
  },
  {
    "id": "build_route_sheet",
    "type": "tool_call",
    "description": "Use LLM to organize deliveries into an optimized route schedule",
    "tool_name": "llm_generate_text",
    "input": {
      "prompt": "Create a delivery route sheet for tomorrow ({{new Date(Date.now() + 86400000).toISOString().split('T')[0]}}). Organize the following deliveries by delivery time window, and for each stop include: order ID, customer name, address, delivery window, access notes, and item count.\n\nDeliveries:\n{{JSON.stringify(get_tomorrow_orders.output.records.map(o => ({id: o.id, address: o.site_address, time_window: o.delivery_time_window, access_notes: o.access_notes, customer: o.customer})))}}\n\nFormat as a clear, printable route sheet."
    }
  },
  {
    "id": "get_routes",
    "type": "tool_call",
    "description": "Get delivery routes for tomorrow to find assigned drivers",
    "tool_name": "query_records",
    "input": {
      "table_name": "Delivery Routes",
      "filters": {
        "route_date": "{{new Date(Date.now() + 86400000).toISOString().split('T')[0]}}"
      }
    }
  },
  {
    "id": "email_drivers",
    "type": "foreach",
    "description": "Email each driver their route sheet",
    "items": "{{get_routes.output.records}}",
    "steps": [
      {
        "id": "send_route",
        "type": "tool_call",
        "description": "Send route sheet to driver",
        "tool_name": "gmail_send_email",
        "input": {
          "to": "{{email_drivers.item.driver_email}}",
          "subject": "Delivery Route — {{new Date(Date.now() + 86400000).toISOString().split('T')[0]}} ({{get_tomorrow_orders.output.records.length}} stops)",
          "body": "Hi {{email_drivers.item.driver_name}},\n\nHere is your delivery route for tomorrow:\n\n{{build_route_sheet.output.text}}\n\nPlease review the access notes for each stop. Contact dispatch if you have any questions.\n\nDrive safe."
        }
      }
    ]
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| Coordinator time on delivery scheduling | 2-3 hours/day | 15 minutes/day (review only) |
| Failed deliveries from missing info | 15-20% of events | Under 3% |
| Customer "where's my delivery?" calls | 8-12/week | 1-2/week |

-> [Set up this workflow on Lotics](https://lotics.ai)

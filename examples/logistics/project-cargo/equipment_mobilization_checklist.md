# Equipment Mobilization Checklist and Readiness Confirmation

## The problem

A heavy-lift shipment requires coordinating cranes, multi-axle trailers, escort vehicles, and specialized rigging — all from different vendors. If any single piece of equipment fails to mobilize on time, the entire operation is delayed at a cost of $5,000-20,000/day in standby charges. Project managers manually call each vendor 48 hours before the lift to confirm readiness, but with 5-8 vendors per operation and multiple operations per month, confirmations slip through the cracks.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Operations | operation_name, project_id, operation_date, location, status | Lift/transport operations |
| Equipment Bookings | operation_id, vendor_id, equipment_type, capacity, mobilization_date, confirmed, confirmed_at | Booked equipment |
| Vendors | name, email, phone, equipment_specialty | Equipment vendors |

## Example prompts

- "48 hours before each operation, email every booked vendor asking them to confirm equipment mobilization and track their responses."
- "Automatically send mobilization confirmation requests to all vendors 2 days before each heavy lift."

## Workflow

**Trigger:** Recurring daily schedule at 07:00 UTC to check for operations in 48 hours.

```json
[
  {
    "id": "find_upcoming_ops",
    "type": "tool_call",
    "description": "Find operations scheduled in exactly 2 days",
    "tool_name": "query_records",
    "input": {
      "table_id": "operations",
      "filter": {
        "and": [
          { "field": "operation_date", "operator": "greater_than_or_equals", "value": "{{DATE_ADD(NOW(), 2, 'days')}}" },
          { "field": "operation_date", "operator": "less_than", "value": "{{DATE_ADD(NOW(), 3, 'days')}}" },
          { "field": "status", "operator": "equals", "value": "scheduled" }
        ]
      }
    }
  },
  {
    "id": "process_operations",
    "type": "foreach",
    "description": "For each upcoming operation, request confirmation from all vendors",
    "items": "{{find_upcoming_ops.output.data}}",
    "steps": [
      {
        "id": "get_bookings",
        "type": "tool_call",
        "description": "Get all equipment bookings for this operation",
        "tool_name": "query_records",
        "input": {
          "table_id": "equipment_bookings",
          "filter": {
            "and": [
              { "field": "operation_id", "operator": "equals", "value": "{{process_operations.item.id}}" },
              { "field": "confirmed", "operator": "equals", "value": false }
            ]
          }
        }
      },
      {
        "id": "contact_vendors",
        "type": "foreach",
        "description": "Email each unconfirmed vendor for mobilization confirmation",
        "items": "{{get_bookings.output.data}}",
        "steps": [
          {
            "id": "get_vendor",
            "type": "tool_call",
            "description": "Fetch vendor contact details",
            "tool_name": "get_record",
            "input": {
              "table_id": "vendors",
              "record_id": "{{contact_vendors.item.vendor_id}}"
            }
          },
          {
            "id": "send_confirmation_request",
            "type": "tool_call",
            "description": "Email the vendor asking for mobilization confirmation",
            "tool_name": "outlook_send_email",
            "input": {
              "to": "{{get_vendor.output.data.email}}",
              "subject": "Mobilization Confirmation Required — {{process_operations.item.operation_name}} on {{process_operations.item.operation_date}}",
              "body": "Dear {{get_vendor.output.data.name}},\n\nThis is a mobilization confirmation request for the following operation:\n\nOperation: {{process_operations.item.operation_name}}\nDate: {{process_operations.item.operation_date}}\nLocation: {{process_operations.item.location}}\n\nEquipment booked:\n- Type: {{contact_vendors.item.equipment_type}}\n- Capacity: {{contact_vendors.item.capacity}}\n- Required mobilization date: {{contact_vendors.item.mobilization_date}}\n\nPlease reply to confirm that the equipment will be mobilized on time, or notify us immediately of any issues.\n\nThis confirmation is required within 24 hours.\n\nRegards"
            }
          }
        ]
      },
      {
        "id": "check_all_confirmed",
        "type": "if",
        "description": "If there are unconfirmed bookings, alert the project manager",
        "condition": "{{get_bookings.output.data.length > 0}}",
        "then": [
          {
            "id": "alert_pm",
            "type": "tool_call",
            "description": "Alert project manager about pending confirmations",
            "tool_name": "send_notification",
            "input": {
              "message": "Operation {{process_operations.item.operation_name}} in 48 hours: {{get_bookings.output.data.length}} vendor(s) have not yet confirmed equipment mobilization. Confirmation requests sent. Follow up if no response by end of day.",
              "record_id": "{{process_operations.item.id}}"
            }
          }
        ],
        "else": []
      }
    ]
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| Operations delayed by equipment no-show | 15-20% | Under 3% |
| Standby charges from delays | $15,000-40,000/quarter | Under $5,000/quarter |
| Manual confirmation call time | 3-4 hrs per operation | 15 min review |

→ [Set up this workflow on Lotics](https://lotics.ai)

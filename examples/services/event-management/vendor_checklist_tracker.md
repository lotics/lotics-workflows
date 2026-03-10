# Vendor Confirmation Checklist

## The problem

An event management company runs 4-6 events per month, each involving 8-15 vendors (caterers, florists, AV crews, decorators, photographers). Two weeks before each event, the coordinator manually emails every vendor to confirm availability, delivery times, and special requirements. With 60+ vendor confirmations per month, follow-ups fall through the cracks. At least once a quarter, a vendor no-shows because confirmation was never received, causing last-minute scrambling and cost overruns of $2,000-$5,000 per incident.

## Data model

| Table | Key fields | Purpose |
|-------|-----------|---------|
| Events | event_name, event_date, venue, client_name, status, coordinator_id | Event master records |
| Event Vendors | event_id, vendor_id, service_type, confirmation_status, delivery_time, special_requirements, amount | Vendor assignments per event |
| Vendors | company_name, contact_name, email, phone, category | Vendor directory |

## Example prompts

- "14 days before each event, email every assigned vendor to confirm their attendance, delivery time, and requirements. Track who hasn't responded after 3 days and notify me."
- "Automatically send vendor confirmation requests two weeks before events and flag any vendor that hasn't confirmed within 3 days."

## Workflow

**Trigger:** Recurring schedule, daily at 10:00

```json
[
  {
    "id": "find_upcoming_events",
    "type": "tool_call",
    "description": "Find events happening in exactly 14 days",
    "tool_name": "query_records",
    "input": {
      "table_name": "Events",
      "filters": {
        "event_date": "{{trigger.recurring_schedule.scheduled_at_date_plus_14d}}",
        "status": "active"
      }
    }
  },
  {
    "id": "process_events",
    "type": "foreach",
    "description": "Send vendor confirmations for each upcoming event",
    "items": "{{find_upcoming_events.output.records}}",
    "steps": [
      {
        "id": "get_event_vendors",
        "type": "tool_call",
        "description": "Get all vendor assignments for this event",
        "tool_name": "query_records",
        "input": {
          "table_name": "Event Vendors",
          "filters": {
            "event_id": "{{item.id}}"
          }
        }
      },
      {
        "id": "contact_vendors",
        "type": "foreach",
        "description": "Email each vendor for confirmation",
        "items": "{{get_event_vendors.output.records}}",
        "steps": [
          {
            "id": "get_vendor",
            "type": "tool_call",
            "description": "Fetch vendor contact details",
            "tool_name": "get_record",
            "input": {
              "table_name": "Vendors",
              "record_id": "{{item.vendor_id}}"
            }
          },
          {
            "id": "send_confirmation_request",
            "type": "tool_call",
            "description": "Email vendor requesting confirmation",
            "tool_name": "gmail_send_email",
            "input": {
              "to": "{{get_vendor.output.email}}",
              "subject": "Confirmation Required: {{item.service_type}} for {{process_events.item.event_name}} on {{process_events.item.event_date}}",
              "body": "Hi {{get_vendor.output.contact_name}},\n\nThis is a confirmation request for the upcoming event:\n\nEvent: {{process_events.item.event_name}}\nDate: {{process_events.item.event_date}}\nVenue: {{process_events.item.venue}}\nYour Service: {{item.service_type}}\nScheduled Delivery/Arrival: {{item.delivery_time}}\nSpecial Requirements: {{item.special_requirements}}\n\nPlease reply to confirm your availability and the details above. If anything has changed, let us know immediately.\n\nThank you,\nEvent Coordination Team"
            }
          },
          {
            "id": "update_status",
            "type": "tool_call",
            "description": "Mark vendor as awaiting confirmation",
            "tool_name": "update_record",
            "input": {
              "table_name": "Event Vendors",
              "record_id": "{{item.id}}",
              "data": {
                "confirmation_status": "awaiting_response"
              }
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
|--------|--------|-------|
| Vendor no-shows per quarter | 1-2 | 0 |
| Coordinator time on confirmations | 8 hrs/month | 1 hr/month (follow-ups only) |
| Confirmation coverage | 85% (some missed) | 100% |

-> [Set up this workflow on Lotics](https://lotics.ai)

# Class Waitlist Auto-Promotion

## The problem

Popular group classes (spinning, yoga, HIIT) fill up fast and maintain waitlists of 5-10 people. When a member cancels, front-desk staff must manually check the waitlist, call the next person, and update the roster. During peak hours this takes 10-15 minutes per cancellation. If the desk is busy, the spot stays empty and revenue is lost. Members on the waitlist get frustrated when they find out a spot opened and closed without them being notified.

## Data model

| Table | Key fields | Purpose |
|-------|-----------|---------|
| Class Schedule | class_name, instructor, date_time, max_capacity, enrolled_count, status | Scheduled group classes |
| Class Enrollments | class_id, member_id, status, enrolled_at, position | Enrollment and waitlist records |
| Members | full_name, email, phone | Member contact info |

## Example prompts

- "When someone cancels a class enrollment, automatically promote the first person on the waitlist and notify them by email that they've been added to the class."
- "If a class enrollment is cancelled and there's a waitlist, move the next waitlisted member in and send them a confirmation."

## Workflow

**Trigger:** When a class enrollment record is updated and status changes to "cancelled"

```json
[
  {
    "id": "get_class",
    "type": "tool_call",
    "description": "Fetch the class details to check capacity",
    "tool_name": "get_record",
    "input": {
      "table_name": "Class Schedule",
      "record_id": "{{trigger.record_updated.next_data.class_id}}"
    }
  },
  {
    "id": "find_waitlist",
    "type": "tool_call",
    "description": "Find the next waitlisted member sorted by position",
    "tool_name": "query_records",
    "input": {
      "table_name": "Class Enrollments",
      "filters": {
        "class_id": "{{trigger.record_updated.next_data.class_id}}",
        "status": "waitlisted"
      },
      "sort": [{"field": "position", "direction": "asc"}],
      "limit": 1
    }
  },
  {
    "id": "has_waitlist",
    "type": "if",
    "description": "Check if anyone is on the waitlist",
    "condition": "{{find_waitlist.output.records.length > 0}}",
    "then": [
      {
        "id": "promote_member",
        "type": "tool_call",
        "description": "Update the waitlisted enrollment to enrolled status",
        "tool_name": "update_record",
        "input": {
          "table_name": "Class Enrollments",
          "record_id": "{{find_waitlist.output.records[0].id}}",
          "data": {
            "status": "enrolled"
          }
        }
      },
      {
        "id": "update_class_count",
        "type": "tool_call",
        "description": "Keep enrolled count accurate on the class record",
        "tool_name": "update_record",
        "input": {
          "table_name": "Class Schedule",
          "record_id": "{{get_class.output.id}}",
          "data": {
            "enrolled_count": "{{get_class.output.enrolled_count}}"
          }
        }
      },
      {
        "id": "get_promoted_member",
        "type": "tool_call",
        "description": "Fetch the promoted member's contact details",
        "tool_name": "get_record",
        "input": {
          "table_name": "Members",
          "record_id": "{{find_waitlist.output.records[0].member_id}}"
        }
      },
      {
        "id": "notify_member",
        "type": "tool_call",
        "description": "Email the promoted member that they're now enrolled",
        "tool_name": "gmail_send_email",
        "input": {
          "to": "{{get_promoted_member.output.email}}",
          "subject": "You're in! Spot opened in {{get_class.output.class_name}}",
          "body": "Hi {{get_promoted_member.output.full_name}},\n\nA spot just opened up in {{get_class.output.class_name}} on {{get_class.output.date_time}} with {{get_class.output.instructor}}.\n\nYou've been automatically moved from the waitlist to enrolled. No action needed.\n\nIf you can no longer make it, please cancel through your account so the next person on the waitlist can take the spot.\n\nSee you in class!"
        }
      }
    ],
    "else": []
  }
]
```

## Results

| Metric | Before | After |
|--------|--------|-------|
| Time to fill cancelled spots | 10-15 min (manual) | Instant (automated) |
| Unfilled spots from cancellations | 30% stay empty | 0% when waitlist exists |
| Member complaints about waitlist | 5-8/week | Near zero |

-> [Set up this workflow on Lotics](https://lotics.ai)

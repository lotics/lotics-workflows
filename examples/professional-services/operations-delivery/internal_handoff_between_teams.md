# Internal Handoff Between Teams

## The problem

When a consulting engagement moves from the discovery phase to implementation, the handoff between the strategy team and the delivery team is chaotic. Context gets lost in Slack threads, key documents aren't attached, and the implementation team wastes the first week re-learning what the strategy team already figured out. For a firm running 40+ engagements a year, this adds up to hundreds of wasted hours and frustrated clients who have to repeat themselves.

## Data model

| Table | Key fields | Purpose |
|-------|-----------|---------|
| Engagements | engagement_name, client, phase, strategy_lead, delivery_lead, handoff_status | Tracks each client engagement lifecycle |
| Handoff Checklists | engagement (link), item_name, completed, completed_by, notes | Checklist items that must be done before handoff |
| Engagement Documents | engagement (link), file, document_type, uploaded_by | Key files attached to the engagement |

## Example prompts

- "When an engagement phase changes to 'Implementation', check if all handoff checklist items are done. If yes, notify the delivery lead and send them a summary with all documents. If not, alert the strategy lead about what's missing."
- "Automate the team handoff process: verify the checklist, compile documents, send a briefing to the new team lead, and lock the discovery record."

## Workflow

**Trigger:** `record_updated` on the Engagements table — fires when `phase` changes to `Implementation`.

```json
[
  {
    "id": "get_checklist_items",
    "type": "tool_call",
    "description": "Fetch all handoff checklist items for this engagement",
    "tool_name": "query_records",
    "input": {
      "table_name": "Handoff Checklists",
      "filters": {
        "engagement": "{{trigger.record_updated.record_id}}"
      }
    }
  },
  {
    "id": "check_all_complete",
    "type": "if",
    "description": "Verify all checklist items are marked complete",
    "condition": "{{get_checklist_items.output.records.every(item => item.completed)}}",
    "then": [
      {
        "id": "get_documents",
        "type": "tool_call",
        "description": "Fetch all documents attached to this engagement",
        "tool_name": "query_records",
        "input": {
          "table_name": "Engagement Documents",
          "filters": {
            "engagement": "{{trigger.record_updated.record_id}}"
          }
        }
      },
      {
        "id": "generate_briefing",
        "type": "tool_call",
        "description": "Generate a handoff briefing summarizing the engagement context",
        "tool_name": "llm_generate_text",
        "input": {
          "prompt": "Write a concise handoff briefing for the delivery team taking over this engagement.\n\nEngagement: {{trigger.record_updated.next_data.engagement_name}}\nClient: {{trigger.record_updated.next_data.client}}\n\nChecklist items completed:\n{{JSON.stringify(get_checklist_items.output.records)}}\n\nDocuments available:\n{{JSON.stringify(get_documents.output.records)}}\n\nInclude sections: Client Context, Key Decisions Made, Open Items, and Document Index."
        }
      },
      {
        "id": "post_briefing_comment",
        "type": "tool_call",
        "description": "Post the briefing as a comment on the engagement record",
        "tool_name": "create_record_comments",
        "input": {
          "record_id": "{{trigger.record_updated.record_id}}",
          "comment": "{{generate_briefing.output.text}}"
        }
      },
      {
        "id": "notify_delivery_lead",
        "type": "tool_call",
        "description": "Notify the delivery lead that handoff is ready",
        "tool_name": "send_notification",
        "input": {
          "member_id": "{{trigger.record_updated.next_data.delivery_lead}}",
          "title": "Handoff ready: {{trigger.record_updated.next_data.engagement_name}}",
          "message": "The strategy team has completed the handoff checklist for {{trigger.record_updated.next_data.engagement_name}}. A briefing has been posted to the engagement record with all documents indexed."
        }
      },
      {
        "id": "update_handoff_status",
        "type": "tool_call",
        "description": "Mark the engagement handoff as complete and lock the discovery phase",
        "tool_name": "update_record",
        "input": {
          "table_name": "Engagements",
          "record_id": "{{trigger.record_updated.record_id}}",
          "data": {
            "handoff_status": "Complete"
          }
        }
      },
      {
        "id": "lock_engagement",
        "type": "tool_call",
        "description": "Lock the engagement record to prevent accidental edits to discovery data",
        "tool_name": "lock_record",
        "input": {
          "table_name": "Engagements",
          "record_id": "{{trigger.record_updated.record_id}}"
        }
      }
    ],
    "else": [
      {
        "id": "find_incomplete_items",
        "type": "tool_call",
        "description": "Query specifically for incomplete checklist items",
        "tool_name": "query_records",
        "input": {
          "table_name": "Handoff Checklists",
          "filters": {
            "engagement": "{{trigger.record_updated.record_id}}",
            "completed": false
          }
        }
      },
      {
        "id": "alert_strategy_lead",
        "type": "tool_call",
        "description": "Notify the strategy lead about incomplete items blocking handoff",
        "tool_name": "send_notification",
        "input": {
          "member_id": "{{trigger.record_updated.next_data.strategy_lead}}",
          "title": "Handoff blocked: {{trigger.record_updated.next_data.engagement_name}}",
          "message": "{{find_incomplete_items.output.total}} checklist items are incomplete. The handoff to the delivery team cannot proceed until all items are marked done."
        }
      },
      {
        "id": "revert_phase",
        "type": "tool_call",
        "description": "Revert the phase back to Discovery since handoff requirements aren't met",
        "tool_name": "update_record",
        "input": {
          "table_name": "Engagements",
          "record_id": "{{trigger.record_updated.record_id}}",
          "data": {
            "phase": "Discovery",
            "handoff_status": "Blocked"
          }
        }
      }
    ]
  }
]
```

## Results

| Metric | Before | After |
|--------|--------|-------|
| Avg. ramp-up time for delivery team | 5 days | 1 day |
| Context loss incidents per quarter | 8-12 | 0 |
| Incomplete handoffs reaching delivery | ~30% | 0% |

-> [Set up this workflow on Lotics](https://lotics.ai)

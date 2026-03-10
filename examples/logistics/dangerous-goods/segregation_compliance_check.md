# DG Segregation Compliance Check

## The problem

Dangerous goods of incompatible classes cannot be stored or transported together — for example, flammable liquids (Class 3) must be segregated from oxidizers (Class 5.1). IMDG and IATA regulations define a detailed segregation matrix. Warehouses and freight forwarders loading mixed-DG containers must manually check every item combination against the matrix. A single segregation violation can result in a $25,000-100,000 fine and, more critically, a catastrophic incident. With 5+ DG items per container, the number of pairwise checks grows quickly, and manual verification is both slow and unreliable.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Containers | container_number, shipment_id, status | Container master |
| Container DG Items | container_id, un_number, proper_shipping_name, class, subsidiary_risk, packing_group, qty, location_in_container | DG items loaded in container |
| Segregation Rules | class_a, class_b, segregation_type, restriction | IMDG segregation matrix |
| Compliance Checks | container_id, check_date, compliant, violations, checked_by | Audit trail |

## Example prompts

- "When a DG item is added to a container, check it against all existing items in that container for segregation conflicts."
- "Automatically validate dangerous goods segregation every time a new item is loaded into a container."

## Workflow

**Trigger:** Container DG Item record created (new item added to container).

```json
[
  {
    "id": "get_new_item",
    "type": "tool_call",
    "description": "Fetch the newly added DG item",
    "tool_name": "get_record",
    "input": {
      "table_id": "container_dg_items",
      "record_id": "{{trigger.record_updated.record_id}}"
    }
  },
  {
    "id": "get_existing_items",
    "type": "tool_call",
    "description": "Fetch all other DG items already in this container",
    "tool_name": "query_records",
    "input": {
      "table_id": "container_dg_items",
      "filter": {
        "and": [
          { "field": "container_id", "operator": "equals", "value": "{{get_new_item.output.data.container_id}}" },
          { "field": "id", "operator": "not_equals", "value": "{{get_new_item.output.data.id}}" }
        ]
      }
    }
  },
  {
    "id": "get_segregation_rules",
    "type": "tool_call",
    "description": "Fetch all segregation rules from the IMDG matrix",
    "tool_name": "query_records",
    "input": {
      "table_id": "segregation_rules"
    }
  },
  {
    "id": "check_segregation",
    "type": "tool_call",
    "description": "Check the new item against all existing items using segregation rules",
    "tool_name": "llm_generate_text",
    "input": {
      "prompt": "Check IMDG dangerous goods segregation compliance. New item being loaded: {{JSON.stringify({ un: get_new_item.output.data.un_number, name: get_new_item.output.data.proper_shipping_name, class: get_new_item.output.data.class, subsidiary_risk: get_new_item.output.data.subsidiary_risk })}}. Existing items in container: {{JSON.stringify(get_existing_items.output.data.map(function(i) { return { un: i.un_number, name: i.proper_shipping_name, class: i.class, subsidiary_risk: i.subsidiary_risk } }))}}. Segregation rules: {{JSON.stringify(get_segregation_rules.output.data)}}. Check every pair of (new item class, existing item class) including subsidiary risks against the segregation rules. Return JSON: { \"compliant\": true/false, \"violations\": [{ \"item_a\": string, \"class_a\": string, \"item_b\": string, \"class_b\": string, \"segregation_type\": string, \"restriction\": string }] }. Only return JSON."
    }
  },
  {
    "id": "handle_result",
    "type": "if",
    "description": "If violations found, block and alert; otherwise log compliance",
    "condition": "{{JSON.parse(check_segregation.output.text).compliant === true}}",
    "then": [
      {
        "id": "log_compliant",
        "type": "tool_call",
        "description": "Log the successful compliance check",
        "tool_name": "create_record",
        "input": {
          "table_id": "compliance_checks",
          "data": {
            "container_id": "{{get_new_item.output.data.container_id}}",
            "check_date": "{{NOW()}}",
            "compliant": true,
            "violations": "",
            "checked_by": "automated_workflow"
          }
        }
      }
    ],
    "else": [
      {
        "id": "log_violation",
        "type": "tool_call",
        "description": "Log the segregation violation",
        "tool_name": "create_record",
        "input": {
          "table_id": "compliance_checks",
          "data": {
            "container_id": "{{get_new_item.output.data.container_id}}",
            "check_date": "{{NOW()}}",
            "compliant": false,
            "violations": "{{check_segregation.output.text}}",
            "checked_by": "automated_workflow"
          }
        }
      },
      {
        "id": "lock_container",
        "type": "tool_call",
        "description": "Lock the container to prevent further loading until resolved",
        "tool_name": "lock_record",
        "input": {
          "table_id": "containers",
          "record_id": "{{get_new_item.output.data.container_id}}",
          "reason": "SEGREGATION VIOLATION — loading blocked until resolved"
        }
      },
      {
        "id": "alert_dg_team",
        "type": "tool_call",
        "description": "Send urgent alert about the segregation violation",
        "tool_name": "send_notification",
        "input": {
          "message": "SEGREGATION VIOLATION in container {{get_new_item.output.data.container_id}}: {{get_new_item.output.data.un_number}} ({{get_new_item.output.data.proper_shipping_name}}) conflicts with existing cargo. Violations: {{check_segregation.output.text}}. Container loading BLOCKED. Remove the conflicting item or reassign to a different container.",
          "record_id": "{{get_new_item.output.data.container_id}}"
        }
      }
    ]
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| Segregation violations caught pre-shipment | ~60% | 100% |
| Regulatory fines from DG violations | $10,000-50,000/year | $0 |
| Manual segregation check time | 15-20 min per container | Automatic |

→ [Set up this workflow on Lotics](https://lotics.ai)

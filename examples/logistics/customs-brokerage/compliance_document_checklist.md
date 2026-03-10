# Compliance Document Checklist Enforcement

## The problem

Different commodity types require different supporting documents for customs clearance — certificates of origin, phytosanitary certificates, import permits, lab test reports, CITES permits, etc. Brokers must know which documents are required for each HS chapter and ensure they are all present before submitting the declaration. Missing a single document causes customs to reject the declaration, adding 2-5 days of delay and $200-500 in storage charges. New brokers frequently miss requirements for unfamiliar commodity types.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Declarations | declaration_number, importer_id, status | Declaration master |
| Declaration Lines | declaration_id, hs_code, description | Line items |
| Document Requirements | hs_chapter, required_doc_type, description, mandatory | Rules for what's needed per HS chapter |
| Declaration Documents | declaration_id, doc_type, file_id, verified | Submitted supporting docs |

## Example prompts

- "Before a declaration is submitted, check all line items against the document requirements table and block submission if anything is missing."
- "Validate that all required compliance documents are attached before allowing customs submission."

## Workflow

**Trigger:** Declaration record submit action (pre-submission validation).

```json
[
  {
    "id": "get_declaration",
    "type": "tool_call",
    "description": "Fetch the declaration being submitted",
    "tool_name": "get_record",
    "input": {
      "table_id": "declarations",
      "record_id": "{{trigger.record_submit.record_id}}"
    }
  },
  {
    "id": "get_line_items",
    "type": "tool_call",
    "description": "Fetch all declaration line items to check HS codes",
    "tool_name": "query_records",
    "input": {
      "table_id": "declaration_lines",
      "filter": {
        "field": "declaration_id",
        "operator": "equals",
        "value": "{{trigger.record_submit.record_id}}"
      }
    }
  },
  {
    "id": "get_attached_docs",
    "type": "tool_call",
    "description": "Fetch all documents already attached to this declaration",
    "tool_name": "query_records",
    "input": {
      "table_id": "declaration_documents",
      "filter": {
        "field": "declaration_id",
        "operator": "equals",
        "value": "{{trigger.record_submit.record_id}}"
      }
    }
  },
  {
    "id": "get_all_requirements",
    "type": "tool_call",
    "description": "Fetch all document requirements by HS chapter",
    "tool_name": "query_records",
    "input": {
      "table_id": "document_requirements",
      "filter": {
        "field": "mandatory",
        "operator": "equals",
        "value": true
      }
    }
  },
  {
    "id": "check_compliance",
    "type": "tool_call",
    "description": "Compare required documents against attached documents to find gaps",
    "tool_name": "llm_generate_text",
    "input": {
      "prompt": "Check customs document compliance. Declaration line items with HS codes: {{JSON.stringify(get_line_items.output.data.map(function(l) { return { hs_code: l.hs_code, description: l.description } }))}}. Document requirements by HS chapter: {{JSON.stringify(get_all_requirements.output.data)}}. Documents already attached: {{JSON.stringify(get_attached_docs.output.data.map(function(d) { return d.doc_type }))}}. For each line item, check if its HS chapter (first 2 digits of HS code) has required docs, and whether those docs are in the attached list. Return JSON: { \"is_compliant\": true/false, \"missing_documents\": [{ \"hs_code\": string, \"commodity\": string, \"required_doc\": string }] }. Only return JSON."
    }
  },
  {
    "id": "handle_result",
    "type": "if",
    "description": "Block submission if documents are missing, otherwise proceed",
    "condition": "{{JSON.parse(check_compliance.output.text).is_compliant === true}}",
    "then": [
      {
        "id": "update_status_ready",
        "type": "tool_call",
        "description": "Mark declaration as ready for submission",
        "tool_name": "update_record",
        "input": {
          "table_id": "declarations",
          "record_id": "{{get_declaration.output.data.id}}",
          "data": {
            "status": "ready_to_submit"
          }
        }
      },
      {
        "id": "notify_ready",
        "type": "tool_call",
        "description": "Notify broker that all documents are in order",
        "tool_name": "send_notification",
        "input": {
          "message": "Declaration {{get_declaration.output.data.declaration_number}} — all compliance documents verified. Ready for customs submission.",
          "record_id": "{{get_declaration.output.data.id}}"
        }
      }
    ],
    "else": [
      {
        "id": "notify_missing",
        "type": "tool_call",
        "description": "Alert the broker about missing compliance documents",
        "tool_name": "send_notification",
        "input": {
          "message": "Declaration {{get_declaration.output.data.declaration_number}} cannot be submitted. Missing documents: {{JSON.stringify(JSON.parse(check_compliance.output.text).missing_documents)}}. Please obtain and attach before resubmitting.",
          "record_id": "{{get_declaration.output.data.id}}"
        }
      },
      {
        "id": "lock_declaration",
        "type": "tool_call",
        "description": "Lock the declaration to prevent premature submission",
        "tool_name": "lock_record",
        "input": {
          "table_id": "declarations",
          "record_id": "{{get_declaration.output.data.id}}",
          "reason": "Missing compliance documents — submission blocked"
        }
      }
    ]
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| Declarations rejected for missing docs | 10-15% | Under 1% |
| Delay from rejected declarations | 2-5 days each | Prevented entirely |
| New broker onboarding errors | Frequent | Eliminated by automation |

→ [Set up this workflow on Lotics](https://lotics.ai)

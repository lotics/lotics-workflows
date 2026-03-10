# Cargo Dimension and Transport Document Package

## The problem

Project cargo shipments require detailed technical documentation — GA (General Arrangement) drawings, lifting plans, transport plans, and dimension/weight certificates — assembled into a single package for port authorities, crane operators, and transport companies. Each stakeholder needs specific documents, and missing a document at the port gate can delay a $500,000 lift by days. Operations staff spend 2-3 hours per shipment assembling and distributing the right documents to the right parties.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Projects | project_name, client_id, cargo_description, length_m, width_m, height_m, weight_tonnes | Project master with dimensions |
| Project Documents | project_id, doc_type, doc_name, file_id, uploaded_at | Uploaded technical documents |
| Stakeholders | project_id, role, name, email, required_doc_types | Who needs what documents |

## Example prompts

- "When a project moves to 'documentation complete', assemble each stakeholder's document package and email it to them."
- "Automatically distribute the right technical documents to each stakeholder when the document set is finalized."

## Workflow

**Trigger:** Project record updated with status changed to "documentation_complete".

```json
[
  {
    "id": "get_project",
    "type": "tool_call",
    "description": "Fetch the project record with cargo details",
    "tool_name": "get_record",
    "input": {
      "table_id": "projects",
      "record_id": "{{trigger.record_updated.record_id}}"
    }
  },
  {
    "id": "get_all_documents",
    "type": "tool_call",
    "description": "Fetch all uploaded documents for this project",
    "tool_name": "query_records",
    "input": {
      "table_id": "project_documents",
      "filter": {
        "field": "project_id",
        "operator": "equals",
        "value": "{{trigger.record_updated.record_id}}"
      }
    }
  },
  {
    "id": "get_stakeholders",
    "type": "tool_call",
    "description": "Fetch all stakeholders who need document packages",
    "tool_name": "query_records",
    "input": {
      "table_id": "stakeholders",
      "filter": {
        "field": "project_id",
        "operator": "equals",
        "value": "{{trigger.record_updated.record_id}}"
      }
    }
  },
  {
    "id": "generate_summary_sheet",
    "type": "tool_call",
    "description": "Generate a cargo summary sheet with dimensions and weight",
    "tool_name": "generate_pdf_from_html_template",
    "input": {
      "template_id": "cargo_summary_template",
      "data": {
        "project_name": "{{get_project.output.data.project_name}}",
        "cargo_description": "{{get_project.output.data.cargo_description}}",
        "length_m": "{{get_project.output.data.length_m}}",
        "width_m": "{{get_project.output.data.width_m}}",
        "height_m": "{{get_project.output.data.height_m}}",
        "weight_tonnes": "{{get_project.output.data.weight_tonnes}}"
      }
    }
  },
  {
    "id": "distribute_to_stakeholders",
    "type": "foreach",
    "description": "Send each stakeholder their required document package",
    "items": "{{get_stakeholders.output.data}}",
    "steps": [
      {
        "id": "match_documents",
        "type": "tool_call",
        "description": "Identify which documents this stakeholder needs from the full set",
        "tool_name": "llm_generate_text",
        "input": {
          "prompt": "From these documents: {{JSON.stringify(get_all_documents.output.data.map(function(d) { return { doc_type: d.doc_type, doc_name: d.doc_name, file_id: d.file_id } }))}}. This stakeholder needs these document types: {{JSON.stringify(distribute_to_stakeholders.item.required_doc_types)}}. Return a JSON array of matching file_ids. Only return the JSON array."
        }
      },
      {
        "id": "send_package",
        "type": "tool_call",
        "description": "Email the document package to this stakeholder",
        "tool_name": "gmail_send_email",
        "input": {
          "to": "{{distribute_to_stakeholders.item.email}}",
          "subject": "Technical Document Package — {{get_project.output.data.project_name}} ({{get_project.output.data.cargo_description}})",
          "body": "Dear {{distribute_to_stakeholders.item.name}},\n\nPlease find attached the technical document package for project {{get_project.output.data.project_name}}.\n\nCargo: {{get_project.output.data.cargo_description}}\nDimensions: {{get_project.output.data.length_m}}m x {{get_project.output.data.width_m}}m x {{get_project.output.data.height_m}}m\nWeight: {{get_project.output.data.weight_tonnes}} tonnes\n\nDocuments included as per your requirements. Please review and confirm receipt.\n\nRegards",
          "attachments": "{{JSON.parse(match_documents.output.text)}}"
        }
      },
      {
        "id": "log_distribution",
        "type": "tool_call",
        "description": "Log the document distribution for audit trail",
        "tool_name": "create_record_comments",
        "input": {
          "table_id": "projects",
          "record_id": "{{get_project.output.data.id}}",
          "comment": "Document package sent to {{distribute_to_stakeholders.item.name}} ({{distribute_to_stakeholders.item.role}}): {{distribute_to_stakeholders.item.required_doc_types}}"
        }
      }
    ]
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| Time to assemble and distribute doc packages | 2-3 hours per project | Under 10 minutes |
| Missing documents at port/site | 2-3 per quarter | 0 |
| Document distribution errors | 10-15% of stakeholders | 0% |

→ [Set up this workflow on Lotics](https://lotics.ai)

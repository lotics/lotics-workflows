# Container Gate-In Inspection Workflow

## The problem

When a container arrives at the depot, inspectors walk around it noting damage — dents, holes, rust, door seal condition — on paper forms that get transcribed later. Photos are taken on personal phones and often never make it into the system. When a shipping line disputes a damage claim, the depot has no timestamped photographic evidence linked to the inspection record. Disputed claims cost depots $500-2,000 per container, and with 50+ gate-ins per day, even 5% disputes add up fast.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Containers | container_number, size, type, owner_line, current_status, location_in_yard | Container master |
| Inspections | container_id, inspector_name, gate_in_date, condition_grade, damage_notes, status | Inspection records |
| Damage Items | inspection_id, damage_type, location_on_container, severity, photo_file_id, repair_needed | Individual damage line items |

## Example prompts

- "When an inspector submits the gate-in form, create the inspection record, extract damage details from the uploaded photos, and update the container status."
- "Process container gate-in inspections automatically — log damage, attach photos, and flag containers needing repair."

## Workflow

**Trigger:** Form submitted by yard inspector with container number, photos, and initial notes.

```json
[
  {
    "id": "find_container",
    "type": "tool_call",
    "description": "Look up the container record by container number",
    "tool_name": "query_records",
    "input": {
      "table_id": "containers",
      "filter": {
        "field": "container_number",
        "operator": "equals",
        "value": "{{trigger.form_submitted.data.container_number}}"
      }
    }
  },
  {
    "id": "extract_damage_from_photos",
    "type": "tool_call",
    "description": "Use AI to extract damage details from the inspection photos",
    "tool_name": "extract_file_data",
    "input": {
      "file_id": "{{trigger.form_submitted.data.photo_file_ids[0]}}",
      "extraction_prompt": "Analyze this container inspection photo. Identify: damage type (dent, hole, rust, seal damage, floor damage), location on container (front, back, left, right, top, floor, door), severity (minor, moderate, major), and whether repair is needed (yes/no). Return JSON array of damage items."
    }
  },
  {
    "id": "create_inspection",
    "type": "tool_call",
    "description": "Create the inspection record",
    "tool_name": "create_record",
    "input": {
      "table_id": "inspections",
      "data": {
        "container_id": "{{find_container.output.data[0].id}}",
        "inspector_name": "{{trigger.form_submitted.data.inspector_name}}",
        "gate_in_date": "{{NOW()}}",
        "damage_notes": "{{trigger.form_submitted.data.notes}}",
        "status": "completed"
      }
    }
  },
  {
    "id": "attach_photos",
    "type": "tool_call",
    "description": "Attach all inspection photos to the inspection record",
    "tool_name": "add_files_to_record",
    "input": {
      "table_id": "inspections",
      "record_id": "{{create_inspection.output.data.id}}",
      "file_ids": "{{trigger.form_submitted.data.photo_file_ids}}"
    }
  },
  {
    "id": "log_damage_items",
    "type": "foreach",
    "description": "Create individual damage item records from extracted data",
    "items": "{{JSON.parse(extract_damage_from_photos.output.data)}}",
    "steps": [
      {
        "id": "create_damage_item",
        "type": "tool_call",
        "description": "Create a damage line item record",
        "tool_name": "create_record",
        "input": {
          "table_id": "damage_items",
          "data": {
            "inspection_id": "{{create_inspection.output.data.id}}",
            "damage_type": "{{log_damage_items.item.damage_type}}",
            "location_on_container": "{{log_damage_items.item.location}}",
            "severity": "{{log_damage_items.item.severity}}",
            "repair_needed": "{{log_damage_items.item.repair_needed}}"
          }
        }
      }
    ]
  },
  {
    "id": "update_container_status",
    "type": "tool_call",
    "description": "Update the container record with gate-in status and yard location",
    "tool_name": "update_record",
    "input": {
      "table_id": "containers",
      "record_id": "{{find_container.output.data[0].id}}",
      "data": {
        "current_status": "in_yard",
        "location_in_yard": "{{trigger.form_submitted.data.yard_position}}"
      }
    }
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| Inspection documentation time | 20-30 min per container | Under 5 minutes |
| Disputed damage claims | 5-8% of gate-ins | Under 1% |
| Missing inspection photos | 30-40% of records | 0% |

→ [Set up this workflow on Lotics](https://lotics.ai)

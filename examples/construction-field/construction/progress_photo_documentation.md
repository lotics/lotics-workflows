# Progress Photo Documentation

## The problem

Owners, architects, and lenders require weekly progress photos with structured documentation -- phase, area, description, and comparison to drawings. Site teams take 50-100 photos per week on their phones but dump them into a shared folder with no labels or organization. The project engineer spends 3-4 hours every Friday sorting, labeling, and compiling photos into a presentable report. Photos without context are useless for dispute resolution and lender draw requests.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Progress Photos | project (link), phase, area, description, photo_date, submitted_by | Individual photo records with metadata |
| Projects | project_name, owner_email, architect_email, lender_email | Project contacts for report distribution |
| Photo Reports | project (link), report_date, report_file, total_photos | Weekly compiled photo reports |

## Example prompts

- "When a photo is submitted via the progress photo form, use AI to generate a description from the image, then every Friday at 3 PM, compile all photos from the week into a PDF report grouped by phase and area, and email it to the owner, architect, and lender."
- "Auto-describe submitted site photos using AI, then generate a weekly photo progress report PDF and distribute it to all project stakeholders."

## Workflow

**Trigger:** When the record submit action fires on the Progress Photos table

```json
[
  {
    "id": "read_photo",
    "type": "tool_call",
    "description": "Read the uploaded photo file",
    "tool_name": "read_files",
    "input": {
      "record_id": "{{trigger.record_submit.record_id}}",
      "table_name": "Progress Photos"
    }
  },
  {
    "id": "generate_description",
    "type": "tool_call",
    "description": "Use LLM to generate a structured description of the construction progress photo",
    "tool_name": "llm_generate_text",
    "input": {
      "prompt": "You are a construction documentation specialist. Based on the photo metadata below, generate a concise professional description (2-3 sentences) suitable for a progress report to an owner or lender. Include what work is visible, the phase of construction, and any notable conditions.\n\nPhase: {{trigger.record_submit.data.phase}}\nArea: {{trigger.record_submit.data.area}}\nSubmitted by: {{trigger.record_submit.data.submitted_by}}\nDate: {{trigger.record_submit.data.photo_date}}"
    }
  },
  {
    "id": "update_photo_description",
    "type": "tool_call",
    "description": "Save the AI-generated description to the photo record",
    "tool_name": "update_record",
    "input": {
      "table_name": "Progress Photos",
      "record_id": "{{trigger.record_submit.record_id}}",
      "data": {
        "description": "{{generate_description.output.text}}"
      }
    }
  }
]
```

-> [Set up this workflow on Lotics](https://lotics.ai)

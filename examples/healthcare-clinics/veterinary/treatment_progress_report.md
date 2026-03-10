# Treatment Progress Report for Pet Owners

## The problem

Pets undergoing multi-visit treatments — post-surgical recovery, chronic condition management, dental treatment plans — require regular updates to owners. Veterinarians document progress in clinical notes, but translating those into owner-friendly updates falls to vet techs who are already stretched thin. Owners call in for updates, tying up phone lines, and some disengage entirely when they don't hear anything, skipping follow-up visits that are critical to recovery.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Treatment Plans | patient_id, pet_name, owner_name, owner_email, diagnosis, plan_description, start_date, status, provider | Active and completed treatment plans |
| Visit Notes | treatment_plan_id, visit_date, weight, vitals, clinical_notes, next_steps, provider | Per-visit clinical documentation |
| Patients | pet_name, species, breed, owner_name, owner_email, status | Patient directory |

## Example prompts

- "After a vet adds visit notes for a pet on an active treatment plan, use AI to rewrite the clinical notes into a simple owner-friendly update and email it to the owner with the next steps."
- "When visit notes are added, generate a pet owner progress report from the clinical notes and send it by email automatically."

## Workflow

**Trigger:** `button_pressed` — Veterinarian clicks "Send Owner Update" after completing visit notes.

```json
[
  {
    "id": "get_visit_notes",
    "type": "tool_call",
    "description": "Retrieve the visit notes that the vet just completed",
    "tool_name": "get_record",
    "input": {
      "table_name": "Visit Notes",
      "record_id": "{{trigger.button_pressed.record_id}}"
    }
  },
  {
    "id": "get_treatment_plan",
    "type": "tool_call",
    "description": "Look up the treatment plan for context on the diagnosis and overall plan",
    "tool_name": "query_records",
    "input": {
      "table_name": "Treatment Plans",
      "filters": {
        "id": "{{get_visit_notes.output.treatment_plan_id}}"
      }
    }
  },
  {
    "id": "get_visit_history",
    "type": "tool_call",
    "description": "Pull all previous visit notes to understand the treatment trajectory",
    "tool_name": "query_records",
    "input": {
      "table_name": "Visit Notes",
      "filters": {
        "treatment_plan_id": "{{get_visit_notes.output.treatment_plan_id}}"
      },
      "sort": {
        "field": "visit_date",
        "direction": "asc"
      }
    }
  },
  {
    "id": "generate_owner_update",
    "type": "tool_call",
    "description": "Use AI to translate clinical notes into a clear, reassuring owner-friendly update",
    "tool_name": "llm_generate_text",
    "input": {
      "prompt": "You are a veterinary communications assistant. Write a warm, clear email update for a pet owner about their pet's treatment progress. Avoid medical jargon — use plain language.\n\nPet name: {{get_treatment_plan.output.records[0].pet_name}}\nDiagnosis: {{get_treatment_plan.output.records[0].diagnosis}}\nTreatment plan: {{get_treatment_plan.output.records[0].plan_description}}\nToday's visit date: {{get_visit_notes.output.visit_date}}\nToday's weight: {{get_visit_notes.output.weight}}\nToday's clinical notes: {{get_visit_notes.output.clinical_notes}}\nNext steps from vet: {{get_visit_notes.output.next_steps}}\nTotal visits so far: {{get_visit_history.output.records.length}}\nProvider: {{get_visit_notes.output.provider}}\n\nInclude: how the pet is doing, what happened today, what the owner should watch for at home, and what comes next. Keep it under 200 words. Sign off as the veterinary team."
    }
  },
  {
    "id": "send_owner_update",
    "type": "tool_call",
    "description": "Email the AI-generated progress report to the pet owner",
    "tool_name": "gmail_send_email",
    "input": {
      "to": "{{get_treatment_plan.output.records[0].owner_email}}",
      "subject": "Update on {{get_treatment_plan.output.records[0].pet_name}}'s treatment - Visit {{get_visit_history.output.records.length}}",
      "body": "Hi {{get_treatment_plan.output.records[0].owner_name}},\n\n{{generate_owner_update.output.text}}"
    }
  },
  {
    "id": "log_communication",
    "type": "tool_call",
    "description": "Add a comment to the visit notes confirming the owner update was sent",
    "tool_name": "create_record_comments",
    "input": {
      "table_name": "Visit Notes",
      "record_id": "{{trigger.button_pressed.record_id}}",
      "comment": "Owner progress update emailed to {{get_treatment_plan.output.records[0].owner_email}} on {{new Date().toISOString().split('T')[0]}}."
    }
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| Time spent writing owner updates | 10 min/visit | 0 (one-click) |
| Owner calls requesting status updates | 15/week | 3/week |
| Follow-up visit completion rate | 72% | 91% |
| Owner satisfaction score | 3.8/5 | 4.6/5 |

-> [Set up this workflow on Lotics](https://lotics.ai)

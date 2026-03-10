# New Pet Patient Onboarding

## The problem

When a new client registers their pet, the front desk must create the patient record, request prior medical records from the previous vet, set up the vaccination schedule, and send the owner a welcome packet. With 8-12 new patients per week, this manual process takes 15-20 minutes each and steps get missed — especially requesting prior records, which delays the first exam because the veterinarian has no medical history to review.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Patients | pet_name, species, breed, weight, date_of_birth, owner_name, owner_email, previous_vet, previous_vet_email, status | Pet patient records |
| Vaccinations | patient_id, pet_name, vaccine_name, next_due_date, status, owner_email | Vaccination tracking |
| Documents | patient_id, document_type, file, uploaded_date, status | Medical records and forms |

## Example prompts

- "When a new patient record is submitted, email the previous vet requesting medical records, create the initial vaccination schedule based on species, and send the owner a welcome email with what to bring to their first visit."
- "Automate new pet onboarding — request records from the old vet, set up vaccines, and welcome the owner, all triggered when we add a new patient."

## Workflow

**Trigger:** `record_submit` — Staff submits a new patient record after initial registration.

```json
[
  {
    "id": "request_prior_records",
    "type": "if",
    "description": "If a previous vet is listed, email them requesting medical records",
    "condition": "{{trigger.record_submit.data.previous_vet_email !== ''}}",
    "then": [
      {
        "id": "email_previous_vet",
        "type": "tool_call",
        "description": "Send a records request to the previous veterinary clinic",
        "tool_name": "gmail_send_email",
        "input": {
          "to": "{{trigger.record_submit.data.previous_vet_email}}",
          "subject": "Medical Records Request - {{trigger.record_submit.data.pet_name}} (Owner: {{trigger.record_submit.data.owner_name}})",
          "body": "Dear {{trigger.record_submit.data.previous_vet}},\n\nWe are writing to request the medical records for the following patient who has transferred to our clinic:\n\nPet Name: {{trigger.record_submit.data.pet_name}}\nSpecies: {{trigger.record_submit.data.species}}\nBreed: {{trigger.record_submit.data.breed}}\nOwner: {{trigger.record_submit.data.owner_name}}\n\nPlease send vaccination history, lab results, surgical records, and any current medications or ongoing treatment plans.\n\nYou may reply to this email with records attached or fax them to our office.\n\nThank you for your prompt attention.\n\nBest regards,\nRecords Department"
        }
      },
      {
        "id": "create_records_tracker",
        "type": "tool_call",
        "description": "Create a document record to track the pending records request",
        "tool_name": "create_record",
        "input": {
          "table_name": "Documents",
          "data": {
            "patient_id": "{{trigger.record_submit.record_id}}",
            "document_type": "Prior Medical Records",
            "uploaded_date": "{{new Date().toISOString().split('T')[0]}}",
            "status": "Requested"
          }
        }
      }
    ],
    "else": []
  },
  {
    "id": "determine_vaccines",
    "type": "tool_call",
    "description": "Use LLM to determine the standard vaccination schedule based on species and age",
    "tool_name": "llm_generate_text",
    "input": {
      "prompt": "Determine the standard core vaccinations needed for a {{trigger.record_submit.data.species}} (breed: {{trigger.record_submit.data.breed}}, date of birth: {{trigger.record_submit.data.date_of_birth}}). List each vaccine name and the next due date in JSON array format: [{\"vaccine_name\": \"...\", \"next_due_date\": \"YYYY-MM-DD\"}]. Use today's date {{new Date().toISOString().split('T')[0]}} as the baseline. Include only core vaccines. Return only the JSON array, no other text."
    }
  },
  {
    "id": "create_vaccine_records",
    "type": "foreach",
    "description": "Create a vaccination record for each recommended vaccine",
    "items": "{{JSON.parse(determine_vaccines.output.text)}}",
    "steps": [
      {
        "id": "create_vaccine",
        "type": "tool_call",
        "description": "Create the vaccination schedule entry",
        "tool_name": "create_record",
        "input": {
          "table_name": "Vaccinations",
          "data": {
            "patient_id": "{{trigger.record_submit.record_id}}",
            "pet_name": "{{trigger.record_submit.data.pet_name}}",
            "vaccine_name": "{{item.vaccine_name}}",
            "next_due_date": "{{item.next_due_date}}",
            "status": "Scheduled",
            "owner_email": "{{trigger.record_submit.data.owner_email}}"
          }
        }
      }
    ]
  },
  {
    "id": "send_welcome_email",
    "type": "tool_call",
    "description": "Send the pet owner a welcome email with first-visit instructions",
    "tool_name": "gmail_send_email",
    "input": {
      "to": "{{trigger.record_submit.data.owner_email}}",
      "subject": "Welcome to our clinic - {{trigger.record_submit.data.pet_name}}'s first visit",
      "body": "Hi {{trigger.record_submit.data.owner_name}},\n\nWelcome! We're excited to care for {{trigger.record_submit.data.pet_name}}.\n\nHere's what to bring to your first appointment:\n- Any medications {{trigger.record_submit.data.pet_name}} is currently taking\n- Previous vaccination records (if you have copies)\n- A list of any health concerns or questions\n- Your pet's favorite treat (to make the visit more comfortable)\n\nWe've set up {{trigger.record_submit.data.pet_name}}'s vaccination schedule and will send reminders as vaccines come due.\n\nIf you have questions before your visit, reply to this email or call our front desk.\n\nWe look forward to meeting {{trigger.record_submit.data.pet_name}}!\n\nWarm regards,\nYour Veterinary Team"
    }
  },
  {
    "id": "update_patient_status",
    "type": "tool_call",
    "description": "Mark the patient record as fully onboarded",
    "tool_name": "update_record",
    "input": {
      "table_name": "Patients",
      "record_id": "{{trigger.record_submit.record_id}}",
      "data": {
        "status": "Active"
      }
    }
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| Onboarding time per new patient | 18 min | 2 min (review only) |
| Prior records requested on time | 70% | 100% |
| Missing vaccination schedules at first visit | 25% | 0% |
| Owner satisfaction (first-visit survey) | 3.6/5 | 4.7/5 |

-> [Set up this workflow on Lotics](https://lotics.ai)

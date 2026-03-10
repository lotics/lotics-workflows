# Patient Intake Form Processing

## The problem

New patients fill out intake forms (insurance info, medical history, consent) that staff must manually enter into the system. At clinics onboarding 10-15 new patients per week, data entry takes 20-30 minutes per patient. Errors in transcription lead to insurance claim rejections, and missing consent forms create compliance risk during audits.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Intake Submissions | patient_name, email, phone, insurance_provider, insurance_id, allergies, medications, consent_signed, form_files, status | Raw intake form submissions |
| Patients | full_name, email, phone, insurance_provider, insurance_id, allergies, medications, consent_on_file | Master patient records |
| Compliance Log | patient_name, document_type, date_signed, verified_by | Audit trail for signed documents |

## Example prompts

- "When a patient submits an intake form, extract the data from the uploaded PDF, create a patient record, and log the consent in our compliance table. Notify the front desk if insurance info is missing."
- "Automate new patient intake so the form data gets pulled into our patients table and any missing fields get flagged for the front desk."

## Workflow

**Trigger:** `form_submitted` — Patient submits the online intake form.

```json
[
  {
    "id": "extract_form_data",
    "type": "tool_call",
    "description": "Extract structured data from the uploaded intake PDF",
    "tool_name": "extract_file_data",
    "input": {
      "table_name": "Intake Submissions",
      "record_id": "{{trigger.form_submitted.record_id}}",
      "field_name": "form_files"
    }
  },
  {
    "id": "create_patient",
    "type": "tool_call",
    "description": "Create a new patient record from the extracted intake data",
    "tool_name": "create_record",
    "input": {
      "table_name": "Patients",
      "data": {
        "full_name": "{{trigger.form_submitted.data.patient_name}}",
        "email": "{{trigger.form_submitted.data.email}}",
        "phone": "{{trigger.form_submitted.data.phone}}",
        "insurance_provider": "{{trigger.form_submitted.data.insurance_provider}}",
        "insurance_id": "{{trigger.form_submitted.data.insurance_id}}",
        "allergies": "{{trigger.form_submitted.data.allergies}}",
        "medications": "{{trigger.form_submitted.data.medications}}",
        "consent_on_file": "{{trigger.form_submitted.data.consent_signed}}"
      }
    }
  },
  {
    "id": "check_insurance",
    "type": "if",
    "description": "Check whether insurance information was provided",
    "condition": "{{trigger.form_submitted.data.insurance_provider === '' || trigger.form_submitted.data.insurance_id === ''}}",
    "then": [
      {
        "id": "flag_missing_insurance",
        "type": "tool_call",
        "description": "Notify front desk that insurance info is missing for this patient",
        "tool_name": "send_notification",
        "input": {
          "message": "New patient {{trigger.form_submitted.data.patient_name}} submitted intake form with missing insurance information. Please follow up before their first visit.",
          "channel": "front-desk"
        }
      },
      {
        "id": "mark_incomplete",
        "type": "tool_call",
        "description": "Update intake submission status to incomplete",
        "tool_name": "update_record",
        "input": {
          "table_name": "Intake Submissions",
          "record_id": "{{trigger.form_submitted.record_id}}",
          "data": {
            "status": "Incomplete - Missing Insurance"
          }
        }
      }
    ],
    "else": [
      {
        "id": "mark_complete",
        "type": "tool_call",
        "description": "Update intake submission status to complete",
        "tool_name": "update_record",
        "input": {
          "table_name": "Intake Submissions",
          "record_id": "{{trigger.form_submitted.record_id}}",
          "data": {
            "status": "Complete"
          }
        }
      }
    ]
  },
  {
    "id": "log_consent",
    "type": "tool_call",
    "description": "Create a compliance log entry for the signed consent form",
    "tool_name": "create_record",
    "input": {
      "table_name": "Compliance Log",
      "data": {
        "patient_name": "{{trigger.form_submitted.data.patient_name}}",
        "document_type": "Intake Consent Form",
        "date_signed": "{{trigger.form_submitted.data.consent_signed}}",
        "verified_by": "Automated Workflow"
      }
    }
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| Data entry time per patient | 25 min | 0 min |
| Insurance claim rejections from typos | 8% | 1% |
| Missing consent forms found in audits | 3-5 per quarter | 0 |
| Staff hours on intake processing/week | 5 hrs | 0.5 hrs (review only) |

-> [Set up this workflow on Lotics](https://lotics.ai)

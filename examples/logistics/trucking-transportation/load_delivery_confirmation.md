# Load Delivery Confirmation and POD Processing

## The problem

After a driver delivers a load, the proof of delivery (POD) — a signed receipt, photo, or scanned document — needs to reach dispatch and billing within hours so invoicing can start. In practice, drivers text photos to dispatchers who manually upload them, or worse, hand in paper PODs at the end of the week. This creates a 3-7 day invoicing lag and frequent disputes when PODs go missing.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Loads | load_number, driver_id, customer_id, origin, destination, status, delivered_at | Load master |
| Drivers | name, email, phone | Driver contact info |
| Customers | name, email, billing_contact | Customer billing info |
| Invoices | load_id, amount, status, created_at | Billing records |

## Example prompts

- "When a driver submits the delivery form, mark the load as delivered, extract POD data from the uploaded photo, and notify billing to invoice."
- "Process proof of delivery submissions automatically — update the load, file the POD, and kick off invoicing."

## Workflow

**Trigger:** Form submitted by driver with load ID, delivery timestamp, and POD photo upload.

```json
[
  {
    "id": "get_load",
    "type": "tool_call",
    "description": "Fetch the load record being delivered",
    "tool_name": "get_record",
    "input": {
      "table_id": "loads",
      "record_id": "{{trigger.form_submitted.data.load_id}}"
    }
  },
  {
    "id": "extract_pod_data",
    "type": "tool_call",
    "description": "Extract delivery details from the uploaded POD image",
    "tool_name": "extract_file_data",
    "input": {
      "file_id": "{{trigger.form_submitted.data.pod_file_id}}",
      "extraction_prompt": "Extract: receiver name, signature present (yes/no), delivery date, delivery time, any notes or exceptions."
    }
  },
  {
    "id": "update_load_delivered",
    "type": "tool_call",
    "description": "Mark the load as delivered with timestamp",
    "tool_name": "update_record",
    "input": {
      "table_id": "loads",
      "record_id": "{{get_load.output.data.id}}",
      "data": {
        "status": "delivered",
        "delivered_at": "{{trigger.form_submitted.data.delivery_timestamp}}",
        "pod_receiver_name": "{{extract_pod_data.output.data.receiver_name}}",
        "pod_has_signature": "{{extract_pod_data.output.data.signature_present}}"
      }
    }
  },
  {
    "id": "attach_pod",
    "type": "tool_call",
    "description": "Attach the POD file to the load record",
    "tool_name": "add_files_to_record",
    "input": {
      "table_id": "loads",
      "record_id": "{{get_load.output.data.id}}",
      "file_ids": ["{{trigger.form_submitted.data.pod_file_id}}"]
    }
  },
  {
    "id": "check_exceptions",
    "type": "if",
    "description": "Check if POD has exceptions that need review before invoicing",
    "condition": "{{extract_pod_data.output.data.notes !== '' && extract_pod_data.output.data.notes !== null}}",
    "then": [
      {
        "id": "notify_dispatch_exception",
        "type": "tool_call",
        "description": "Alert dispatch about delivery exceptions",
        "tool_name": "send_notification",
        "input": {
          "message": "Load {{get_load.output.data.load_number}} delivered with exceptions: {{extract_pod_data.output.data.notes}}. Review before invoicing.",
          "record_id": "{{get_load.output.data.id}}"
        }
      }
    ],
    "else": [
      {
        "id": "get_customer",
        "type": "tool_call",
        "description": "Fetch customer billing details",
        "tool_name": "get_record",
        "input": {
          "table_id": "customers",
          "record_id": "{{get_load.output.data.customer_id}}"
        }
      },
      {
        "id": "notify_billing",
        "type": "tool_call",
        "description": "Notify billing team to generate invoice",
        "tool_name": "send_notification",
        "input": {
          "message": "Load {{get_load.output.data.load_number}} delivered to {{get_customer.output.data.name}} — POD confirmed with signature. Ready for invoicing.",
          "record_id": "{{get_load.output.data.id}}"
        }
      }
    ]
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| POD-to-invoice lag | 3-7 days | Same day |
| Missing or lost PODs | 5-10% of loads | 0% |
| Billing disputes from missing PODs | 8-12/month | 1-2/month |

→ [Set up this workflow on Lotics](https://lotics.ai)

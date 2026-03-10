# Cold Chain Compliance Report Generation

## The problem

GDP regulations require pharmaceutical distributors to maintain and document temperature records for every shipment of temperature-sensitive products. Auditors and customers routinely request cold chain compliance reports showing that products were stored and transported within the required 2-8C range throughout the supply chain. Generating these reports manually — pulling temperature logger data, cross-referencing with shipment records, and formatting a compliant report — takes the quality team 45-60 minutes per shipment. With 50+ temperature-sensitive shipments per week, this consumes an entire FTE.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Shipments | shipment_ref, customer_id, product_sku, batch_number, ship_date, delivery_date, temp_min, temp_max, temp_range | Shipment master |
| Temperature Readings | shipment_id, timestamp, temperature, logger_serial | Continuous logger data |
| Compliance Reports | shipment_id, report_type, min_temp, max_temp, mean_temp, excursion_count, compliant, file_id, generated_at | Generated reports |
| Customers | name, email | Customer contacts |

## Example prompts

- "When a shipment is delivered, generate a cold chain compliance report from the temperature readings and email it to the customer."
- "Automatically produce the GDP temperature report on delivery and attach it to the shipment."

## Workflow

**Trigger:** Shipment record updated with status changed to "delivered".

```json
[
  {
    "id": "get_shipment",
    "type": "tool_call",
    "description": "Fetch the delivered shipment record",
    "tool_name": "get_record",
    "input": {
      "table_id": "shipments",
      "record_id": "{{trigger.record_updated.record_id}}"
    }
  },
  {
    "id": "get_temp_readings",
    "type": "tool_call",
    "description": "Query all temperature readings for this shipment",
    "tool_name": "query_records",
    "input": {
      "table_id": "temperature_readings",
      "filter": {
        "field": "shipment_id",
        "operator": "equals",
        "value": "{{trigger.record_updated.record_id}}"
      }
    }
  },
  {
    "id": "analyze_temperatures",
    "type": "tool_call",
    "description": "Analyze temperature data for compliance statistics",
    "tool_name": "llm_generate_text",
    "input": {
      "prompt": "Analyze these temperature readings for GDP cold chain compliance. Required range: {{get_shipment.output.data.temp_min}}C to {{get_shipment.output.data.temp_max}}C. Readings: {{JSON.stringify(get_temp_readings.output.data.map(function(r) { return { timestamp: r.timestamp, temperature: r.temperature } }))}}. Calculate: min_temp, max_temp, mean_temp, total_readings, excursion_count (readings outside range), excursion_duration_minutes, compliant (true if no excursions or excursions under 15 minutes total). Return JSON: { \"min_temp\": number, \"max_temp\": number, \"mean_temp\": number, \"total_readings\": number, \"excursion_count\": number, \"excursion_duration_minutes\": number, \"compliant\": boolean, \"summary\": string }. Only return JSON."
    }
  },
  {
    "id": "generate_report_pdf",
    "type": "tool_call",
    "description": "Generate the cold chain compliance report PDF",
    "tool_name": "generate_pdf_from_html_template",
    "input": {
      "template_id": "cold_chain_report_template",
      "data": {
        "shipment_ref": "{{get_shipment.output.data.shipment_ref}}",
        "product_sku": "{{get_shipment.output.data.product_sku}}",
        "batch_number": "{{get_shipment.output.data.batch_number}}",
        "ship_date": "{{get_shipment.output.data.ship_date}}",
        "delivery_date": "{{get_shipment.output.data.delivery_date}}",
        "required_range": "{{get_shipment.output.data.temp_min}}C to {{get_shipment.output.data.temp_max}}C",
        "analysis": "{{analyze_temperatures.output.text}}",
        "readings": "{{get_temp_readings.output.data}}"
      }
    }
  },
  {
    "id": "create_report_record",
    "type": "tool_call",
    "description": "Create the compliance report record",
    "tool_name": "create_record",
    "input": {
      "table_id": "compliance_reports",
      "data": {
        "shipment_id": "{{get_shipment.output.data.id}}",
        "report_type": "cold_chain_gdp",
        "min_temp": "{{JSON.parse(analyze_temperatures.output.text).min_temp}}",
        "max_temp": "{{JSON.parse(analyze_temperatures.output.text).max_temp}}",
        "mean_temp": "{{JSON.parse(analyze_temperatures.output.text).mean_temp}}",
        "excursion_count": "{{JSON.parse(analyze_temperatures.output.text).excursion_count}}",
        "compliant": "{{JSON.parse(analyze_temperatures.output.text).compliant}}",
        "file_id": "{{generate_report_pdf.output.file_id}}",
        "generated_at": "{{NOW()}}"
      }
    }
  },
  {
    "id": "attach_to_shipment",
    "type": "tool_call",
    "description": "Attach the report PDF to the shipment record",
    "tool_name": "add_files_to_record",
    "input": {
      "table_id": "shipments",
      "record_id": "{{get_shipment.output.data.id}}",
      "file_ids": ["{{generate_report_pdf.output.file_id}}"]
    }
  },
  {
    "id": "get_customer",
    "type": "tool_call",
    "description": "Fetch customer contact for report delivery",
    "tool_name": "get_record",
    "input": {
      "table_id": "customers",
      "record_id": "{{get_shipment.output.data.customer_id}}"
    }
  },
  {
    "id": "email_report",
    "type": "tool_call",
    "description": "Email the cold chain report to the customer",
    "tool_name": "outlook_send_email",
    "input": {
      "to": "{{get_customer.output.data.email}}",
      "subject": "Cold Chain Compliance Report — Shipment {{get_shipment.output.data.shipment_ref}}",
      "body": "Dear {{get_customer.output.data.name}},\n\nPlease find attached the cold chain compliance report for shipment {{get_shipment.output.data.shipment_ref}} (Product: {{get_shipment.output.data.product_sku}}, Batch: {{get_shipment.output.data.batch_number}}).\n\nCompliance status: {{JSON.parse(analyze_temperatures.output.text).compliant ? 'COMPLIANT' : 'NON-COMPLIANT'}}\nTemperature range observed: {{JSON.parse(analyze_temperatures.output.text).min_temp}}C to {{JSON.parse(analyze_temperatures.output.text).max_temp}}C\n\nPlease retain this report for your records as required under GDP.\n\nRegards",
      "attachments": ["{{generate_report_pdf.output.file_id}}"]
    }
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| Time to generate one compliance report | 45-60 minutes | Under 2 minutes |
| Reports generated per week | 50+ (manual) | 50+ (automatic) |
| Quality team hours on report generation | 40+ hrs/week | Under 2 hrs/week |

→ [Set up this workflow on Lotics](https://lotics.ai)

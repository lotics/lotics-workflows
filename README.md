# Lotics Workflows

Ready-to-use workflow examples for [Lotics](https://lotics.ai), the AI-native operations platform. Each example includes a data model, natural-language prompts, and workflow JSON you can deploy by pasting the prompts into the Lotics AI assistant.

## Workflow engine reference

Workflows are declarative step sequences triggered by events. The AI assistant builds these from natural language — the JSON below is what it produces.

**Triggers** — what starts the workflow:

`record_updated` · `recurring_schedule` · `receive_gmail_email` · `receive_outlook_email` · `button_pressed` · `form_submitted` · `record_submit` · `app_action` · `receive_webhook`

**Steps** — what the workflow does:

| Step | Purpose |
|------|---------|
| `tool_call` | Execute a tool (see below) |
| `if` / `switch` | Branch on a condition or value |
| `foreach` | Loop over an array |
| `wait` | Pause for a duration |
| `wait_for_event` | Pause until an external event (payment, webhook) |
| `return` | Exit early with success or error |

**Tools** — 37 operations available in workflows:

| Category | Tools |
|----------|-------|
| Records | `create_record`, `update_record`, `delete_record`, `query_records`, `get_record`, `aggregate_records`, `query_record_logs`, `lock_record`, `unlock_record` |
| Files | `read_files`, `add_files_to_record`, `extract_file_data`, `rename_files`, `rename_files_in_record`, `remove_files_from_record`, `reorder_files_in_record` |
| Email | `gmail_send_email`, `gmail_query_emails`, `gmail_read_email`, `gmail_reply_email`, `outlook_send_email`, `outlook_query_emails`, `outlook_read_email`, `outlook_reply_email` |
| Documents | `generate_pdf_from_html_template`, `generate_pdf_from_pdf_form_template`, `generate_excel_from_template` |
| AI | `llm_generate_text` |
| Other | `send_notification`, `query_members`, `create_record_comments`, `get_record_comments`, `create_payment`, `web_scrape_url`, `csv_read_head`, `csv_read_header` |

**Expressions** — `{{ }}` wraps JavaScript expressions. Access trigger data via `trigger.{namespace}.*` and previous step results via `{step_id}.output.*`.

## Examples

### [Logistics](examples/logistics/)

[Freight Forwarding](examples/logistics/freight-forwarding/) · [Import & Export](examples/logistics/import-export/) · [Trucking & Transportation](examples/logistics/trucking-transportation/) · [Warehousing & Fulfillment](examples/logistics/warehousing-fulfillment/) · [Container Depot](examples/logistics/container-depot/) · [Customs Brokerage](examples/logistics/customs-brokerage/) · [Food Distribution](examples/logistics/food-distribution/) · [Pharma Distribution](examples/logistics/pharma-distribution/) · [Project Cargo](examples/logistics/project-cargo/) · [Sourcing Agent](examples/logistics/sourcing-agent/) · [Dangerous Goods](examples/logistics/dangerous-goods/)

### [Manufacturing](examples/manufacturing/)

[Production Operations](examples/manufacturing/production-operations/) · [Procurement & Sourcing](examples/manufacturing/procurement-sourcing/) · [Quality Control](examples/manufacturing/manufacturing-quality-control/) · [Maintenance & Facilities](examples/manufacturing/maintenance-facilities/)

### [Construction & Field](examples/construction-field/)

[Project Management](examples/construction-field/construction-project-management/) · [Field Service](examples/construction-field/field-service/) · [Equipment & Asset Tracking](examples/construction-field/equipment-asset-tracking/) · [Construction](examples/construction-field/construction/) · [Equipment Rental](examples/construction-field/equipment-rental/) · [HVAC / Field Service](examples/construction-field/hvac-field-service/) · [Event Rental](examples/construction-field/event-rental/) · [Auto Repair](examples/construction-field/auto-repair/)

### [Retail & Distribution](examples/retail-distribution/)

[Inventory & Replenishment](examples/retail-distribution/inventory-replenishment/) · [Multi-Location Operations](examples/retail-distribution/multi-location-operations/)

### [Professional Services](examples/professional-services/)

[Operations & Delivery](examples/professional-services/operations-delivery/) · [Finance & Back Office](examples/professional-services/finance-back-office/) · [Law Firms](examples/professional-services/law-firms/) · [Recruitment / Staffing](examples/professional-services/recruitment-staffing/)

### [Real Estate](examples/real-estate/)

[Property Management](examples/real-estate/property-management/) · [Development & Planning](examples/real-estate/development-planning/)

### [Healthcare & Clinics](examples/healthcare-clinics/)

[Clinic Operations](examples/healthcare-clinics/clinic-operations/) · [Medical Supply Management](examples/healthcare-clinics/medical-supply-management/) · [Veterinary](examples/healthcare-clinics/veterinary/)

### [Services](examples/services/)

[Spa & Aesthetics](examples/services/salon-spa/) · [Fitness / Gym](examples/services/gym-fitness/) · [Language Schools](examples/services/language-schools/) · [Laundry & Dry Cleaning](examples/services/laundry-dry-cleaning/) · [Photography Studio](examples/services/photography-studio/) · [Event Management](examples/services/event-management/)

### [Government & Public Sector](examples/public-sector/)

[Internal Operations](examples/public-sector/government-operations/) · [Infrastructure Projects](examples/public-sector/infrastructure-projects/)

## Using these examples

1. Open [Lotics](https://lotics.ai) and start a conversation with the AI assistant
2. Copy the example prompts from any workflow above
3. The assistant creates the tables, fields, and automation for you

→ [lotics.ai](https://lotics.ai)

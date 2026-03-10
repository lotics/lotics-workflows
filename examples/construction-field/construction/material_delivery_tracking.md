# Material Delivery Tracking

## The problem

Construction projects depend on materials arriving on time. A concrete pour scheduled for Tuesday requires rebar delivered Monday and forms in place by Friday. When a supplier emails a delivery confirmation or delay notice, someone must manually update the schedule, notify the site super, and adjust downstream tasks. With 10-20 deliveries per week across multiple projects, delays cascade silently -- crews show up to a site with nothing to install, costing $2,000-5,000 per wasted crew day.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Material Orders | material_name, supplier_email, project (link), expected_delivery_date, status, quantity, unit_cost | Ordered materials |
| Projects | project_name, site_super_email, schedule_status | Active construction projects |
| Schedule Tasks | project (link), task_name, start_date, depends_on_material (link), assigned_crew | Tasks dependent on material delivery |

## Example prompts

- "When we receive an email from a supplier about a delivery, use AI to extract the delivery date and material details, update the matching material order, and if the delivery is late, find all schedule tasks that depend on it and notify the site superintendent."
- "Automatically process supplier delivery emails -- match them to open material orders, update the status, and alert the site super if a delay will impact the schedule."

## Workflow

**Trigger:** When an Outlook email is received from a supplier

```json
[
  {
    "id": "parse_delivery_email",
    "type": "tool_call",
    "description": "Use LLM to extract delivery details from supplier email",
    "tool_name": "llm_generate_text",
    "input": {
      "prompt": "Extract the following from this supplier delivery email as JSON: material_name, delivery_date (ISO format), delivery_status (one of: confirmed, delayed, shipped, delivered), delay_reason (if applicable, else null), quantity.\n\nFrom: {{trigger.receive_outlook_email.from}}\nSubject: {{trigger.receive_outlook_email.subject}}\nBody: {{trigger.receive_outlook_email.body}}",
      "response_format": "json"
    }
  },
  {
    "id": "find_matching_order",
    "type": "tool_call",
    "description": "Find the material order matching this delivery",
    "tool_name": "query_records",
    "input": {
      "table_name": "Material Orders",
      "filters": {
        "supplier_email": "{{trigger.receive_outlook_email.from}}",
        "status": ["Ordered", "Shipped"]
      }
    }
  },
  {
    "id": "update_order",
    "type": "tool_call",
    "description": "Update the material order with delivery info",
    "tool_name": "update_record",
    "input": {
      "table_name": "Material Orders",
      "record_id": "{{find_matching_order.output.records[0].id}}",
      "data": {
        "status": "{{JSON.parse(parse_delivery_email.output.text).delivery_status === 'delivered' ? 'Delivered' : JSON.parse(parse_delivery_email.output.text).delivery_status === 'delayed' ? 'Delayed' : 'Shipped'}}",
        "expected_delivery_date": "{{JSON.parse(parse_delivery_email.output.text).delivery_date}}"
      }
    }
  },
  {
    "id": "check_delay",
    "type": "if",
    "description": "If the delivery is delayed, check for impacted schedule tasks",
    "condition": "{{JSON.parse(parse_delivery_email.output.text).delivery_status === 'delayed'}}",
    "then": [
      {
        "id": "find_dependent_tasks",
        "type": "tool_call",
        "description": "Find schedule tasks that depend on this material",
        "tool_name": "query_records",
        "input": {
          "table_name": "Schedule Tasks",
          "filters": {
            "depends_on_material": "{{find_matching_order.output.records[0].id}}"
          }
        }
      },
      {
        "id": "get_project",
        "type": "tool_call",
        "description": "Fetch the project to get site super contact",
        "tool_name": "get_record",
        "input": {
          "table_name": "Projects",
          "record_id": "{{find_matching_order.output.records[0].project}}"
        }
      },
      {
        "id": "alert_site_super",
        "type": "tool_call",
        "description": "Email the site superintendent about the delay and impacted tasks",
        "tool_name": "outlook_send_email",
        "input": {
          "to": "{{get_project.output.record.site_super_email}}",
          "subject": "Material Delay Alert: {{JSON.parse(parse_delivery_email.output.text).material_name}} — {{get_project.output.record.project_name}}",
          "body": "The delivery of {{JSON.parse(parse_delivery_email.output.text).material_name}} has been delayed.\n\nNew expected delivery: {{JSON.parse(parse_delivery_email.output.text).delivery_date}}\nReason: {{JSON.parse(parse_delivery_email.output.text).delay_reason}}\n\nImpacted schedule tasks:\n{{find_dependent_tasks.output.records.map(t => '- ' + t.task_name + ' (start date: ' + t.start_date + ', crew: ' + t.assigned_crew + ')').join('\\n')}}\n\nPlease review the schedule and reassign crews as needed."
        }
      },
      {
        "id": "add_delay_comment",
        "type": "tool_call",
        "description": "Log the delay on the material order",
        "tool_name": "create_record_comments",
        "input": {
          "table_name": "Material Orders",
          "record_id": "{{find_matching_order.output.records[0].id}}",
          "comment": "Supplier reported delay. New delivery date: {{JSON.parse(parse_delivery_email.output.text).delivery_date}}. Reason: {{JSON.parse(parse_delivery_email.output.text).delay_reason}}. {{find_dependent_tasks.output.records.length}} schedule tasks impacted."
        }
      }
    ],
    "else": []
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| Delay detection time | 1-2 days (manual email review) | Immediate on email receipt |
| Wasted crew days from material delays | 4-6 per month | Under 1 per month |
| Schedule update accuracy | Delayed and inconsistent | Real-time from supplier emails |

-> [Set up this workflow on Lotics](https://lotics.ai)

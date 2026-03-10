# Maintenance Work Order Routing and Assignment

## The problem

A beverage bottling plant has 120+ pieces of equipment across four production lines. When a machine breaks down, the operator calls or radios the maintenance office, and the maintenance planner manually creates a work order, looks up who is on shift, checks technician specializations, and assigns the job. This process takes 20-40 minutes per request, during which the line sits idle. With 8-12 unplanned breakdowns per week, the plant loses an average of 4 hours of production time weekly just on dispatching delays.

## Data model

| Table | Key fields | Purpose |
|-------|-----------|---------|
| Work Orders | wo_id, equipment_id, description, priority, status, assigned_to, requested_by, requested_at, completed_at | Maintenance job ticket |
| Equipment | equipment_id, name, line, location, equipment_type, criticality, last_maintenance | Asset registry |
| Technicians | tech_id, name, specializations, current_shift, status | Maintenance team roster |
| Work Order Log | log_id, wo_id, action, performed_by, timestamp, notes | Audit trail for work order activity |

## Example prompts

- "When an operator submits a maintenance request, look up the equipment criticality, find an available technician with the right specialization, assign the work order, and notify both the technician and the production supervisor."
- "Automatically route maintenance requests to the right technician based on equipment type and urgency."

## Workflow

**Trigger:** A maintenance request form is submitted by an operator

```json
[
  {
    "id": "get_request",
    "type": "tool_call",
    "description": "Fetch the submitted work order details",
    "tool_name": "get_record",
    "input": {
      "table_id": "work_orders",
      "record_id": "{{trigger.form_submitted.record_id}}"
    }
  },
  {
    "id": "get_equipment",
    "type": "tool_call",
    "description": "Look up the equipment details for criticality and type",
    "tool_name": "get_record",
    "input": {
      "table_id": "equipment",
      "record_id": "{{get_request.output.data.equipment_id}}"
    }
  },
  {
    "id": "find_technicians",
    "type": "tool_call",
    "description": "Find available technicians with the matching specialization",
    "tool_name": "query_records",
    "input": {
      "table_id": "technicians",
      "filters": {
        "specializations__contains": "{{get_equipment.output.data.equipment_type}}",
        "status": "Available"
      }
    }
  },
  {
    "id": "check_availability",
    "type": "if",
    "description": "Check if a qualified technician is available",
    "condition": "{{find_technicians.output.records.length > 0}}",
    "then": [
      {
        "id": "set_priority",
        "type": "switch",
        "description": "Set work order priority based on equipment criticality",
        "expression": "{{get_equipment.output.data.criticality}}",
        "cases": {
          "Critical": [
            {
              "id": "assign_critical",
              "type": "tool_call",
              "description": "Assign the work order with Emergency priority",
              "tool_name": "update_record",
              "input": {
                "table_id": "work_orders",
                "record_id": "{{trigger.form_submitted.record_id}}",
                "data": {
                  "priority": "Emergency",
                  "status": "Assigned",
                  "assigned_to": "{{find_technicians.output.records[0].tech_id}}"
                }
              }
            }
          ],
          "High": [
            {
              "id": "assign_high",
              "type": "tool_call",
              "description": "Assign the work order with High priority",
              "tool_name": "update_record",
              "input": {
                "table_id": "work_orders",
                "record_id": "{{trigger.form_submitted.record_id}}",
                "data": {
                  "priority": "High",
                  "status": "Assigned",
                  "assigned_to": "{{find_technicians.output.records[0].tech_id}}"
                }
              }
            }
          ],
          "default": [
            {
              "id": "assign_normal",
              "type": "tool_call",
              "description": "Assign the work order with Normal priority",
              "tool_name": "update_record",
              "input": {
                "table_id": "work_orders",
                "record_id": "{{trigger.form_submitted.record_id}}",
                "data": {
                  "priority": "Normal",
                  "status": "Assigned",
                  "assigned_to": "{{find_technicians.output.records[0].tech_id}}"
                }
              }
            }
          ]
        }
      },
      {
        "id": "update_tech_status",
        "type": "tool_call",
        "description": "Mark the assigned technician as busy",
        "tool_name": "update_record",
        "input": {
          "table_id": "technicians",
          "record_id": "{{find_technicians.output.records[0].record_id}}",
          "data": {
            "status": "On Job"
          }
        }
      },
      {
        "id": "notify_tech",
        "type": "tool_call",
        "description": "Notify the assigned technician with job details",
        "tool_name": "send_notification",
        "input": {
          "title": "New Work Order: {{get_request.output.data.wo_id}}",
          "message": "Equipment: {{get_equipment.output.data.name}} ({{get_equipment.output.data.location}})\nLine: {{get_equipment.output.data.line}}\nIssue: {{get_request.output.data.description}}\nPriority: {{get_equipment.output.data.criticality}}"
        }
      },
      {
        "id": "log_assignment",
        "type": "tool_call",
        "description": "Log the assignment action for audit",
        "tool_name": "create_record",
        "input": {
          "table_id": "work_order_log",
          "data": {
            "wo_id": "{{get_request.output.data.wo_id}}",
            "action": "Auto-assigned",
            "performed_by": "System",
            "notes": "Assigned to {{find_technicians.output.records[0].name}} based on specialization match and availability."
          }
        }
      }
    ],
    "else": [
      {
        "id": "mark_unassigned",
        "type": "tool_call",
        "description": "Set work order to Pending Assignment",
        "tool_name": "update_record",
        "input": {
          "table_id": "work_orders",
          "record_id": "{{trigger.form_submitted.record_id}}",
          "data": {
            "status": "Pending Assignment"
          }
        }
      },
      {
        "id": "notify_planner",
        "type": "tool_call",
        "description": "Alert the maintenance planner that no qualified technician is available",
        "tool_name": "send_notification",
        "input": {
          "title": "No technician available for WO {{get_request.output.data.wo_id}}",
          "message": "Equipment: {{get_equipment.output.data.name}} ({{get_equipment.output.data.equipment_type}}). No available technician with matching specialization. Manual assignment required."
        }
      }
    ]
  }
]
```

## Results

| Metric | Before | After |
|--------|--------|-------|
| Dispatch time per work order | 20-40 minutes | < 1 minute |
| Weekly production loss from dispatch delays | 4 hours | < 30 minutes |
| Misrouted work orders | 15% | < 2% |

-> [Set up this workflow on Lotics](https://lotics.ai)

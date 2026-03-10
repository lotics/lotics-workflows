# Production Order Tracking with Automatic Material Allocation

## The problem

A mid-size electronics assembler runs 80-120 production orders per week across three shifts. Planners manually check raw material availability in spreadsheets before releasing orders to the floor, causing 2-3 hour delays per order and frequent stock-outs discovered only after production has started. On average, 15% of orders stall mid-run because a component was double-allocated to another order.

## Data model

| Table | Key fields | Purpose |
|-------|-----------|---------|
| Production Orders | order_id, product_sku, quantity, status, priority, scheduled_start, line_assignment | Tracks each manufacturing order from creation to completion |
| Bill of Materials | bom_id, product_sku, component_sku, qty_per_unit | Defines material requirements per finished product |
| Inventory Levels | component_sku, warehouse, qty_on_hand, qty_allocated, qty_available | Real-time stock position per component |
| Material Allocations | allocation_id, order_id, component_sku, qty_allocated, status | Links reserved materials to specific production orders |

## Example prompts

- "When a production order is set to Approved, check if all required materials are available. If yes, allocate them and move the order to Ready. If not, flag the missing components and notify the planner."
- "Automatically reserve raw materials for approved production orders and alert the planning team when any component is short."

## Workflow

**Trigger:** A production order's status changes to "Approved"

```json
[
  {
    "id": "get_order",
    "type": "tool_call",
    "description": "Fetch the full production order details",
    "tool_name": "get_record",
    "input": {
      "table_id": "production_orders",
      "record_id": "{{trigger.record_updated.record_id}}"
    }
  },
  {
    "id": "get_bom",
    "type": "tool_call",
    "description": "Look up all components required for this product",
    "tool_name": "query_records",
    "input": {
      "table_id": "bill_of_materials",
      "filters": {
        "product_sku": "{{get_order.output.data.product_sku}}"
      }
    }
  },
  {
    "id": "check_each_component",
    "type": "foreach",
    "description": "Check inventory availability for each BOM component",
    "items": "{{get_bom.output.records}}",
    "steps": [
      {
        "id": "check_stock",
        "type": "tool_call",
        "description": "Query available inventory for this component",
        "tool_name": "query_records",
        "input": {
          "table_id": "inventory_levels",
          "filters": {
            "component_sku": "{{check_each_component.item.component_sku}}"
          }
        }
      },
      {
        "id": "evaluate_availability",
        "type": "if",
        "description": "Check if available quantity covers the required amount",
        "condition": "{{check_stock.output.records[0].qty_available >= check_each_component.item.qty_per_unit * get_order.output.data.quantity}}",
        "then": [
          {
            "id": "allocate_material",
            "type": "tool_call",
            "description": "Create a material allocation record for this component",
            "tool_name": "create_record",
            "input": {
              "table_id": "material_allocations",
              "data": {
                "order_id": "{{get_order.output.data.order_id}}",
                "component_sku": "{{check_each_component.item.component_sku}}",
                "qty_allocated": "{{check_each_component.item.qty_per_unit * get_order.output.data.quantity}}",
                "status": "Reserved"
              }
            }
          },
          {
            "id": "decrement_available",
            "type": "tool_call",
            "description": "Update inventory to reflect the new allocation",
            "tool_name": "update_record",
            "input": {
              "table_id": "inventory_levels",
              "record_id": "{{check_stock.output.records[0].record_id}}",
              "data": {
                "qty_allocated": "{{check_stock.output.records[0].qty_allocated + check_each_component.item.qty_per_unit * get_order.output.data.quantity}}"
              }
            }
          }
        ],
        "else": [
          {
            "id": "flag_shortage",
            "type": "tool_call",
            "description": "Notify the planner about the material shortage",
            "tool_name": "send_notification",
            "input": {
              "title": "Material Shortage on Order {{get_order.output.data.order_id}}",
              "message": "Component {{check_each_component.item.component_sku}} is short. Required: {{check_each_component.item.qty_per_unit * get_order.output.data.quantity}}, Available: {{check_stock.output.records[0].qty_available}}."
            }
          }
        ]
      }
    ]
  },
  {
    "id": "update_order_status",
    "type": "tool_call",
    "description": "Move the production order to Ready status",
    "tool_name": "update_record",
    "input": {
      "table_id": "production_orders",
      "record_id": "{{trigger.record_updated.record_id}}",
      "data": {
        "status": "Ready"
      }
    }
  }
]
```

## Results

| Metric | Before | After |
|--------|--------|-------|
| Order release delay | 2-3 hours | < 5 minutes |
| Mid-run stock-outs | 15% of orders | < 1% of orders |
| Planner time on material checks | 12 hrs/week | 1 hr/week (exceptions only) |

-> [Set up this workflow on Lotics](https://lotics.ai)

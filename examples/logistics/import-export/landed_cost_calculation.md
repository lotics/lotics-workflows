# Landed Cost Calculation on Shipment Arrival

## The problem

Import teams need accurate landed costs — purchase price plus freight, insurance, customs duties, and local charges — before goods hit the warehouse. Finance typically waits 2-3 weeks post-arrival to reconcile all cost components, delaying inventory valuation and making it impossible to price products accurately. Errors in duty rate application alone cause $3,000-8,000/month in over- or under-payments.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Shipments | shipment_ref, supplier_id, incoterm, currency, total_fob, status | Import shipment master |
| Cost Lines | shipment_id, cost_type, amount, currency, vendor | Individual cost components |
| Products | shipment_id, hs_code, product_name, qty, unit_price, duty_rate | Line items with tariff data |
| Landed Cost Summary | shipment_id, total_cost, cost_per_unit, calculated_at | Final rolled-up cost |

## Example prompts

- "When a shipment status changes to 'arrived', calculate the total landed cost by summing all cost lines and duties, then create a summary record."
- "Automatically compute landed cost per unit when goods arrive and notify finance."

## Workflow

**Trigger:** Shipment record updated with status changing to "arrived".

```json
[
  {
    "id": "get_shipment",
    "type": "tool_call",
    "description": "Fetch the arrived shipment details",
    "tool_name": "get_record",
    "input": {
      "table_id": "shipments",
      "record_id": "{{trigger.record_updated.record_id}}"
    }
  },
  {
    "id": "get_cost_lines",
    "type": "tool_call",
    "description": "Query all cost lines (freight, insurance, handling, etc.) for this shipment",
    "tool_name": "query_records",
    "input": {
      "table_id": "cost_lines",
      "filter": {
        "field": "shipment_id",
        "operator": "equals",
        "value": "{{trigger.record_updated.record_id}}"
      }
    }
  },
  {
    "id": "get_products",
    "type": "tool_call",
    "description": "Query all product lines to calculate duty amounts",
    "tool_name": "query_records",
    "input": {
      "table_id": "products",
      "filter": {
        "field": "shipment_id",
        "operator": "equals",
        "value": "{{trigger.record_updated.record_id}}"
      }
    }
  },
  {
    "id": "aggregate_costs",
    "type": "tool_call",
    "description": "Sum all cost lines for this shipment",
    "tool_name": "aggregate_records",
    "input": {
      "table_id": "cost_lines",
      "filter": {
        "field": "shipment_id",
        "operator": "equals",
        "value": "{{trigger.record_updated.record_id}}"
      },
      "aggregate": {
        "field": "amount",
        "function": "sum"
      }
    }
  },
  {
    "id": "calculate_landed",
    "type": "tool_call",
    "description": "Use LLM to calculate total landed cost including duties from product duty rates",
    "tool_name": "llm_generate_text",
    "input": {
      "prompt": "Calculate landed cost. FOB total: {{get_shipment.output.data.total_fob}}. Additional costs (freight, insurance, handling): {{aggregate_costs.output.value}}. Products with duty rates: {{JSON.stringify(get_products.output.data.map(function(p) { return { name: p.product_name, qty: p.qty, unit_price: p.unit_price, duty_rate: p.duty_rate } }))}}. Return JSON with fields: total_duties, total_landed_cost, total_units, cost_per_unit. Only return the JSON object, nothing else."
    }
  },
  {
    "id": "create_summary",
    "type": "tool_call",
    "description": "Create a landed cost summary record",
    "tool_name": "create_record",
    "input": {
      "table_id": "landed_cost_summary",
      "data": {
        "shipment_id": "{{get_shipment.output.data.id}}",
        "total_cost": "{{JSON.parse(calculate_landed.output.text).total_landed_cost}}",
        "cost_per_unit": "{{JSON.parse(calculate_landed.output.text).cost_per_unit}}",
        "calculated_at": "{{NOW()}}"
      }
    }
  },
  {
    "id": "notify_finance",
    "type": "tool_call",
    "description": "Notify finance team that landed cost is ready",
    "tool_name": "send_notification",
    "input": {
      "message": "Landed cost calculated for shipment {{get_shipment.output.data.shipment_ref}}: total {{JSON.parse(calculate_landed.output.text).total_landed_cost}}, per-unit {{JSON.parse(calculate_landed.output.text).cost_per_unit}}.",
      "record_id": "{{create_summary.output.data.id}}"
    }
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| Time to produce landed cost | 2-3 weeks | Same day as arrival |
| Duty calculation errors | $3,000-8,000/month | Under $500/month |
| Inventory valuation delay | 2-3 weeks | 0 days |

→ [Set up this workflow on Lotics](https://lotics.ai)

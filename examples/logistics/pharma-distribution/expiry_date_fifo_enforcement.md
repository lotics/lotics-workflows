# Expiry Date FEFO Enforcement and Short-Dated Stock Alert

## The problem

Pharmaceutical distributors must ship product using FEFO (First Expiry, First Out) to ensure customers receive stock with maximum remaining shelf life. GDP mandates a minimum remaining shelf life at time of delivery — typically 60-75% of total shelf life. When warehouse staff pick the wrong batch or short-dated stock accumulates unnoticed, the distributor either ships non-compliant product (risking audit findings and customer rejection) or writes off expired inventory. A distributor with 3,000+ pharma SKUs writes off $20,000-40,000/quarter in expired stock.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Inventory Batches | sku, product_name, batch_number, expiry_date, qty_on_hand, total_shelf_life_days, status | Batch-level inventory |
| Products | sku, product_name, min_remaining_shelf_life_pct | Shelf life rules |
| Short Date Alerts | batch_id, sku, expiry_date, remaining_shelf_life_pct, qty_on_hand, action_required, status | Alerts for at-risk stock |

## Example prompts

- "Every morning, scan all pharma inventory for batches below the minimum remaining shelf life threshold and create alerts with recommended actions."
- "Daily FEFO compliance check — flag batches that can no longer be shipped and alert the inventory team."

## Workflow

**Trigger:** Recurring daily schedule at 05:00 UTC.

```json
[
  {
    "id": "get_all_batches",
    "type": "tool_call",
    "description": "Query all active inventory batches with stock on hand",
    "tool_name": "query_records",
    "input": {
      "table_id": "inventory_batches",
      "filter": {
        "and": [
          { "field": "qty_on_hand", "operator": "greater_than", "value": 0 },
          { "field": "status", "operator": "equals", "value": "available" }
        ]
      }
    }
  },
  {
    "id": "evaluate_shelf_life",
    "type": "tool_call",
    "description": "Calculate remaining shelf life percentage for each batch and identify non-compliant stock",
    "tool_name": "llm_generate_text",
    "input": {
      "prompt": "Evaluate FEFO compliance. Current date: {{NOW()}}. Inventory batches: {{JSON.stringify(get_all_batches.output.data.map(function(b) { return { id: b.id, sku: b.sku, product_name: b.product_name, batch_number: b.batch_number, expiry_date: b.expiry_date, total_shelf_life_days: b.total_shelf_life_days, qty_on_hand: b.qty_on_hand } }))}}. For each batch, calculate remaining_shelf_life_pct = (days until expiry / total_shelf_life_days) * 100. Flag batches where remaining_shelf_life_pct < 50 (below shippable threshold). Return JSON: { \"at_risk_batches\": [{ \"id\": string, \"sku\": string, \"product_name\": string, \"batch_number\": string, \"expiry_date\": string, \"remaining_pct\": number, \"qty_on_hand\": number, \"action\": \"discount_transfer\" | \"return_to_manufacturer\" | \"destroy\" }], \"total_at_risk_value_estimate\": number }. Only return JSON."
    }
  },
  {
    "id": "check_any_at_risk",
    "type": "if",
    "description": "Only proceed if there are at-risk batches",
    "condition": "{{JSON.parse(evaluate_shelf_life.output.text).at_risk_batches.length > 0}}",
    "then": [
      {
        "id": "create_alerts",
        "type": "foreach",
        "description": "Create short-date alert records for each at-risk batch",
        "items": "{{JSON.parse(evaluate_shelf_life.output.text).at_risk_batches}}",
        "steps": [
          {
            "id": "create_alert",
            "type": "tool_call",
            "description": "Create a short-date alert record",
            "tool_name": "create_record",
            "input": {
              "table_id": "short_date_alerts",
              "data": {
                "batch_id": "{{create_alerts.item.id}}",
                "sku": "{{create_alerts.item.sku}}",
                "expiry_date": "{{create_alerts.item.expiry_date}}",
                "remaining_shelf_life_pct": "{{create_alerts.item.remaining_pct}}",
                "qty_on_hand": "{{create_alerts.item.qty_on_hand}}",
                "action_required": "{{create_alerts.item.action}}",
                "status": "open"
              }
            }
          }
        ]
      },
      {
        "id": "notify_inventory_team",
        "type": "tool_call",
        "description": "Send a summary notification to the inventory team",
        "tool_name": "send_notification",
        "input": {
          "message": "FEFO Alert: {{JSON.parse(evaluate_shelf_life.output.text).at_risk_batches.length}} batch(es) below shippable shelf life threshold. Estimated at-risk value: ${{JSON.parse(evaluate_shelf_life.output.text).total_at_risk_value_estimate}}. Review short-date alerts and take action today."
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
| Quarterly expired stock write-off | $20,000-40,000 | Under $5,000 |
| Shipments rejected for short shelf life | 4-8/month | 0-1/month |
| GDP FEFO audit non-conformances | 3-5 per audit | 0 |

→ [Set up this workflow on Lotics](https://lotics.ai)

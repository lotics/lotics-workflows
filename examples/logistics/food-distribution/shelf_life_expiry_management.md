# Shelf Life and Expiry Date Management

## The problem

Food distributors carry products with shelf lives ranging from 3 days (fresh produce) to 12 months (frozen). When short-dated stock sits unnoticed, it either ships to a customer who rejects it (costing the delivery round-trip and a credit note) or gets written off as waste. A mid-size distributor carrying 2,000+ SKUs writes off $8,000-15,000/month in expired or near-expired inventory because nobody checked batch expiry dates until it was too late.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Inventory Batches | sku, product_name, batch_number, expiry_date, qty_on_hand, location, status | Batch-level stock with expiry |
| Clearance Actions | batch_id, action_type, discount_pct, new_price, status, created_at | Markdown or donation actions |
| Products | sku, product_name, category, min_shelf_life_days | Product master with shelf life rules |

## Example prompts

- "Every morning, find inventory batches expiring within 5 days and create clearance actions — discount for sale or flag for donation."
- "Daily scan for short-dated stock and trigger markdown or write-off decisions automatically."

## Workflow

**Trigger:** Recurring daily schedule at 06:00 UTC.

```json
[
  {
    "id": "find_expiring_batches",
    "type": "tool_call",
    "description": "Query inventory batches expiring within 5 days that still have stock",
    "tool_name": "query_records",
    "input": {
      "table_id": "inventory_batches",
      "filter": {
        "and": [
          { "field": "expiry_date", "operator": "less_than_or_equals", "value": "{{DATE_ADD(NOW(), 5, 'days')}}" },
          { "field": "expiry_date", "operator": "greater_than", "value": "{{NOW()}}" },
          { "field": "qty_on_hand", "operator": "greater_than", "value": 0 },
          { "field": "status", "operator": "equals", "value": "available" }
        ]
      }
    }
  },
  {
    "id": "process_expiring",
    "type": "foreach",
    "description": "Determine and create the appropriate clearance action for each batch",
    "items": "{{find_expiring_batches.output.data}}",
    "steps": [
      {
        "id": "decide_action",
        "type": "switch",
        "description": "Choose clearance action based on days until expiry",
        "value": "{{find_expiring_batches.output.data.indexOf(process_expiring.item)}}",
        "cases": {},
        "default": [
          {
            "id": "evaluate_days",
            "type": "if",
            "description": "If more than 2 days left, discount for quick sale; otherwise flag for donation",
            "condition": "{{DATE_DIFF(process_expiring.item.expiry_date, NOW(), 'days') > 2}}",
            "then": [
              {
                "id": "create_discount_action",
                "type": "tool_call",
                "description": "Create a clearance discount action",
                "tool_name": "create_record",
                "input": {
                  "table_id": "clearance_actions",
                  "data": {
                    "batch_id": "{{process_expiring.item.id}}",
                    "action_type": "discount",
                    "discount_pct": 30,
                    "status": "pending",
                    "created_at": "{{NOW()}}"
                  }
                }
              },
              {
                "id": "update_batch_clearance",
                "type": "tool_call",
                "description": "Mark batch as on clearance",
                "tool_name": "update_record",
                "input": {
                  "table_id": "inventory_batches",
                  "record_id": "{{process_expiring.item.id}}",
                  "data": {
                    "status": "clearance"
                  }
                }
              }
            ],
            "else": [
              {
                "id": "create_donation_action",
                "type": "tool_call",
                "description": "Flag batch for food bank donation",
                "tool_name": "create_record",
                "input": {
                  "table_id": "clearance_actions",
                  "data": {
                    "batch_id": "{{process_expiring.item.id}}",
                    "action_type": "donation",
                    "status": "pending",
                    "created_at": "{{NOW()}}"
                  }
                }
              },
              {
                "id": "update_batch_donation",
                "type": "tool_call",
                "description": "Mark batch for donation",
                "tool_name": "update_record",
                "input": {
                  "table_id": "inventory_batches",
                  "record_id": "{{process_expiring.item.id}}",
                  "data": {
                    "status": "pending_donation"
                  }
                }
              }
            ]
          }
        ]
      }
    ]
  },
  {
    "id": "summary_notification",
    "type": "if",
    "description": "Send summary if any batches were flagged",
    "condition": "{{find_expiring_batches.output.data.length > 0}}",
    "then": [
      {
        "id": "notify_team",
        "type": "tool_call",
        "description": "Notify inventory team about expiring batches",
        "tool_name": "send_notification",
        "input": {
          "message": "Daily expiry scan: {{find_expiring_batches.output.data.length}} batch(es) expiring within 5 days flagged for clearance or donation. Review and action today."
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
| Monthly expired inventory write-off | $8,000-15,000 | Under $2,000 |
| Customer rejections for short-dated product | 10-15/month | 1-2/month |
| Products recovered via clearance/donation | 20-30% | 80-90% |

→ [Set up this workflow on Lotics](https://lotics.ai)

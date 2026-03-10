# Cross-Location Pricing Compliance Audit

## The problem

A retail chain with 22 stores enforces centralized pricing — every location should charge the same price for the same SKU unless a regional promotion is active. In practice, store managers occasionally override prices at the POS, apply unauthorized discounts, or fail to update shelf tags after a price change. A spot-check audit last quarter found 8% of sampled SKUs had pricing inconsistencies across locations, resulting in margin erosion on high-volume items and customer complaints when prices differ between nearby stores.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Master Prices | `sku`, `product_name`, `standard_price`, `effective_date`, `category` | Centrally managed price list |
| Branch Prices | `branch_id`, `sku`, `local_price`, `last_verified_at`, `source` | Actual prices observed or reported at each location |
| Active Promotions | `promo_id`, `sku`, `promo_price`, `start_date`, `end_date`, `applicable_branches` | Authorized temporary price overrides |
| Price Violations | `violation_id`, `branch_id`, `sku`, `expected_price`, `actual_price`, `variance_pct`, `status`, `reported_at` | Logged pricing discrepancies |
| Branches | `branch_id`, `branch_name`, `manager_email`, `region` | Store directory |

## Example prompts

- "Every Monday morning, compare each branch's reported prices against the master price list, account for active promotions, and flag any unauthorized price differences to the branch manager."
- "Run a weekly pricing compliance audit across all locations — identify SKUs where the local price doesn't match the master price or an active promo, and generate a violation report."

## Workflow

**Trigger:** `recurring_schedule` — runs every Monday at 7:00 AM.

```json
[
  {
    "id": "get_master_prices",
    "type": "tool_call",
    "description": "Fetch the current master price list",
    "tool_name": "query_records",
    "input": {
      "table": "Master Prices"
    }
  },
  {
    "id": "get_branch_prices",
    "type": "tool_call",
    "description": "Fetch all reported branch-level prices",
    "tool_name": "query_records",
    "input": {
      "table": "Branch Prices"
    }
  },
  {
    "id": "get_active_promos",
    "type": "tool_call",
    "description": "Fetch all currently active promotions to exclude authorized overrides",
    "tool_name": "query_records",
    "input": {
      "table": "Active Promotions",
      "filters": {
        "end_date_after": "{{ new Date().toISOString().split('T')[0] }}"
      }
    }
  },
  {
    "id": "analyze_compliance",
    "type": "tool_call",
    "description": "Use LLM to identify pricing violations excluding authorized promotions",
    "tool_name": "llm_generate_text",
    "input": {
      "prompt": "Compare branch prices against master prices. Branch prices: {{ JSON.stringify(get_branch_prices.output.records) }}. Master prices: {{ JSON.stringify(get_master_prices.output.records) }}. Active promotions: {{ JSON.stringify(get_active_promos.output.records) }}. Return a JSON array of violations where the local price differs from the master price AND the difference is not explained by an active promotion for that branch. Each violation should include branch_id, sku, expected_price, actual_price, and variance_pct."
    }
  },
  {
    "id": "get_branches",
    "type": "tool_call",
    "description": "Fetch branch details for manager contact information",
    "tool_name": "query_records",
    "input": {
      "table": "Branches"
    }
  },
  {
    "id": "log_violations",
    "type": "foreach",
    "description": "Create a violation record for each identified pricing discrepancy",
    "items": "{{ JSON.parse(analyze_compliance.output.text) }}",
    "steps": [
      {
        "id": "create_violation",
        "type": "tool_call",
        "description": "Log the pricing violation for tracking and resolution",
        "tool_name": "create_record",
        "input": {
          "table": "Price Violations",
          "data": {
            "branch_id": "{{ item.branch_id }}",
            "sku": "{{ item.sku }}",
            "expected_price": "{{ item.expected_price }}",
            "actual_price": "{{ item.actual_price }}",
            "variance_pct": "{{ item.variance_pct }}",
            "status": "Open",
            "reported_at": "{{ new Date().toISOString() }}"
          }
        }
      }
    ]
  },
  {
    "id": "aggregate_by_branch",
    "type": "tool_call",
    "description": "Count open violations grouped by branch for the notification summary",
    "tool_name": "aggregate_records",
    "input": {
      "table": "Price Violations",
      "filters": {
        "status": "Open",
        "reported_at_after": "{{ new Date(Date.now() - 24 * 60 * 60 * 1000).toISOString() }}"
      },
      "group_by": "branch_id",
      "aggregation": "count"
    }
  },
  {
    "id": "notify_operations",
    "type": "tool_call",
    "description": "Send a compliance summary to the operations director",
    "tool_name": "send_notification",
    "input": {
      "message": "Weekly pricing audit complete. {{ JSON.parse(analyze_compliance.output.text).length }} violations found across locations. Violation records have been created and branch managers will be notified. Review the Price Violations table for details."
    }
  },
  {
    "id": "email_managers",
    "type": "foreach",
    "description": "Email each branch manager who has pricing violations",
    "items": "{{ aggregate_by_branch.output.records }}",
    "steps": [
      {
        "id": "find_manager",
        "type": "tool_call",
        "description": "Look up the branch manager's contact details",
        "tool_name": "query_records",
        "input": {
          "table": "Branches",
          "filters": {
            "branch_id": "{{ item.branch_id }}"
          }
        }
      },
      {
        "id": "send_manager_email",
        "type": "tool_call",
        "description": "Email the branch manager about their pricing violations",
        "tool_name": "outlook_send_email",
        "input": {
          "to": "{{ find_manager.output.records[0].manager_email }}",
          "subject": "Action Required: Pricing Compliance — {{ item.count }} Violation(s)",
          "body": "Hi,\n\nThe weekly pricing audit for {{ find_manager.output.records[0].branch_name }} found {{ item.count }} SKU(s) with prices that do not match the master price list or any active promotion.\n\nPlease review the violations in the system and correct shelf tags and POS prices by end of day Wednesday.\n\nThank you."
        }
      }
    ]
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| Pricing inconsistencies detected | Quarterly spot-checks | Weekly full audit |
| SKUs with unauthorized prices | 8% | < 1% |
| Time to resolve price discrepancy | 2-6 weeks | 3 days |
| Customer pricing complaints per month | 18 | 3 |

-> [Set up this workflow on Lotics](https://lotics.ai)

# Cycle Count Scheduling and Discrepancy Resolution

## The problem

A distributor with three warehouses runs full physical inventory counts twice a year, shutting down operations for two days each time. Between counts, shrinkage and mis-picks accumulate undetected — the average variance at count time is 6.2%. High-value and fast-moving SKUs need more frequent verification, but scheduling and tracking cycle counts manually across 12,000 SKUs is impractical.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Inventory | `sku`, `location`, `qty_on_hand`, `last_counted_at`, `abc_class` | Stock positions with ABC classification for count priority |
| Cycle Counts | `count_id`, `sku`, `location`, `expected_qty`, `counted_qty`, `variance`, `status`, `assigned_to`, `counted_at` | Individual count tasks and their results |
| Products | `sku`, `product_name`, `unit_cost`, `category` | Product master for value calculations |
| Adjustments | `adjustment_id`, `sku`, `location`, `qty_change`, `reason`, `approved_by` | Inventory adjustment log for audit trail |

## Example prompts

- "Every weekday morning, generate cycle count tasks for the top 20 highest-value SKUs that haven't been counted in the last 7 days, and assign them to the warehouse team."
- "Schedule daily cycle counts prioritized by ABC class — A items weekly, B items biweekly, C items monthly — and flag any count with more than 3% variance for manager review."

## Workflow

**Trigger:** `recurring_schedule` — runs every weekday at 6:00 AM before the warehouse shift starts.

```json
[
  {
    "id": "find_due_items",
    "type": "tool_call",
    "description": "Query A-class inventory items that have not been counted in the last 7 days",
    "tool_name": "query_records",
    "input": {
      "table": "Inventory",
      "filters": {
        "abc_class": "A",
        "last_counted_at_before": "{{ Date.now() - 7 * 24 * 60 * 60 * 1000 }}"
      },
      "sort": { "field": "last_counted_at", "direction": "asc" },
      "limit": 20
    }
  },
  {
    "id": "create_count_tasks",
    "type": "foreach",
    "description": "Create a cycle count task for each item due for counting",
    "items": "{{ find_due_items.output.records }}",
    "steps": [
      {
        "id": "create_count",
        "type": "tool_call",
        "description": "Create the cycle count record with expected quantity",
        "tool_name": "create_record",
        "input": {
          "table": "Cycle Counts",
          "data": {
            "sku": "{{ item.sku }}",
            "location": "{{ item.location }}",
            "expected_qty": "{{ item.qty_on_hand }}",
            "status": "Pending",
            "assigned_to": "warehouse-team"
          }
        }
      }
    ]
  },
  {
    "id": "notify_team",
    "type": "tool_call",
    "description": "Notify the warehouse team of today's count assignments",
    "tool_name": "send_notification",
    "input": {
      "message": "Daily cycle count: {{ find_due_items.output.records.length }} SKUs queued for counting today. Please complete all counts before end of shift."
    }
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| Inventory variance at year-end | 6.2% | 1.4% |
| Warehouse shutdown days per year | 4 | 0 |
| Time to detect shrinkage | Up to 6 months | 1-2 weeks |
| SKUs counted per month | ~2,000 (during full count) | ~1,800 (continuous) |

-> [Set up this workflow on Lotics](https://lotics.ai)

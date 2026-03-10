# Parts Reorder Alert

## The problem

Auto repair shops stock 500-2,000 unique parts. When a common part runs out (brake pads, oil filters, belts), the vehicle sits in the bay waiting for a next-day delivery, blocking the lift and delaying 2-3 other jobs. Parts managers do weekly manual counts, but fast-moving items deplete between counts. Shops lose an average of $800-1,200 per week in lost labor revenue from parts stockouts, plus $200-400 in rush delivery fees.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Parts Inventory | part_name, part_number, category, quantity_on_hand, reorder_point, reorder_quantity, unit_cost, supplier_email | Parts catalog with stock levels |
| Parts Usage | part (link), repair_order (link), quantity_used, used_date | Parts consumed on repairs |
| Purchase Orders | supplier_email, status, order_date, items, total_cost | Orders placed with suppliers |

## Example prompts

- "When a part is used on a repair order and the inventory drops below the reorder point, automatically create a purchase order for that part, email the supplier, and notify the parts manager."
- "Track parts usage in real time. When stock hits the reorder threshold, auto-generate a PO and send it to the supplier so we never run out of common parts."

## Workflow

**Trigger:** When a Parts Usage record is updated (quantity_used entered)

```json
[
  {
    "id": "get_part",
    "type": "tool_call",
    "description": "Fetch the part record to check current stock level",
    "tool_name": "get_record",
    "input": {
      "table_name": "Parts Inventory",
      "record_id": "{{trigger.record_updated.next_data.part}}"
    }
  },
  {
    "id": "update_stock",
    "type": "tool_call",
    "description": "Deduct the used quantity from inventory",
    "tool_name": "update_record",
    "input": {
      "table_name": "Parts Inventory",
      "record_id": "{{trigger.record_updated.next_data.part}}",
      "data": {
        "quantity_on_hand": "{{get_part.output.record.quantity_on_hand - trigger.record_updated.next_data.quantity_used}}"
      }
    }
  },
  {
    "id": "check_reorder",
    "type": "if",
    "description": "Check if stock has dropped to or below the reorder point",
    "condition": "{{(get_part.output.record.quantity_on_hand - trigger.record_updated.next_data.quantity_used) <= get_part.output.record.reorder_point}}",
    "then": [
      {
        "id": "create_po",
        "type": "tool_call",
        "description": "Create a purchase order for the part",
        "tool_name": "create_record",
        "input": {
          "table_name": "Purchase Orders",
          "data": {
            "supplier_email": "{{get_part.output.record.supplier_email}}",
            "status": "Sent",
            "order_date": "{{new Date().toISOString().split('T')[0]}}",
            "items": "{{get_part.output.record.part_number}} — {{get_part.output.record.part_name}} x {{get_part.output.record.reorder_quantity}}",
            "total_cost": "{{get_part.output.record.unit_cost * get_part.output.record.reorder_quantity}}"
          }
        }
      },
      {
        "id": "email_supplier",
        "type": "tool_call",
        "description": "Email the supplier with the purchase order",
        "tool_name": "gmail_send_email",
        "input": {
          "to": "{{get_part.output.record.supplier_email}}",
          "subject": "Purchase Order — {{get_part.output.record.part_number}} {{get_part.output.record.part_name}}",
          "body": "Please process the following order:\n\nPart: {{get_part.output.record.part_name}}\nPart Number: {{get_part.output.record.part_number}}\nQuantity: {{get_part.output.record.reorder_quantity}}\nUnit Cost: ${{get_part.output.record.unit_cost}}\nTotal: ${{get_part.output.record.unit_cost * get_part.output.record.reorder_quantity}}\n\nPlease confirm order receipt and expected delivery date.\n\nThank you."
        }
      },
      {
        "id": "notify_parts_manager",
        "type": "tool_call",
        "description": "Alert the parts manager about the auto-reorder",
        "tool_name": "send_notification",
        "input": {
          "message": "Auto-reorder triggered: {{get_part.output.record.part_name}} ({{get_part.output.record.part_number}}) dropped to {{get_part.output.record.quantity_on_hand - trigger.record_updated.next_data.quantity_used}} units (reorder point: {{get_part.output.record.reorder_point}}). PO for {{get_part.output.record.reorder_quantity}} units sent to supplier.",
          "record_id": "{{trigger.record_updated.next_data.part}}",
          "table_name": "Parts Inventory"
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
| Parts stockouts per week | 3-5 | Under 1 |
| Lost labor revenue from waiting on parts | $800-1,200/week | Under $200/week |
| Rush delivery fees | $200-400/week | Near zero |

-> [Set up this workflow on Lotics](https://lotics.ai)

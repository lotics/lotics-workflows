# Inbound Delivery Tracking and Goods Receipt

## The problem

A metal fabrication shop receives 30-40 shipments per week from suppliers. The receiving clerk logs deliveries on paper, then manually updates the PO system and notifies the requesting department. Partial shipments are common -- suppliers deliver 70-80% of an order with the remainder backordered -- but the PO system only shows "Open" or "Closed," so planners cannot tell how much has actually arrived. This leads to duplicate orders (costing $8,000/month on average) and production delays when planners assume material has arrived when it has not.

## Data model

| Table | Key fields | Purpose |
|-------|-----------|---------|
| Purchase Orders | po_id, supplier_id, status, total_qty_ordered, total_qty_received | Master PO record |
| PO Line Items | line_id, po_id, component, qty_ordered, qty_received, qty_outstanding | Per-component tracking |
| Goods Receipts | receipt_id, po_id, received_by, received_at, notes | Header for each delivery event |
| Receipt Line Items | receipt_line_id, receipt_id, line_id, qty_received, condition | Actual quantities received per component |

## Example prompts

- "When a goods receipt form is submitted, update the PO line items with the received quantities. If everything is fully received, close the PO. If it's a partial delivery, notify the buyer with what's still outstanding."
- "Track partial deliveries against purchase orders and automatically update outstanding quantities when goods are received."

## Workflow

**Trigger:** A goods receipt form is submitted by the receiving clerk

```json
[
  {
    "id": "get_receipt",
    "type": "tool_call",
    "description": "Fetch the submitted goods receipt record",
    "tool_name": "get_record",
    "input": {
      "table_id": "goods_receipts",
      "record_id": "{{trigger.form_submitted.record_id}}"
    }
  },
  {
    "id": "get_receipt_lines",
    "type": "tool_call",
    "description": "Query all line items on this receipt",
    "tool_name": "query_records",
    "input": {
      "table_id": "receipt_line_items",
      "filters": {
        "receipt_id": "{{get_receipt.output.data.receipt_id}}"
      }
    }
  },
  {
    "id": "update_each_line",
    "type": "foreach",
    "description": "Update each PO line item with the newly received quantity",
    "items": "{{get_receipt_lines.output.records}}",
    "steps": [
      {
        "id": "get_po_line",
        "type": "tool_call",
        "description": "Fetch the current PO line item to get existing received quantity",
        "tool_name": "get_record",
        "input": {
          "table_id": "po_line_items",
          "record_id": "{{update_each_line.item.line_id}}"
        }
      },
      {
        "id": "update_po_line",
        "type": "tool_call",
        "description": "Add the received quantity to the PO line item",
        "tool_name": "update_record",
        "input": {
          "table_id": "po_line_items",
          "record_id": "{{update_each_line.item.line_id}}",
          "data": {
            "qty_received": "{{get_po_line.output.data.qty_received + update_each_line.item.qty_received}}",
            "qty_outstanding": "{{get_po_line.output.data.qty_ordered - (get_po_line.output.data.qty_received + update_each_line.item.qty_received)}}"
          }
        }
      },
      {
        "id": "check_condition",
        "type": "if",
        "description": "Flag items received in damaged or non-conforming condition",
        "condition": "{{update_each_line.item.condition != 'Good'}}",
        "then": [
          {
            "id": "comment_damage",
            "type": "tool_call",
            "description": "Add a comment to the PO line item about the condition issue",
            "tool_name": "create_record_comments",
            "input": {
              "table_id": "po_line_items",
              "record_id": "{{update_each_line.item.line_id}}",
              "comment": "Received {{update_each_line.item.qty_received}} units in condition: {{update_each_line.item.condition}}. Receipt ID: {{get_receipt.output.data.receipt_id}}."
            }
          }
        ],
        "else": []
      }
    ]
  },
  {
    "id": "check_all_lines",
    "type": "tool_call",
    "description": "Query remaining outstanding items on this PO",
    "tool_name": "query_records",
    "input": {
      "table_id": "po_line_items",
      "filters": {
        "po_id": "{{get_receipt.output.data.po_id}}",
        "qty_outstanding__gt": 0
      }
    }
  },
  {
    "id": "close_or_notify",
    "type": "if",
    "description": "Close the PO if fully received, or notify the buyer of outstanding items",
    "condition": "{{check_all_lines.output.records.length == 0}}",
    "then": [
      {
        "id": "close_po",
        "type": "tool_call",
        "description": "Mark the PO as fully received and closed",
        "tool_name": "update_record",
        "input": {
          "table_id": "purchase_orders",
          "record_id": "{{get_receipt.output.data.po_id}}",
          "data": {
            "status": "Closed - Fully Received"
          }
        }
      }
    ],
    "else": [
      {
        "id": "update_partial",
        "type": "tool_call",
        "description": "Update PO status to partially received",
        "tool_name": "update_record",
        "input": {
          "table_id": "purchase_orders",
          "record_id": "{{get_receipt.output.data.po_id}}",
          "data": {
            "status": "Partially Received"
          }
        }
      },
      {
        "id": "notify_buyer",
        "type": "tool_call",
        "description": "Notify the buyer about outstanding items",
        "tool_name": "send_notification",
        "input": {
          "title": "Partial delivery on PO {{get_receipt.output.data.po_id}}",
          "message": "{{check_all_lines.output.records.length}} line items still outstanding after receipt {{get_receipt.output.data.receipt_id}}. Please follow up with the supplier on remaining quantities."
        }
      }
    ]
  }
]
```

## Results

| Metric | Before | After |
|--------|--------|-------|
| Duplicate orders from poor visibility | $8,000/month | < $500/month |
| Time to update PO after delivery | 1-2 hours | Immediate |
| Partial delivery visibility | None (binary Open/Closed) | Real-time per-line tracking |

-> [Set up this workflow on Lotics](https://lotics.ai)

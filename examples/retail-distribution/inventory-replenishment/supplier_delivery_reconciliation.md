# Supplier Delivery Reconciliation

## The problem

A retail chain receives 60-80 supplier deliveries per week across two distribution centers. Receiving staff scan items in, but reconciling the delivered quantities against purchase orders happens hours or days later — often by a different person cross-referencing paper packing slips with the PO system. Short-shipments go unnoticed until a store reports a stockout, and overcharges on invoices slip through because nobody catches the quantity mismatch in time. The company estimates $140K/year in unrecovered short-shipments.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Purchase Orders | `po_number`, `supplier`, `status`, `expected_delivery`, `total_cost` | Outbound orders awaiting delivery |
| PO Line Items | `po_number`, `sku`, `qty_ordered`, `qty_received`, `unit_cost`, `receive_status` | Expected vs. received quantities per SKU |
| Receiving Log | `receipt_id`, `po_number`, `sku`, `qty_scanned`, `received_at`, `dock_door` | Raw scan data from warehouse receiving |
| Suppliers | `supplier_name`, `email`, `account_rep`, `penalty_terms` | Supplier contacts and contract terms |
| Discrepancies | `discrepancy_id`, `po_number`, `sku`, `qty_expected`, `qty_received`, `variance`, `resolution`, `status` | Tracked mismatches for follow-up |

## Example prompts

- "When a receiving log entry is submitted, compare scanned quantities against the PO line items — if there's a shortage, log a discrepancy and email the supplier's account rep."
- "After warehouse staff submit a delivery receipt, auto-reconcile it with the purchase order. Flag short-shipments, update received quantities, and notify procurement of any variances."

## Workflow

**Trigger:** `form_submitted` — warehouse staff submit a delivery receipt form after scanning all items from a shipment.

```json
[
  {
    "id": "get_po_lines",
    "type": "tool_call",
    "description": "Fetch all line items for the purchase order being received",
    "tool_name": "query_records",
    "input": {
      "table": "PO Line Items",
      "filters": {
        "po_number": "{{ trigger.form_submitted.data.po_number }}"
      }
    }
  },
  {
    "id": "get_scanned_items",
    "type": "tool_call",
    "description": "Fetch all scanned receiving log entries for this PO",
    "tool_name": "query_records",
    "input": {
      "table": "Receiving Log",
      "filters": {
        "po_number": "{{ trigger.form_submitted.data.po_number }}"
      }
    }
  },
  {
    "id": "reconcile_lines",
    "type": "foreach",
    "description": "Compare each PO line item against scanned quantities and update receive status",
    "items": "{{ get_po_lines.output.records }}",
    "steps": [
      {
        "id": "find_scan_match",
        "type": "tool_call",
        "description": "Query the scanned quantity for this specific SKU",
        "tool_name": "query_records",
        "input": {
          "table": "Receiving Log",
          "filters": {
            "po_number": "{{ item.po_number }}",
            "sku": "{{ item.sku }}"
          }
        }
      },
      {
        "id": "update_line_received",
        "type": "tool_call",
        "description": "Update the PO line item with the received quantity",
        "tool_name": "update_record",
        "input": {
          "table": "PO Line Items",
          "record_id": "{{ item.id }}",
          "data": {
            "qty_received": "{{ find_scan_match.output.records[0].qty_scanned }}",
            "receive_status": "{{ find_scan_match.output.records[0].qty_scanned >= item.qty_ordered ? 'Complete' : 'Short' }}"
          }
        }
      },
      {
        "id": "check_shortage",
        "type": "if",
        "description": "If received quantity is less than ordered, log a discrepancy",
        "condition": "{{ find_scan_match.output.records[0].qty_scanned < item.qty_ordered }}",
        "then": [
          {
            "id": "log_discrepancy",
            "type": "tool_call",
            "description": "Create a discrepancy record for the short-shipment",
            "tool_name": "create_record",
            "input": {
              "table": "Discrepancies",
              "data": {
                "po_number": "{{ item.po_number }}",
                "sku": "{{ item.sku }}",
                "qty_expected": "{{ item.qty_ordered }}",
                "qty_received": "{{ find_scan_match.output.records[0].qty_scanned }}",
                "variance": "{{ item.qty_ordered - find_scan_match.output.records[0].qty_scanned }}",
                "status": "Open"
              }
            }
          }
        ],
        "else": []
      }
    ]
  },
  {
    "id": "get_supplier",
    "type": "tool_call",
    "description": "Fetch supplier contact details for notification",
    "tool_name": "query_records",
    "input": {
      "table": "Suppliers",
      "filters": {
        "supplier_name": "{{ trigger.form_submitted.data.supplier }}"
      }
    }
  },
  {
    "id": "check_any_discrepancies",
    "type": "tool_call",
    "description": "Check whether any discrepancies were logged for this PO",
    "tool_name": "query_records",
    "input": {
      "table": "Discrepancies",
      "filters": {
        "po_number": "{{ trigger.form_submitted.data.po_number }}",
        "status": "Open"
      }
    }
  },
  {
    "id": "notify_if_discrepancies",
    "type": "if",
    "description": "If discrepancies exist, email the supplier and notify procurement",
    "condition": "{{ check_any_discrepancies.output.records.length > 0 }}",
    "then": [
      {
        "id": "email_supplier_shortage",
        "type": "tool_call",
        "description": "Email the supplier about short-shipped items",
        "tool_name": "gmail_send_email",
        "input": {
          "to": "{{ get_supplier.output.records[0].email }}",
          "subject": "Short Shipment — PO {{ trigger.form_submitted.data.po_number }}",
          "body": "Hi {{ get_supplier.output.records[0].account_rep }},\n\nWe have received PO {{ trigger.form_submitted.data.po_number }} and identified {{ check_any_discrepancies.output.records.length }} line item(s) with quantity shortages.\n\nPlease review and advise on replacement shipment timing or credit.\n\nThank you."
        }
      },
      {
        "id": "notify_procurement",
        "type": "tool_call",
        "description": "Alert the procurement team about the delivery discrepancies",
        "tool_name": "send_notification",
        "input": {
          "message": "Delivery for PO {{ trigger.form_submitted.data.po_number }} from {{ trigger.form_submitted.data.supplier }} has {{ check_any_discrepancies.output.records.length }} short-shipped line item(s). Supplier has been notified. Review discrepancies and adjust inventory forecasts."
        }
      }
    ],
    "else": [
      {
        "id": "mark_po_complete",
        "type": "tool_call",
        "description": "Mark the purchase order as fully received",
        "tool_name": "update_record",
        "input": {
          "table": "Purchase Orders",
          "filters": {
            "po_number": "{{ trigger.form_submitted.data.po_number }}"
          },
          "data": {
            "status": "Received"
          }
        }
      }
    ]
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| Short-shipment recovery rate | 22% | 89% |
| Reconciliation time per delivery | 25 minutes | < 1 minute |
| Annual unrecovered shortages | $140,000 | $18,000 |
| Invoice disputes caught before payment | 15% | 94% |

-> [Set up this workflow on Lotics](https://lotics.ai)

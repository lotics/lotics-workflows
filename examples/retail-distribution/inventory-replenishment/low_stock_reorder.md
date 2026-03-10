# Low Stock Auto-Reorder

## The problem

A mid-size retailer with 4,000+ SKUs manually checks stock levels in spreadsheets every Monday. By the time a buyer spots a low-stock item, it has often already sold out — causing 12-18% of weekly revenue to come from back-ordered or substituted products. Supplier lead times vary from 2 to 14 days, so reorder timing matters as much as reorder quantity.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Products | `sku`, `product_name`, `category`, `reorder_point`, `reorder_qty`, `unit_cost` | Master product catalog with replenishment thresholds |
| Inventory | `sku`, `location`, `qty_on_hand`, `qty_reserved`, `last_counted_at` | Real-time stock positions per warehouse |
| Purchase Orders | `po_number`, `supplier`, `status`, `order_date`, `expected_delivery`, `total_cost` | Outbound orders to suppliers |
| PO Line Items | `po_number`, `sku`, `qty_ordered`, `unit_cost`, `qty_received` | Individual items within a purchase order |
| Suppliers | `supplier_name`, `lead_time_days`, `email`, `payment_terms` | Supplier directory and performance data |

## Example prompts

- "When any product's on-hand stock drops below its reorder point, automatically create a purchase order to the preferred supplier and email them the PO."
- "Set up a workflow that watches inventory updates, and whenever available quantity falls under the reorder threshold, generates a supplier PO and notifies our purchasing team."

## Workflow

**Trigger:** `record_updated` on the Inventory table — fires whenever `qty_on_hand` changes.

```json
[
  {
    "id": "check_below_reorder",
    "type": "if",
    "description": "Check whether available stock has fallen below the reorder point",
    "condition": "{{ trigger.record_updated.next_data.qty_on_hand - trigger.record_updated.next_data.qty_reserved < trigger.record_updated.next_data.reorder_point }}",
    "then": [
      {
        "id": "get_product",
        "type": "tool_call",
        "description": "Fetch product details including reorder quantity and preferred supplier",
        "tool_name": "query_records",
        "input": {
          "table": "Products",
          "filters": {
            "sku": "{{ trigger.record_updated.next_data.sku }}"
          }
        }
      },
      {
        "id": "get_supplier",
        "type": "tool_call",
        "description": "Fetch supplier contact and lead-time information",
        "tool_name": "query_records",
        "input": {
          "table": "Suppliers",
          "filters": {
            "supplier_name": "{{ get_product.output.records[0].supplier }}"
          }
        }
      },
      {
        "id": "create_po",
        "type": "tool_call",
        "description": "Create a new purchase order for the reorder quantity",
        "tool_name": "create_record",
        "input": {
          "table": "Purchase Orders",
          "data": {
            "supplier": "{{ get_supplier.output.records[0].supplier_name }}",
            "status": "Pending",
            "order_date": "{{ trigger.record_updated.timestamp }}",
            "total_cost": "{{ get_product.output.records[0].reorder_qty * get_product.output.records[0].unit_cost }}"
          }
        }
      },
      {
        "id": "create_line_item",
        "type": "tool_call",
        "description": "Add the SKU as a line item on the purchase order",
        "tool_name": "create_record",
        "input": {
          "table": "PO Line Items",
          "data": {
            "po_number": "{{ create_po.output.record.po_number }}",
            "sku": "{{ trigger.record_updated.next_data.sku }}",
            "qty_ordered": "{{ get_product.output.records[0].reorder_qty }}",
            "unit_cost": "{{ get_product.output.records[0].unit_cost }}"
          }
        }
      },
      {
        "id": "email_supplier",
        "type": "tool_call",
        "description": "Send the purchase order to the supplier via email",
        "tool_name": "gmail_send_email",
        "input": {
          "to": "{{ get_supplier.output.records[0].email }}",
          "subject": "Purchase Order {{ create_po.output.record.po_number }} — {{ get_product.output.records[0].product_name }}",
          "body": "Hi,\n\nPlease find below a new purchase order:\n\nPO #: {{ create_po.output.record.po_number }}\nSKU: {{ trigger.record_updated.next_data.sku }}\nProduct: {{ get_product.output.records[0].product_name }}\nQuantity: {{ get_product.output.records[0].reorder_qty }}\nUnit Cost: ${{ get_product.output.records[0].unit_cost }}\n\nPlease confirm receipt and expected ship date.\n\nThank you."
        }
      },
      {
        "id": "notify_purchasing",
        "type": "tool_call",
        "description": "Notify the purchasing team about the auto-generated PO",
        "tool_name": "send_notification",
        "input": {
          "message": "Auto-reorder triggered: PO {{ create_po.output.record.po_number }} created for {{ get_product.output.records[0].reorder_qty }} units of {{ get_product.output.records[0].product_name }} (SKU {{ trigger.record_updated.next_data.sku }}). Current stock: {{ trigger.record_updated.next_data.qty_on_hand }}."
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
| Stockout incidents per month | 35 | 6 |
| Average reorder response time | 3 days | < 5 minutes |
| Back-order revenue impact | 12-18% | < 3% |
| Manual PO creation time per week | 8 hours | 0 hours |

-> [Set up this workflow on Lotics](https://lotics.ai)

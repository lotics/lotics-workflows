# Outbound Packing Slip and Shipping Label Generation

## The problem

For every outbound order, the warehouse needs a packing slip listing items, quantities, and lot/serial numbers, plus a shipping label with the correct carrier and address. Packers currently print these from two different systems, manually cross-referencing order data. Mislabeled packages cause wrong-address deliveries (costing $25-50 each to reroute) and packing slip errors trigger customer complaints. At 200+ shipments per day, even a 2% error rate means 4+ mislabeled packages daily.

## Data model

| Table | Key fields | Purpose |
|---|---|---|
| Orders | order_number, customer_id, ship_method, ship_to_address, status | Sales order master |
| Order Lines | order_id, sku, product_name, qty, lot_number | Items per order |
| Customers | name, email, phone | Customer info |

## Example prompts

- "When an order status changes to 'packed', generate the packing slip PDF and notify shipping that it's ready for label and dispatch."
- "Automatically create packing slips when orders are packed."

## Workflow

**Trigger:** Order record updated with status changed to "packed".

```json
[
  {
    "id": "get_order",
    "type": "tool_call",
    "description": "Fetch the packed order details",
    "tool_name": "get_record",
    "input": {
      "table_id": "orders",
      "record_id": "{{trigger.record_updated.record_id}}"
    }
  },
  {
    "id": "get_order_lines",
    "type": "tool_call",
    "description": "Fetch all line items for the order",
    "tool_name": "query_records",
    "input": {
      "table_id": "order_lines",
      "filter": {
        "field": "order_id",
        "operator": "equals",
        "value": "{{trigger.record_updated.record_id}}"
      }
    }
  },
  {
    "id": "get_customer",
    "type": "tool_call",
    "description": "Fetch customer details for the packing slip",
    "tool_name": "get_record",
    "input": {
      "table_id": "customers",
      "record_id": "{{get_order.output.data.customer_id}}"
    }
  },
  {
    "id": "generate_packing_slip",
    "type": "tool_call",
    "description": "Generate the packing slip PDF with order and line item details",
    "tool_name": "generate_pdf_from_html_template",
    "input": {
      "template_id": "packing_slip_template",
      "data": {
        "order_number": "{{get_order.output.data.order_number}}",
        "customer_name": "{{get_customer.output.data.name}}",
        "ship_to_address": "{{get_order.output.data.ship_to_address}}",
        "ship_method": "{{get_order.output.data.ship_method}}",
        "line_items": "{{get_order_lines.output.data}}"
      }
    }
  },
  {
    "id": "attach_packing_slip",
    "type": "tool_call",
    "description": "Attach the packing slip to the order record",
    "tool_name": "add_files_to_record",
    "input": {
      "table_id": "orders",
      "record_id": "{{get_order.output.data.id}}",
      "file_ids": ["{{generate_packing_slip.output.file_id}}"]
    }
  },
  {
    "id": "notify_shipping",
    "type": "tool_call",
    "description": "Notify shipping team that packing slip is ready",
    "tool_name": "send_notification",
    "input": {
      "message": "Order {{get_order.output.data.order_number}} packed and packing slip generated. Ship via {{get_order.output.data.ship_method}} to {{get_customer.output.data.name}}.",
      "record_id": "{{get_order.output.data.id}}"
    }
  }
]
```

## Results

| Metric | Before | After |
|---|---|---|
| Packing slip preparation time | 3-5 min per order | Automatic |
| Mislabeled packages per day | 4+ | Under 1 |
| Rerouting costs | $3,000-5,000/month | Under $500/month |

→ [Set up this workflow on Lotics](https://lotics.ai)

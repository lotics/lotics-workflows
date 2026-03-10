# Purchase Order Approval and Dispatch

## The problem

A food packaging manufacturer processes 60+ purchase orders per week. POs under $5,000 need a procurement manager's sign-off; POs above $5,000 require additional VP approval. Today, POs sit in an email queue and approvers have no visibility into what is pending. Average approval cycle is 3.2 days, and 25% of POs miss the supplier's order cutoff window, pushing delivery out by a full week and disrupting production schedules.

## Data model

| Table | Key fields | Purpose |
|-------|-----------|---------|
| Purchase Orders | po_id, supplier_id, total_amount, line_items, requested_by, status, approved_by, approved_at | Tracks PO from draft to dispatched |
| PO Line Items | line_id, po_id, component, quantity, unit_price, delivery_date | Individual items on the PO |
| Suppliers | supplier_id, name, email, order_cutoff_day, payment_terms | Supplier directory with ordering windows |
| Approval Log | log_id, po_id, approver, decision, comments, decided_at | Audit trail of approval decisions |

## Example prompts

- "When a purchase order is submitted, route it for approval based on the total amount. Under $5,000 goes to the procurement manager; $5,000 and above goes to the VP first. Once approved, email the PO as a PDF to the supplier."
- "Automate PO approvals with tiered routing and send the confirmed order to the supplier automatically."

## Workflow

**Trigger:** A purchase order status changes to "Submitted"

```json
[
  {
    "id": "get_po",
    "type": "tool_call",
    "description": "Fetch the full purchase order record",
    "tool_name": "get_record",
    "input": {
      "table_id": "purchase_orders",
      "record_id": "{{trigger.record_updated.record_id}}"
    }
  },
  {
    "id": "route_approval",
    "type": "if",
    "description": "Route based on PO total: >= $5,000 needs VP approval first",
    "condition": "{{get_po.output.data.total_amount >= 5000}}",
    "then": [
      {
        "id": "notify_vp",
        "type": "tool_call",
        "description": "Send approval request to the VP",
        "tool_name": "send_notification",
        "input": {
          "title": "PO {{get_po.output.data.po_id}} requires your approval (${{get_po.output.data.total_amount}})",
          "message": "Purchase order for {{get_po.output.data.supplier_id}} totaling ${{get_po.output.data.total_amount}} needs VP approval. Requested by {{get_po.output.data.requested_by}}."
        }
      },
      {
        "id": "update_pending_vp",
        "type": "tool_call",
        "description": "Set status to Pending VP Approval",
        "tool_name": "update_record",
        "input": {
          "table_id": "purchase_orders",
          "record_id": "{{trigger.record_updated.record_id}}",
          "data": {
            "status": "Pending VP Approval"
          }
        }
      },
      {
        "id": "wait_vp",
        "type": "wait_for_event",
        "description": "Wait for the VP to approve or reject the PO",
        "event": "record_updated",
        "filters": {
          "record_id": "{{trigger.record_updated.record_id}}",
          "field": "status",
          "value_in": ["VP Approved", "Rejected"]
        }
      },
      {
        "id": "check_vp_decision",
        "type": "if",
        "description": "Check if the VP approved",
        "condition": "{{wait_vp.output.next_data.status == 'VP Approved'}}",
        "then": [
          {
            "id": "log_vp_approval",
            "type": "tool_call",
            "description": "Record the VP approval in the audit log",
            "tool_name": "create_record",
            "input": {
              "table_id": "approval_log",
              "data": {
                "po_id": "{{get_po.output.data.po_id}}",
                "approver": "VP",
                "decision": "Approved",
                "decided_at": "{{wait_vp.output.updated_at}}"
              }
            }
          }
        ],
        "else": [
          {
            "id": "log_vp_rejection",
            "type": "tool_call",
            "description": "Record the VP rejection and stop the workflow",
            "tool_name": "create_record",
            "input": {
              "table_id": "approval_log",
              "data": {
                "po_id": "{{get_po.output.data.po_id}}",
                "approver": "VP",
                "decision": "Rejected",
                "decided_at": "{{wait_vp.output.updated_at}}"
              }
            }
          },
          {
            "id": "stop_rejected",
            "type": "return",
            "description": "End workflow -- PO was rejected"
          }
        ]
      }
    ],
    "else": []
  },
  {
    "id": "generate_po_pdf",
    "type": "tool_call",
    "description": "Generate a formatted PDF of the purchase order",
    "tool_name": "generate_pdf_from_html_template",
    "input": {
      "template_id": "purchase_order_template",
      "data": {
        "po_id": "{{get_po.output.data.po_id}}",
        "supplier_id": "{{get_po.output.data.supplier_id}}",
        "line_items": "{{get_po.output.data.line_items}}",
        "total_amount": "{{get_po.output.data.total_amount}}"
      }
    }
  },
  {
    "id": "get_supplier",
    "type": "tool_call",
    "description": "Fetch the supplier's email address",
    "tool_name": "get_record",
    "input": {
      "table_id": "suppliers",
      "record_id": "{{get_po.output.data.supplier_id}}"
    }
  },
  {
    "id": "send_to_supplier",
    "type": "tool_call",
    "description": "Email the PO PDF to the supplier",
    "tool_name": "gmail_send_email",
    "input": {
      "to": "{{get_supplier.output.data.email}}",
      "subject": "Purchase Order {{get_po.output.data.po_id}}",
      "body": "Please find attached Purchase Order {{get_po.output.data.po_id}}. Kindly confirm receipt and expected delivery dates at your earliest convenience.",
      "attachments": ["{{generate_po_pdf.output.file}}"]
    }
  },
  {
    "id": "mark_dispatched",
    "type": "tool_call",
    "description": "Update PO status to Dispatched",
    "tool_name": "update_record",
    "input": {
      "table_id": "purchase_orders",
      "record_id": "{{trigger.record_updated.record_id}}",
      "data": {
        "status": "Dispatched"
      }
    }
  }
]
```

## Results

| Metric | Before | After |
|--------|--------|-------|
| Average PO approval cycle | 3.2 days | 4 hours |
| POs missing supplier cutoff | 25% | < 3% |
| Manual PO dispatch effort | 45 min/PO | 0 min (automated) |

-> [Set up this workflow on Lotics](https://lotics.ai)

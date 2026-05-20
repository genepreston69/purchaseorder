# Purchase Order System

Summit BHC purchase order request and approval system (Highland Leadership site).

| File                | Purpose                                                          |
| ------------------- | ---------------------------------------------------------------- |
| `index.html`        | PO request form — header info, line items, attachments, submit.  |
| `approval.html`     | Approver portal — review a PO and approve/reject with comments.  |
| `APPROVAL_FLOW.md`  | Backend spec — Power Automate flows and SharePoint data.         |

Approvals use a **sequential escalation chain**: higher PO totals require all
lower approvers in order (Dept Head → Controller → CFO → CEO). See
`APPROVAL_FLOW.md` for the routing thresholds and flow details.

Both pages are static HTML. Set the Power Automate HTTP trigger URLs in the
`CONFIGURATION` block at the top of each page's `<script>`. With the URLs left
blank, `approval.html` runs in demo mode against a built-in sample PO.

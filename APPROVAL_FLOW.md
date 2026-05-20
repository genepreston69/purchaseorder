# Purchase Order Approval Flow

Backend specification for the Summit BHC Purchase Order system. The two HTML
pages in this repo are the front end; this document describes the Power
Automate flows and SharePoint data they depend on.

```
  index.html  ──POST──▶  Flow 1: PO Intake  ──┐
                                              ├──▶  SharePoint lists + email
  approval.html ─GET──▶  Flow 2: Get PO  ─────┤
  approval.html ─POST─▶  Flow 3: PO Decision ─┘
```

Each page has a CONFIGURATION block at the top of its `<script>`:

| Page            | Constant       | Points to        |
| --------------- | -------------- | ---------------- |
| `index.html`    | `FLOW_URL`     | Flow 1 (Intake)  |
| `approval.html` | `PO_API_URL`   | Flow 2 (Get PO)  |
| `approval.html` | `DECISION_URL` | Flow 3 (Decision)|

If `approval.html`'s URLs are left blank, the page runs in demo mode with a
built-in sample PO so the UI can be reviewed without a backend.

## Approval routing — sequential escalation

A purchase order's approval chain is derived from its total. Each higher tier
*adds* the next approver; every approver in the chain must approve **in order**.

| PO Total            | Approval chain                              |
| ------------------- | ------------------------------------------- |
| `< $100`            | Auto-approved — no approvals                |
| `$100 – $399.99`    | Dept Head                                   |
| `$400 – $999.99`    | Dept Head → Controller                      |
| `$1,000 – $2,999.99`| Dept Head → Controller → CFO                |
| `$3,000 +`          | Dept Head → Controller → CFO → CEO          |

Reference logic (matches `getApprovalChain` in `index.html`):

```
chain = []
if total >= 100:  chain += ["Dept Head"]
if total >= 400:  chain += ["Controller"]
if total >= 1000: chain += ["CFO"]
if total >= 3000: chain += ["CEO"]
```

The **backend must recompute the total** by summing line items
(`Quantity × UnitCost`) rather than trusting the `POTotal` value the form
sends. The form total is for display only.

The Dept Head is resolved per department from the Department Approval Matrix.
Controller / CFO / CEO are organization-wide roles.

## SharePoint lists

**Purchase Orders** — one item per PO.

| Column                | Type            | Notes                                      |
| --------------------- | --------------- | ------------------------------------------ |
| Title                 | Single line     | `Vendor - date`                            |
| PONumber              | Single line     | Generated, e.g. `PO-2026-0042`             |
| Status                | Choice          | see Status reference below                 |
| RequestedDate         | Date            |                                            |
| Originator            | Person or text  |                                            |
| DepartmentName        | Single line     |                                            |
| DepartmentCode        | Single line     |                                            |
| Vendor                | Single line     |                                            |
| ExpenseCategory       | Choice          |                                            |
| PurchaseJustification | Multiline       |                                            |
| POTotal               | Currency        | Recomputed by Flow 1 from line items       |
| ApprovalChainJSON     | Multiline       | Serialized chain (roles, approvers, state) |
| CurrentStep           | Number          | Index of the active approver (0-based)     |
| StepToken             | Single line     | Random token for the current approver link |
| Attachments           | (list attach.)  | Quote / supporting documents               |

**PO Line Items** — one item per line, linked to a PO.

| Column          | Type        | Notes                          |
| --------------- | ----------- | ------------------------------ |
| Title           | Single line | Item description               |
| PONumber        | Lookup/text | Parent PO                      |
| ItemDescription | Single line |                                |
| ExpenseCategory | Choice      |                                |
| Quantity        | Number      |                                |
| UnitCost        | Currency    |                                |
| LineTotal       | Currency    | `Quantity × UnitCost`          |

**Department Approval Matrix** — maps a department to its Dept Head.

| Column          | Type        | Notes                          |
| --------------- | ----------- | ------------------------------ |
| Title           | Single line | Department name                |
| DepartmentCode  | Single line |                                |
| DeptHeadName    | Single line |                                |
| DeptHeadEmail   | Single line |                                |

Controller / CFO / CEO email addresses can live in a small **Approval Roles**
list or in flow variables.

`ApprovalChainJSON` stores the same shape `approval.html` renders:

```json
[
  { "role": "Dept Head", "approver": "Robert Chen", "email": "rchen@summitbhc.com",
    "status": "Approved", "decidedAt": "2026-05-19T15:42:00Z", "comments": "Approved." },
  { "role": "Controller", "approver": "Maria Delgado", "email": "mdelgado@summitbhc.com",
    "status": "Pending", "decidedAt": null, "comments": "" }
]
```

Step `status` values: `Waiting` (not yet routed), `Pending` (active approver),
`Approved`, `Rejected`.

## Flow 1 — PO Intake

**Trigger:** When an HTTP request is received (POST). Body = the JSON the form
in `index.html` sends.

**Steps:**

1. Parse JSON body.
2. Generate `PONumber` (`PO-{yyyy}-{nnnn}` from a counter or `Get items` count).
3. Compute the true total by summing `Quantity × UnitCost` over `LineItems`.
4. Build the approval chain from the total (escalation table above):
   - Look up the Dept Head from the Department Approval Matrix using
     `DepartmentName` / `DepartmentCode`.
   - Resolve Controller / CFO / CEO from the Approval Roles list.
   - Mark step 0 `Pending`, all later steps `Waiting`.
5. Create the **Purchase Orders** item (`Status` = `Submitted`, store
   `ApprovalChainJSON`, `CurrentStep` = 0, generate a random `StepToken`).
6. For each line item, create a **PO Line Items** item.
7. For each attachment, decode base64 and add it to the PO item's attachments.
8. **If total < $100:** set `Status` = `Auto-Approved`, email the originator a
   confirmation, then respond.
9. **Otherwise:** set `Status` = `Pending Dept Head Approval` and send the
   approval-request email (see templates) to the step-0 approver.
10. Respond `200` with `{ "PONumber": "PO-2026-0042" }`.

The approval email link points the approver at the portal:

```
https://<your-site>/approval.html?id={PONumber}&token={StepToken}
```

## Flow 2 — Get PO

**Trigger:** When an HTTP request is received (GET), query string `id` and
`token`.

**Steps:**

1. `Get items` from Purchase Orders filtered by `PONumber eq '{id}'`.
2. If not found, respond `404`.
3. Validate `token` against the PO's `StepToken` (rejects stale/forged links).
4. `Get items` from PO Line Items filtered by parent `PONumber`.
5. Build and respond `200` with the PO JSON the portal expects: header fields,
   `POTotal`, `LineItems`, `Attachments` (with download `url`), and
   `ApprovalChain` (parsed from `ApprovalChainJSON`).

## Flow 3 — PO Decision

**Trigger:** When an HTTP request is received (POST). Body is what
`approval.html` sends:

```json
{
  "PONumber": "PO-2026-0042",
  "step": "Controller",
  "decision": "Approved",
  "comments": "...",
  "approver": "Maria Delgado",
  "token": "...",
  "decidedAt": "2026-05-20T19:00:00.000Z"
}
```

**Steps:**

1. Look up the PO by `PONumber`; respond `404` if missing.
2. Validate `token` against `StepToken`. Reject mismatches with `403` —
   this is what enforces "only the assigned approver may act".
3. Confirm `step` matches the chain entry at `CurrentStep` and that its status
   is still `Pending` (guards against double submission).
4. Update that chain entry: `status` = decision, `comments`, `decidedAt`.
5. **If `decision` = `Rejected`:**
   - Set PO `Status` = `Rejected`.
   - Email the originator (and any prior approvers) with the rejection reason.
6. **If `decision` = `Approved`:**
   - If a later step exists: mark it `Pending`, increment `CurrentStep`,
     generate a fresh `StepToken`, set `Status` =
     `Pending {next role} Approval`, email the next approver.
   - If this was the last step: set `Status` = `Approved`, email the
     originator and Accounts Payable.
7. Save `ApprovalChainJSON` back to the PO item.
8. Respond `200`.

Regenerating `StepToken` on each hop means an old email link cannot be used to
act on a later step.

## Email templates

**Approval request → current approver**

> Subject: Action needed — PO {PONumber} ({Vendor}, {POTotal})
>
> {OriginatorName} submitted a purchase order that needs your approval as
> {role}.
> Department: {DepartmentName} · Total: {POTotal}
> Review and decide: {portal link with id + token}

**Approved (intermediate) → originator**

> Subject: PO {PONumber} — {role} approved
>
> {role} approved your purchase order. It is now with {next role} for the next
> approval.

**Fully approved → originator + Accounts Payable**

> Subject: PO {PONumber} approved
>
> All approvals are complete. This purchase order has been routed to Accounts
> Payable for processing.

**Rejected → originator**

> Subject: PO {PONumber} — rejected by {role}
>
> {role} rejected your purchase order.
> Reason: {comments}
> You may revise and resubmit a new request.

## Status reference

| Status                       | Meaning                                  |
| ---------------------------- | ---------------------------------------- |
| `Draft`                      | Not yet submitted                        |
| `Submitted`                  | Received, chain being assigned           |
| `Auto-Approved`              | Under $100, no approvals required        |
| `Pending {Role} Approval`    | Awaiting the named approver              |
| `Approved`                   | All approvers approved                   |
| `Rejected`                   | An approver rejected; workflow stopped   |

## Alternative — built-in Approvals connector

Instead of the custom portal, Flows 2 and 3 can be replaced with the Power
Automate **Approvals** connector ("Start and wait for an approval"), looping
one approval per chain step. This gives approve/reject inside Outlook and the
Power Automate app with no custom page. The custom portal is used here because
it shows the full PO, the live approval chain, and prior approvers' comments on
one screen — context the connector's compact card does not provide.

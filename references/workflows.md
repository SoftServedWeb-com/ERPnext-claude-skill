# Frappe Workflows

The Workflow DocType attaches a state machine to another DocType. Once active, the target DocType gets a `workflow_state` field, an "Action" dropdown on the form, and state-aware permission checks.

## Table of contents
- [Schema](#schema)
- [Building a workflow](#building-a-workflow)
- [Conditional routing](#conditional-routing)
- [Programmatic transitions](#programmatic-transitions)
- [Multi-step approval chains](#multi-step-approval-chains)
- [JSON fixture, end to end](#json-fixture-end-to-end)
- [Common pitfalls](#common-pitfalls)

## Schema

The Workflow DocType has these fields:

| Field | Purpose |
|-------|---------|
| `workflow_name` | Display name (also the `name` / primary key) |
| `document_type` | Target DocType (e.g., "Purchase Order") |
| `is_active` | 1 or 0. Only one active workflow per DocType. |
| `override_status` | If 1, the workflow's state replaces ERPnext's built-in `status` field. Usually leave 0. |
| `workflow_state_field` | Defaults to `workflow_state`. Frappe auto-creates this Custom Field. |
| `send_email_alert` | 1 to email the next-state user on transitions |
| `states` (child table: `Workflow Document State`) | The states |
| `transitions` (child table: `Workflow Transition`) | The transitions |

### Workflow Document State

| Field | Purpose |
|-------|---------|
| `state` | Link to `Workflow State` (must exist) |
| `doc_status` | "0" = Draft, "1" = Submitted, "2" = Cancelled. This is critical — it dictates which docstatus the doc has while in this state. |
| `allow_edit` | Role allowed to edit the doc in this state |
| `update_field` | Optional — a field to auto-set when entering this state |
| `update_value` | The value to set |
| `is_optional_state` | If 1, the state can be skipped |
| `next_action_email_template` | Optional Email Template name for the auto-email |
| `message` | Message shown on the form in this state |

### Workflow Transition

| Field | Purpose |
|-------|---------|
| `state` | The from-state |
| `action` | Link to `Workflow Action Master` (button label) |
| `next_state` | The to-state |
| `allowed` | Role allowed to take this transition |
| `condition` | Python expression evaluated against `doc`. Optional. |
| `allow_self_approval` | 1 = creator can approve their own. Default 1, but compliance usually wants 0. |
| `send_email_to_creator` | 1 = email the doc's owner when this transition fires |

## Building a workflow

Recommended path:
1. Create `Workflow State` records for each state name (e.g., "Pending Manager", "Pending CFO"). Set color (visual badge).
2. Create `Workflow Action Master` records for each action label (e.g., "Send to CFO", "Approve", "Reject").
3. Create the `Workflow` record, linking states and transitions.
4. Test on the dev site — submit a doc, take a transition, confirm role permissions work.
5. Export as fixtures (see [fixtures.md](fixtures.md)).

You can skip step 1 and 2 if all state/action names already exist (e.g., reusing "Approved", "Rejected"). But the moment you introduce a new name, you must fixture it.

## Conditional routing

`condition` on a transition is a Python expression with `doc` in scope:

```
# Route low-value POs straight to manager, high-value to CFO
{
    "state": "Draft",
    "action": "Submit for Approval",
    "next_state": "Pending Manager",
    "allowed": "Purchase User",
    "condition": "doc.grand_total < 5000"
},
{
    "state": "Draft",
    "action": "Submit for Approval",
    "next_state": "Pending CFO",
    "allowed": "Purchase User",
    "condition": "doc.grand_total >= 5000"
}
```

Multiple transitions can share the same `state` + `action` — Frappe picks the first whose `condition` is truthy. If no `condition` matches, the action button doesn't appear.

`condition` is sandboxed Python. You have access to `doc`, `frappe.utils` functions, and a few helpers. You do **not** have arbitrary I/O, `import`, or DB writes. For complex routing, put the logic in `doc.before_validate` and set a field that the simple condition reads.

## Programmatic transitions

To advance a doc by code (not via the form):

```python
from frappe.model.workflow import apply_workflow

doc = frappe.get_doc("Purchase Order", "PO-2026-00001")
apply_workflow(doc, "Approve")  # action label, not next_state
```

This validates the role permission, condition, and self-approval rules just like the UI button. Use it from `bench --site X execute` for one-off fixes, or from a controller to chain transitions.

To bypass the workflow checks (rare — usually a code smell), set the state field directly: `doc.db_set("workflow_state", "Approved")`. This skips condition/role checks and side effects (notifications, email alerts). Use only for data migration.

## Multi-step approval chains

A chain like Draft → Pending Manager → Pending CFO → Approved is just states + transitions:

```
Draft        --Submit-->   Pending Manager  (allowed: Purchase User)
Pending Mgr  --Approve-->  Pending CFO      (allowed: Purchase Manager, condition: doc.grand_total >= 5000)
Pending Mgr  --Approve-->  Approved         (allowed: Purchase Manager, condition: doc.grand_total < 5000)
Pending Mgr  --Reject-->   Rejected         (allowed: Purchase Manager)
Pending CFO  --Approve-->  Approved         (allowed: CFO)
Pending CFO  --Reject-->   Rejected         (allowed: CFO)
```

The `doc_status` for `Pending Manager` / `Pending CFO` is 0 (Draft). `Approved` is 1 (Submitted). `Rejected` is 2 (Cancelled).

Set `allow_self_approval: 0` on each Approve transition so the requester can't approve their own doc.

## JSON fixture, end to end

`apps/<app>/<app>/fixtures/workflow.json`:

```json
[
 {
  "doctype": "Workflow",
  "workflow_name": "Purchase Order Approval",
  "document_type": "Purchase Order",
  "is_active": 1,
  "send_email_alert": 1,
  "workflow_state_field": "workflow_state",
  "states": [
   {"state": "Draft", "doc_status": "0", "allow_edit": "Purchase User"},
   {"state": "Pending Manager", "doc_status": "0", "allow_edit": "Purchase Manager"},
   {"state": "Pending CFO", "doc_status": "0", "allow_edit": "CFO"},
   {"state": "Approved", "doc_status": "1", "allow_edit": "Purchase Manager"},
   {"state": "Rejected", "doc_status": "2", "allow_edit": "Purchase User"}
  ],
  "transitions": [
   {"state": "Draft", "action": "Submit for Approval", "next_state": "Pending Manager",
    "allowed": "Purchase User", "allow_self_approval": 1},
   {"state": "Pending Manager", "action": "Approve", "next_state": "Approved",
    "allowed": "Purchase Manager", "condition": "doc.grand_total < 5000",
    "allow_self_approval": 0, "send_email_to_creator": 1},
   {"state": "Pending Manager", "action": "Approve", "next_state": "Pending CFO",
    "allowed": "Purchase Manager", "condition": "doc.grand_total >= 5000",
    "allow_self_approval": 0},
   {"state": "Pending Manager", "action": "Reject", "next_state": "Rejected",
    "allowed": "Purchase Manager", "allow_self_approval": 0, "send_email_to_creator": 1},
   {"state": "Pending CFO", "action": "Approve", "next_state": "Approved",
    "allowed": "CFO", "allow_self_approval": 0, "send_email_to_creator": 1},
   {"state": "Pending CFO", "action": "Reject", "next_state": "Rejected",
    "allowed": "CFO", "allow_self_approval": 0, "send_email_to_creator": 1}
  ]
 }
]
```

`apps/<app>/<app>/fixtures/workflow_state.json`:

```json
[
 {"doctype": "Workflow State", "name": "Pending Manager", "style": "Warning"},
 {"doctype": "Workflow State", "name": "Pending CFO", "style": "Warning"},
 {"doctype": "Workflow State", "name": "Approved", "style": "Success"},
 {"doctype": "Workflow State", "name": "Rejected", "style": "Danger"}
]
```
(Draft already exists by default — don't duplicate it.)

`apps/<app>/<app>/fixtures/workflow_action_master.json`:

```json
[
 {"doctype": "Workflow Action Master", "name": "Submit for Approval"},
 {"doctype": "Workflow Action Master", "name": "Approve"},
 {"doctype": "Workflow Action Master", "name": "Reject"}
]
```

Register in `hooks.py`:

```python
fixtures = [
    {"doctype": "Workflow", "filters": [["name", "in", ["Purchase Order Approval"]]]},
    {"doctype": "Workflow State", "filters": [["name", "in",
        ["Pending Manager", "Pending CFO", "Approved", "Rejected"]]]},
    {"doctype": "Workflow Action Master", "filters": [["name", "in",
        ["Submit for Approval", "Approve", "Reject"]]]},
]
```

Apply: `bench --site <site> migrate && bench --site <site> clear-cache`.

## Common pitfalls

- **Workflow doesn't appear** on the target DocType form → `is_active` is 0, or the user lacks any role mentioned in the transitions. Workflow auto-hides if the user can't take any action.
- **"Workflow state name does not exist"** on migrate → you reference a state in transitions that isn't in the fixtures or DB. Add it to `workflow_state.json`.
- **Approve button missing for a user who should have it** → check role, then check `allow_self_approval` (they may be the doc owner), then check `condition`. The condition expression doesn't show errors in the UI — test it via `bench console`.
- **`workflow_state` field appears twice on the form** → you created a Custom Field with the same name. Delete it; the workflow auto-creates one.
- **Permissions break after activating workflow** → workflow states with `allow_edit` give that role edit perms in that state, but only edit — submit/cancel still need DocType-level perms. Both layers matter.
- **`doc_status` mismatch** → a state with `doc_status: 0` (Draft) but appearing after a Submit transition will leave the doc in Draft despite being "submitted". Make sure your terminal approved state has `doc_status: 1`.
- **Override existing workflow** → only one active workflow per DocType. Set `is_active: 0` on the old one before activating a new one, or migration will refuse.

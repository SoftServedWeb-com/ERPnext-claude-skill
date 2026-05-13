# Multi-doctype orchestration

Patterns for "when X happens to one doc, do something to a related doc": auto-create drafts, propagate values, link records, trigger external systems.

## Table of contents
- [The pattern](#the-pattern)
- [Transaction model — the part that surprises people](#transaction-model--the-part-that-surprises-people)
- [Sync vs background](#sync-vs-background)
- [Idempotency](#idempotency)
- [Permission bypass: when and why](#permission-bypass-when-and-why)
- [Error handling and rollback](#error-handling-and-rollback)
- [Linking records (`against_*` fields)](#linking-records-against_-fields)
- [Worked example: SO → DN with notification](#worked-example-so--dn-with-notification)
- [Common pitfalls](#common-pitfalls)

## The pattern

```python
# apps/<app>/<app>/overrides/sales_order.py
import frappe

def on_submit(doc, method):
    create_delivery_note_draft(doc)

def create_delivery_note_draft(so):
    dn = frappe.new_doc("Delivery Note")
    dn.customer = so.customer
    dn.posting_date = frappe.utils.nowdate()
    for item in so.items:
        dn.append("items", {
            "item_code": item.item_code,
            "qty": item.qty,
            "warehouse": item.warehouse,
            "against_sales_order": so.name,
            "so_detail": item.name,
        })
    dn.insert(ignore_permissions=True)
    return dn
```

Wire it up in `hooks.py`:
```python
doc_events = {
    "Sales Order": {
        "on_submit": "<app>.<app>.overrides.sales_order.on_submit",
    },
}
```

That's the bones. Every detail below is about making this safe, idempotent, and predictable.

## Transaction model — the part that surprises people

Frappe wraps the entire HTTP request in a single DB transaction. That includes:
- The original action (the Sales Order submit)
- Every hook (`validate`, `on_submit`)
- Every spawned `.insert()` / `.save()` call

If **anything** raises an exception that escapes the request, the entire transaction rolls back. Including the original submit.

This is usually what you want — if the Delivery Note can't be created, you probably don't want the Sales Order to be left in a half-submitted state. But it can surprise people:
- User submits Sales Order
- `on_submit` tries to spawn Delivery Note, but item has no warehouse → throws
- User sees an error and the Sales Order didn't submit either

If you want the SO to submit even when the DN spawn fails, wrap in try/except:
```python
def on_submit(doc, method):
    try:
        create_delivery_note_draft(doc)
    except Exception:
        frappe.log_error(message=frappe.get_traceback(),
                         title=f"DN spawn failed for {doc.name}")
        frappe.msgprint("Note: could not auto-create Delivery Note. Please create manually.")
```

This decouples the spawn from the original submit. The original transaction commits; the spawn failure is logged but doesn't roll back.

## Sync vs background

Synchronous (the default): the spawn happens during the original request. Pros: simple, atomic, immediate. Cons: slows the response, fails the original if the spawn fails (unless caught).

Background: push the spawn to a worker queue.
```python
def on_submit(doc, method):
    frappe.enqueue(
        "myapp.myapp.overrides.sales_order.create_delivery_note_draft_bg",
        sales_order=doc.name,
        queue="long",       # default | short | long
        timeout=600,
        enqueue_after_commit=True,  # important — don't run until original txn commits
    )

def create_delivery_note_draft_bg(sales_order):
    so = frappe.get_doc("Sales Order", sales_order)
    create_delivery_note_draft(so)
```

Two important details:
- `enqueue_after_commit=True` — without this, the background job can start *before* the original transaction commits, which can lead to "Sales Order not found" errors in the worker. Always use this when enqueuing from a doc_event.
- The worker process must be running. Verify with `bench doctor`.

Use background when:
- The spawn is slow (talks to external APIs, processes many items)
- The user shouldn't have to wait
- Failures should be retried later rather than blocking the user

Stick with synchronous when:
- The spawn must complete or the original action is wrong
- The user needs to see the spawned record's name in the response
- The spawn is fast and reliable

## Idempotency

Doc_events can fire more than you expect:
- User submits, network glitch, user resubmits → on_submit fires twice (on the same doc, even)
- Scheduler retries a failed job → background spawn re-runs
- Migration re-runs an on_update_after_submit

Make spawns idempotent by checking for existence first:

```python
def create_delivery_note_draft(so):
    existing = frappe.db.exists("Delivery Note", {
        "against_sales_order": so.name,
        "docstatus": ["in", [0, 1]],   # don't count cancelled
    })
    if existing:
        return frappe.get_doc("Delivery Note", existing)

    dn = frappe.new_doc("Delivery Note")
    # ...
    dn.insert(ignore_permissions=True)
    return dn
```

For child-table-level linking (one SO → many DNs but each item only once), check `against_sales_order_item` similarly.

## Permission bypass: when and why

`ignore_permissions=True` skips DocType permission checks on the spawned doc. You usually need it because the user submitting the SO may not have create perm on Delivery Note.

But it bypasses safety. Two principles:
1. **Bypass on the system-driven spawn, not on the user-driven action.** The user submitted an SO they were allowed to submit — the DN being created on their behalf is a system action.
2. **Validate inside the spawn function** before bypassing — duplicate the relevant checks (e.g., does the item exist? is the warehouse allowed?). Don't trust that "permissions would have caught this".

If the spawned doc needs to attribute the creation to the user, set `owner` explicitly:
```python
dn.flags.ignore_permissions = True
dn.owner = frappe.session.user
dn.insert()
```

## Error handling and rollback

When spawning, decide the rollback policy explicitly:

**Atomic (default — raise to abort the whole transaction):**
```python
def on_submit(doc, method):
    create_delivery_note_draft(doc)  # raises propagate up
```

**Best-effort (catch and log):**
```python
def on_submit(doc, method):
    try:
        create_delivery_note_draft(doc)
    except Exception:
        frappe.log_error(title=f"DN spawn failed for SO {doc.name}",
                         message=frappe.get_traceback())
        frappe.msgprint("DN auto-creation failed — see Error Log")
```

**Background retry (queue and retry on failure):**
```python
def on_submit(doc, method):
    frappe.enqueue(
        "myapp.myapp.overrides.sales_order.create_delivery_note_draft_bg",
        sales_order=doc.name,
        queue="long",
        timeout=600,
        enqueue_after_commit=True,
    )
```
Background jobs auto-retry on failure (configurable in `frappe.utils.background_jobs`).

Always tell the user which mode they want. Don't silently pick.

## Linking records (`against_*` fields)

ERPnext doctypes have a standard `against_<source_doctype>` pattern on child rows:

| Source | Child row field |
|--------|------------------|
| Sales Order → Delivery Note | `against_sales_order` (parent), `so_detail` (child row name) |
| Sales Order → Sales Invoice | `sales_order` (parent), `so_detail` |
| Purchase Order → Purchase Receipt | `purchase_order`, `purchase_order_item` |
| Purchase Order → Purchase Invoice | `purchase_order`, `po_detail` |

Setting these correctly is what makes ERPnext's reports (Outstanding SO Items, Pending Receipts) work. Skipping them means downstream reports are wrong.

You can also use `frappe.model.mapper.get_mapped_doc(...)` to handle the mapping declaratively:

```python
from frappe.model.mapper import get_mapped_doc

def make_delivery_note_from_so(source_name):
    return get_mapped_doc("Sales Order", source_name, {
        "Sales Order": {
            "doctype": "Delivery Note",
            "field_map": {"customer": "customer"},
            "validation": {"docstatus": ["=", 1]},
        },
        "Sales Order Item": {
            "doctype": "Delivery Note Item",
            "field_map": {"name": "so_detail", "parent": "against_sales_order"},
            "postprocess": lambda src, tgt, src_parent: setattr(tgt, "qty", src.qty),
        },
    })
```

The mapper has built-in support for tracking already-mapped quantities (so you don't over-deliver). For ERPnext-internal flows, stock ERPnext uses this everywhere — read `erpnext/selling/doctype/sales_order/sales_order.py` for `make_delivery_note` as the reference implementation.

## Worked example: SO → DN with notification

`apps/<app>/<app>/hooks.py`:
```python
doc_events = {
    "Sales Order": {
        "on_submit": "<app>.<app>.overrides.sales_order.spawn_delivery_note",
    },
}
```

`apps/<app>/<app>/overrides/sales_order.py`:
```python
import frappe
from frappe.utils import nowdate

def spawn_delivery_note(doc, method):
    if frappe.db.exists("Delivery Note", {"against_sales_order": doc.name, "docstatus": ["<", 2]}):
        return  # idempotent

    dn = frappe.new_doc("Delivery Note")
    dn.customer = doc.customer
    dn.posting_date = nowdate()
    dn.against_sales_order = doc.name  # also set on parent for easier filtering
    for item in doc.items:
        dn.append("items", {
            "item_code": item.item_code,
            "qty": item.qty,
            "warehouse": item.warehouse,
            "against_sales_order": doc.name,
            "so_detail": item.name,
            "uom": item.uom,
            "rate": item.rate,
        })
    dn.flags.ignore_permissions = True
    dn.insert()

    frappe.sendmail(
        recipients=get_warehouse_team_emails(dn),
        subject=f"New Delivery Note draft: {dn.name} for {dn.customer}",
        message=frappe.render_template(
            "myapp/templates/emails/dn_draft.html",
            {"dn": dn, "so": doc},
        ),
        reference_doctype=dn.doctype,
        reference_name=dn.name,
    )

    frappe.msgprint(
        f"Created Delivery Note draft <a href='/app/delivery-note/{dn.name}'>{dn.name}</a>"
    )

def get_warehouse_team_emails(dn):
    # Replace with your real lookup
    return frappe.db.get_all(
        "User", filters={"role_profile_name": "Warehouse Team"}, pluck="email"
    )
```

Bench commands:
```
bench --site <site> migrate
bench restart
```

Then submit an SO. Confirm:
- DN exists with `docstatus = 0` (Draft)
- DN's items have correct `against_sales_order` and `so_detail`
- Warehouse team got the email
- Re-submitting the same SO doesn't create a duplicate DN (idempotency)

## Common pitfalls

- **"Sales Order not found" in a background worker** — you forgot `enqueue_after_commit=True`. The job ran before the original transaction committed.
- **Doc spawned but invisible to the user** — they don't have read permission on the spawned doctype. Either grant read, or include the link in a notification.
- **Two DNs spawn for the same SO** — idempotency check missing or wrong filter (e.g., `docstatus == 0` but the first DN was already submitted). Use `docstatus < 2` to count anything not cancelled.
- **`on_submit` blocks for 30s, user thinks the page hung** — switch to background spawn. Or at least add `frappe.publish_realtime(...)` to flash progress.
- **Spawn references a field that doesn't exist** — DocType evolved but your spawn code didn't. Tests on a fresh site catch this; manual testing on dev doesn't.
- **Mapper double-counts quantities** — you used `get_mapped_doc` but didn't set the `condition` / `validation` to skip already-mapped items. Read `erpnext.controllers.selling_controller` for how stock ERPnext avoids this.

# doc_events and scheduler_events

The two main ways to run Python code in response to events in a Frappe app.

## Table of contents
- [doc_events: the lifecycle hooks](#doc_events-the-lifecycle-hooks)
- [Event reference](#event-reference)
- [Wiring in hooks.py](#wiring-in-hookspy)
- [Why on_submit can't modify the doc](#why-on_submit-cant-modify-the-doc)
- [Stacking multiple hooks](#stacking-multiple-hooks)
- [scheduler_events](#scheduler_events)
- [Server Script equivalents](#server-script-equivalents)
- [Common pitfalls](#common-pitfalls)

## doc_events: the lifecycle hooks

When a doc moves through its lifecycle (create / save / submit / cancel / delete), Frappe fires a sequence of events. You can hook into any of them from `hooks.py`. The function signature is always:

```python
def handler(doc, method):
    # doc is the Frappe document
    # method is the event name as a string (e.g., "on_submit")
    ...
```

## Event reference

In order, for a typical save → submit → cancel cycle:

| Event | Fires when | Can modify doc? | Notes |
|-------|-----------|-----------------|-------|
| `before_insert` | Before a new doc is inserted | Yes | Doc has no `name` yet (unless autoname is field-based) |
| `after_insert` | After insert | No (use `db_set`) | First event with a guaranteed `name` |
| `before_validate` | Before validate, on every save | Yes | Good place to mutate fields before validation |
| `validate` | Standard validation point | Yes | `frappe.throw` here to block save |
| `before_save` | Before write to DB | Yes | After validate, before write |
| `on_update` | After save (every save, draft or submitted) | No (use `db_set`) | Fires on every save including the first |
| `before_submit` | Before submit | Yes | Last chance to mutate before docstatus = 1 |
| `on_submit` | After submit | No (use `db_set`) | Doc is now docstatus 1; subsequent assignment to `doc.x` is lost |
| `on_update_after_submit` | Save of an already-submitted doc | No (use `db_set`) | Triggered by `db_set` or "Update" UI |
| `before_cancel` | Before cancel | No | docstatus is still 1 |
| `on_cancel` | After cancel | No (use `db_set`) | docstatus is now 2 |
| `on_trash` | Before delete | N/A | Last chance to refuse via `frappe.throw` |
| `after_delete` | After delete | N/A | Doc no longer exists in DB |
| `on_change` | After any change (alias for on_update) | No | Less common; prefer `on_update` |

There is no `before_delete` — use `on_trash`.

## Wiring in hooks.py

```python
# apps/<app>/<app>/hooks.py
doc_events = {
    "Sales Invoice": {
        "validate": "<app>.<app>.overrides.sales_invoice.validate",
        "on_submit": "<app>.<app>.overrides.sales_invoice.on_submit",
        "on_cancel": "<app>.<app>.overrides.sales_invoice.on_cancel",
    },
    "Purchase Order": {
        "validate": "<app>.<app>.overrides.purchase_order.validate",
    },
    "*": {
        # applies to every doctype
        "on_update": "<app>.<app>.utils.audit.record_change",
    },
}
```

Then create the override module (e.g., `apps/<app>/<app>/overrides/sales_invoice.py`):

```python
import frappe

def validate(doc, method):
    if doc.grand_total < 0:
        frappe.throw("Grand total cannot be negative")
    if doc.due_date and doc.due_date < doc.posting_date:
        frappe.throw("Due date cannot be before posting date")

def on_submit(doc, method):
    # Locking the doc by setting a custom flag
    doc.db_set("custom_locked_at", frappe.utils.now())
    # Side effects
    frappe.publish_realtime(
        event="invoice_submitted",
        message={"name": doc.name, "total": doc.grand_total},
        user=doc.owner,
    )

def on_cancel(doc, method):
    doc.db_set("custom_locked_at", None)
```

After editing `hooks.py`:
```
bench --site <site> migrate
bench restart
```

`bench restart` is non-negotiable. The dev server caches the Python module and won't pick up new event registrations until restarted.

## Why on_submit can't modify the doc

After submit, Frappe locks the in-memory doc to prevent accidental mutations from leaking into the saved record (since the save happened *before* `on_submit` fires). Mutating `doc.field = value` in `on_submit` does nothing — there's no subsequent save.

Use `doc.db_set("field", value)` to write directly to the DB. It bypasses validate and on_update_after_submit (use `update_modified=True` if you want the modified timestamp to advance).

This rule also applies to `on_update`, `on_cancel`, `on_update_after_submit`, and `after_insert`. The mnemonic: **any event named `on_*` is after the save — use `db_set`**.

For events named `before_*` or `validate`, you can mutate `doc.field` directly and it persists.

## Stacking multiple hooks

You can register multiple handlers for the same event by making the hook value a list:

```python
doc_events = {
    "Sales Invoice": {
        "validate": [
            "<app>.<app>.overrides.sales_invoice.validate_totals",
            "<app>.<app>.overrides.sales_invoice.validate_compliance",
        ],
    },
}
```

They fire in list order. If any one throws, subsequent handlers don't run and the save is aborted.

Order across apps: if two installed apps both hook `Sales Invoice.validate`, the order is determined by the order in which apps are installed (which is the order in `apps.txt`). Don't rely on cross-app ordering for correctness.

## scheduler_events

For code that runs on a schedule.

```python
scheduler_events = {
    "all": [  # every 4 minutes (yes, "all" means "every tick")
        "<app>.<app>.tasks.heartbeat",
    ],
    "hourly": [
        "<app>.<app>.tasks.check_pending_approvals",
    ],
    "daily": [
        "<app>.<app>.tasks.send_overdue_reminders",
    ],
    "weekly": [
        "<app>.<app>.tasks.weekly_summary",
    ],
    "monthly": [
        "<app>.<app>.tasks.monthly_report",
    ],
    "cron": {
        "0 9 * * 1-5": [  # 9am Mon-Fri
            "<app>.<app>.tasks.morning_digest",
        ],
        "*/15 * * * *": [  # every 15 minutes
            "<app>.<app>.tasks.poll_external_api",
        ],
    },
}
```

The scheduler runs in the worker process, not the web process. The functions take no arguments:

```python
import frappe

def send_overdue_reminders():
    overdue = frappe.get_all(
        "Sales Invoice",
        filters={"due_date": ["<", frappe.utils.nowdate()], "outstanding_amount": [">", 0]},
        fields=["name", "customer", "outstanding_amount"],
    )
    for inv in overdue:
        # send mail
        pass
```

**Background job worker must be running** (`bench start` in dev, supervisor in prod). To verify: `bench --site <site> show-pending-jobs` and check `bench worker` is up.

To test a scheduler function manually without waiting:
```
bench --site <site> execute <app>.<app>.tasks.send_overdue_reminders
```

## Server Script equivalents

If you can't or don't want to deploy code (e.g., site-local logic, or you don't have shell access), the Server Script DocType provides the same hooks:

- DocType Event = doc_events
- Scheduler Event = scheduler_events
- API = custom REST endpoint at `/api/method/<script_name>`
- Permission Query = `permission_query_conditions`

Enable: `bench --site <site> set-config server_script_enabled 1`. Without this, scripts silently don't run.

Inside a Server Script you have `doc`, `frappe`, `frappe.utils`, and a sandboxed subset of stdlib. No arbitrary `import`. No file I/O. No subprocess.

Choose Server Script when:
- The logic is site-specific and shouldn't live in the app
- You need non-developers to be able to tweak it
- You're on a managed hosting tier without shell access

Choose hooks.py when:
- The logic is part of the app's contract
- It needs anything the sandbox forbids
- You want it under version control

## Common pitfalls

- **Hook doesn't fire** → 90% of the time: forgot `bench restart`. The other 10%: typo in the dotted path. Test the import with `bench --site <site> console`:
  ```python
  from frappe.utils import get_attr
  get_attr("myapp.myapp.overrides.sales_invoice.validate")
  ```
- **`doc_events` for a doctype is silently merged** with the framework's own hooks. You don't override built-in ERPnext behavior — you stack on top. To replace, override the DocType class via `override_doctype_class` in hooks.py.
- **`on_submit` modifications "disappear"** → see the rule above. Use `db_set`.
- **Infinite loops from on_update** → if your `on_update` handler calls `doc.save()`, you'll recurse. Use `db_set` for tweaks, or guard with `frappe.flags`.
- **Scheduler function fires twice** → you registered the same function under both `all` and `hourly`, or the worker is running two processes. Check `bench --site <site> show-pending-jobs`.
- **`*` doctype hook fires on system doctypes** too (Communication, Comment, File, etc.), which can be surprising. Filter inside the handler: `if doc.doctype.startswith("Sales"): ...`.
- **Module not found after restart** → you added a new `.py` file but no `__init__.py` in the directory. Frappe needs the package marker.

---
name: erpnext-workflow-automation
description: Build ERPnext v15 / Frappe workflow automation — approval state machines, doc_events hooks (validate/on_submit/on_cancel), client scripts, Notification doctype alerts, scheduled jobs, and multi-doctype business processes. Use this whenever the user is working inside a frappe-bench directory or a custom Frappe app, OR explicitly mentions ERPnext, Frappe, DocType, hooks.py, bench commands, fixtures, or workflow states. Also trigger on business-process language like "approval flow", "when X is submitted then Y", "route based on amount/role", "notify when status changes", "auto-create a draft", or "lock the doc after submit" — even when the user doesn't name ERPnext explicitly. Outputs step-by-step guidance, JSON fixtures, code edits at the right paths, and the bench commands to apply changes.
---

# ERPnext v15 Workflow Automation

You are helping someone wire automation into an ERPnext v15 / Frappe app. The Frappe framework gives you several different mechanisms for automation, and **picking the right one is most of the job**. Picking the wrong one (e.g., a client script when you needed a server-side validation, or hooks.py when the user wanted a state machine UI) leads to brittle code and confused stakeholders.

This skill walks you through:
1. Quick orientation on the Frappe building blocks
2. A decision tree for **which mechanism to reach for**
3. Minimal patterns for each mechanism, with pointers into `references/` for depth
4. Cross-cutting concerns: fixtures, permissions, bench commands, and the v15-specific gotchas that bite people

If you are already an expert in Frappe v15, you can skim. Otherwise, read top to bottom on first use.

---

## Orientation: the pieces

- **App** — a Python package inside `frappe-bench/apps/<app_name>/`. Has a `hooks.py` at `apps/<app>/<app>/hooks.py`. Apps are installed into sites.
- **Site** — a tenant database, at `frappe-bench/sites/<site_name>/`. Apps run *against* a site.
- **DocType** — the schema + class for a record type (Customer, Sales Invoice, your custom thing). Lives at `apps/<app>/<app>/<module>/doctype/<doctype>/`. Contains `<doctype>.json` (schema), `<doctype>.py` (server controller), `<doctype>.js` (client controller).
- **Workflow** (the DocType named "Workflow") — a state-machine definition attached to *another* DocType. States + transitions + the roles allowed to take each transition. This is what people usually mean by "workflow" in ERPnext.
- **hooks.py** — the Python file where you register doc_events, scheduler_events, permission hooks, fixtures lists, etc. App-wide automation lives here.
- **Server Script** (the DocType) — Python automation **stored in the database**, written via the UI, executed in a restricted sandbox. Useful for site-local logic without code deploys. Requires `server_script_enabled: 1` in `site_config.json`.
- **Client Script** (the DocType) — JS form behavior stored in the DB. Same idea as Server Script, but for the form UI. Lives alongside the `<doctype>.js` file shipped by the app.
- **Notification** (the DocType) — declarative email/system/Slack alerts triggered on document events or on a schedule. Jinja templates, role recipients.
- **Fixtures** — JSON snapshots of DocType records (Workflows, Notifications, Custom Fields, Property Setters, Roles) that ship inside an app and load on `bench migrate`. This is how you version-control configuration.

The key dichotomy to internalize: **declarative vs programmatic**.
- Declarative tools (Workflow doctype, Notification doctype, Custom Fields, Property Setter) are configured via the UI, exported as JSON fixtures, and read by Frappe at runtime. Non-developers can change them. Great for state machines, alerts, and form tweaks.
- Programmatic tools (hooks.py, controller `.py`, `<doctype>.js`, Server Scripts) are code. You reach for them when logic is conditional, multi-step, or needs to touch external systems.

Almost every "ERPnext automation" task is a blend of both — declare the *shape* with fixtures, implement the *behavior* with code.

---

## Decision tree: which mechanism?

Walk this top-down. When the answer is yes, that's your primary tool — though you'll often combine it with others.

1. **Does the user want stakeholders to see a status badge, an approval button, and a state history on a record?**
   → **Frappe Workflow** (the DocType). See `references/workflows.md`.

2. **Does logic need to fire on a specific document lifecycle event — validate, before_save, on_submit, on_cancel — regardless of user UI?**
   → **doc_events in hooks.py** (or Server Script if site-local). See `references/doc-events.md`.

3. **Does the user want the form to behave differently — show/hide fields, fetch values, add a custom button, filter a Link field?**
   → **Client Script** (the DocType) or the app-shipped `<doctype>.js`. See `references/client-scripts.md`.

4. **Should an email / system notification / Slack message go out when something happens?**
   → **Notification doctype** for declarative, hooks/Server Script for conditional dispatch. See `references/notifications.md`.

5. **Does something need to run on a schedule (nightly, hourly, cron)?**
   → **scheduler_events in hooks.py**. See `references/doc-events.md#scheduler`.

6. **Is the user describing a brand-new business object with its own approval flow?**
   → **New DocType + Workflow + doc_events**, often combined. See `references/custom-doctype-with-workflow.md`.

7. **Does the user want one document to spawn another (e.g., Sales Order → Delivery Note draft) automatically?**
   → **on_submit doc_event** that calls `frappe.get_doc({...}).insert()`. See `references/multi-doctype-orchestration.md`.

8. **Does the routing depend on amount, role, department, or any condition?**
   → **Workflow with `condition` on transitions** (Python expressions evaluated against the doc) — possibly combined with multiple workflow states. See `references/workflows.md#conditional-routing`.

If you can't tell which bucket the request falls into, ask a clarifying question rather than guessing. Picking the wrong mechanism is expensive to undo.

---

## Pattern 1: Frappe Workflow (state machines)

The most common ERPnext automation pattern. You define states, the transitions between them, and which roles can take each transition.

### Minimal shape

A Workflow has:
- `document_type` — the DocType this workflow applies to (e.g., "Purchase Order")
- `is_active` — must be 1
- `workflow_state_field` — defaults to `workflow_state`. Frappe will auto-add a Custom Field of this name on the target DocType.
- `states` — list of `{state, doc_status, allow_edit, update_field?, update_value?}`. `doc_status` is 0 (Draft), 1 (Submitted), or 2 (Cancelled).
- `transitions` — list of `{state, action, next_state, allowed (role), condition?, allow_self_approval?}`

### When to use a fixture vs the UI

For dev: build it in the UI on your dev site to validate the shape, then export with `bench --site <site> export-fixtures` or hand-craft the JSON and place at `apps/<app>/<app>/fixtures/workflow.json`. Make sure `Workflow` is listed in `hooks.py`:

```python
fixtures = [
    {"doctype": "Workflow", "filters": [["name", "in", ["My PO Approval"]]]},
    {"doctype": "Workflow State", "filters": [["name", "in", ["Pending Manager", "Pending CFO"]]]},
    {"doctype": "Workflow Action Master", "filters": [["name", "in", ["Send to CFO"]]]},
]
```

If you add new state names or action names, you **must** also fixture the corresponding `Workflow State` and `Workflow Action Master` records — they are referenced by name from the workflow's states/transitions. Forgetting these is the #1 reason workflows break after `bench migrate` on a fresh site.

### Conditional routing

Set `condition` on a transition to a Python expression that evaluates against `doc`:
```
doc.grand_total < 5000
```
Use `condition` *together with* an `allowed` role — both must pass.

See `references/workflows.md` for the full schema, more examples, and how to script transitions programmatically.

---

## Pattern 2: doc_events in hooks.py

When you need code to run on a document's lifecycle event for every record of that DocType (or for many DocTypes).

```python
# apps/<app>/<app>/hooks.py
doc_events = {
    "Sales Invoice": {
        "validate": "<app>.<app>.overrides.sales_invoice.validate",
        "on_submit": "<app>.<app>.overrides.sales_invoice.on_submit",
    },
    "*": {  # applies to every doctype
        "on_update": "<app>.<app>.utils.audit.record_change",
    },
}
```

Then in `apps/<app>/<app>/overrides/sales_invoice.py`:
```python
import frappe

def validate(doc, method):
    if doc.grand_total < 0:
        frappe.throw("Grand total cannot be negative")

def on_submit(doc, method):
    # NOTE: you cannot modify `doc` and have it persist here — use db_set
    doc.db_set("custom_locked_at", frappe.utils.now())
```

After editing `hooks.py`: `bench --site <site> migrate` then `bench restart`. Without a restart, the new hooks are not picked up.

See `references/doc-events.md` for the full event list, ordering, the `on_submit`-can't-modify-doc rule, and Server Script equivalents.

---

## Pattern 3: Client Scripts

For form behavior in the desk UI. Two flavors:
- **Shipped as code** in `apps/<app>/<app>/<module>/doctype/<doctype>/<doctype>.js`
- **Stored as data** in the `Client Script` DocType (visible in the UI under "Customize Form" → Client Script)

The first ships with the app and version-controls cleanly. The second is editable in the UI but should be fixtured if you want it to deploy. Prefer the first for app developers, the second for site-local tweaks.

```javascript
// apps/<app>/<app>/<module>/doctype/sales_order/sales_order.js
frappe.ui.form.on("Sales Order", {
    customer: function(frm) {
        if (!frm.doc.customer) return;
        frappe.db.get_value("Customer", frm.doc.customer, "customer_group")
            .then(r => {
                if (r.message.customer_group === "Corporate") {
                    frm.add_custom_button("Run Compliance Check", () => {
                        // open dialog, etc.
                    });
                }
            });
    },
});
```

See `references/client-scripts.md` for the full event list (`refresh`, `onload`, fieldname events, child-table events), dialog patterns, `frm.set_query` for Link filters, and the difference between `frm.set_value` and `frm.doc.x = y`.

---

## Pattern 4: Notifications

The Notification DocType lets you fire an email / system notification / Slack message when:
- A document changes to a specific value (e.g., status == "Approved")
- A document is created / submitted / cancelled
- On a daily/weekly schedule (cron-ish)
- After N days from a date field

```json
{
  "doctype": "Notification",
  "name": "PO Approved - Notify Requester",
  "subject": "Purchase Order {{ doc.name }} approved",
  "document_type": "Purchase Order",
  "event": "Value Change",
  "value_changed": "workflow_state",
  "condition": "doc.workflow_state == 'Approved'",
  "channel": "Email",
  "recipients": [{"receiver_by_document_field": "owner"}],
  "message": "<p>Your PO <b>{{ doc.name }}</b> has been approved.</p>"
}
```

Place this in `apps/<app>/<app>/fixtures/notification.json` and add `Notification` to the fixtures list in `hooks.py`.

For conditional dispatch that's too complex for the Notification doctype (e.g., "email different people based on amount and department"), do it programmatically: in a doc_event, call `frappe.sendmail(...)` or `frappe.publish_realtime(...)`.

See `references/notifications.md` for jinja context, recipient patterns, and how to test notifications without spamming real inboxes.

---

## Pattern 5: New DocType with built-in workflow

When the user describes a brand-new business object ("Project Approval Request", "Vendor Onboarding"), the work is typically:

1. **Create the DocType** with its fields. Place at `apps/<app>/<app>/<module>/doctype/<snake_case>/<snake_case>.json` (plus `.py`, `.js`, `__init__.py`). Or build via UI on dev site, then move into the app's folder.
2. **Add the Workflow** that defines its states + transitions (Pattern 1).
3. **Add validation / on_submit logic** in the controller `.py` or via doc_events (Pattern 2).
4. **Add notifications** for the state transitions (Pattern 4).
5. **Fixture everything** so the install on a fresh site reproduces it.

The DocType `.json` file is hand-editable but tedious. Best workflow: build in the UI, then `bench --site <site> export-fixtures` and move into the app. See `references/custom-doctype-with-workflow.md` for the full walkthrough.

---

## Pattern 6: Multi-doctype orchestration

"When Sales Order is submitted, auto-create a Delivery Note draft and notify warehouse."

```python
def on_submit(doc, method):
    dn = frappe.new_doc("Delivery Note")
    dn.customer = doc.customer
    dn.posting_date = frappe.utils.nowdate()
    for item in doc.items:
        dn.append("items", {
            "item_code": item.item_code,
            "qty": item.qty,
            "against_sales_order": doc.name,
            "so_detail": item.name,
        })
    dn.insert(ignore_permissions=True)  # consider whether you want this
    frappe.msgprint(f"Created Delivery Note <a href='/app/delivery-note/{dn.name}'>{dn.name}</a>")
```

Gotchas to surface to the user:
- `ignore_permissions=True` skips permission checks on the *spawned* doc. Often necessary because the original submitter may not have create perm on the target doctype. But it bypasses safety — confirm intent.
- Spawning a doc in `on_submit` runs synchronously and **blocks the submit response**. If the spawned doc is heavy or talks to external systems, use `frappe.enqueue(...)` to push it to a background job.
- If the spawn fails, the original submit is rolled back (it's all one transaction). That can surprise people.

See `references/multi-doctype-orchestration.md` for transaction model, `frappe.enqueue` patterns, and how to make spawned docs idempotent.

---

## Cross-cutting: fixtures and deployment

Every customization you make on the dev site needs to ship to staging/prod. The mechanism is **fixtures**.

In `hooks.py`:
```python
fixtures = [
    {"doctype": "Workflow", "filters": [["name", "in", ["My PO Approval"]]]},
    {"doctype": "Workflow State", "filters": [["name", "in", ["Pending Manager", "Pending CFO"]]]},
    {"doctype": "Workflow Action Master", "filters": [["name", "in", ["Send to CFO", "Approve", "Reject"]]]},
    {"doctype": "Notification", "filters": [["name", "in", ["PO Approved - Notify Requester"]]]},
    {"doctype": "Custom Field", "filters": [["name", "like", "%-my_custom_field%"]]},
    {"doctype": "Property Setter", "filters": [["module", "=", "My Module"]]},
    "Custom DocType",  # bare string = export all of this doctype
    "Role",
]
```

Then:
```
bench --site <site> export-fixtures --app <app>
```
writes JSON files to `apps/<app>/<app>/fixtures/`. Commit those. On `bench migrate`, Frappe loads them.

Use **filters** (not bare strings) for anything that isn't 100% app-owned, otherwise you'll capture unrelated customizations from your dev site.

See `references/fixtures.md` for the full pattern, common filter recipes, and how to debug a fixture that won't load.

---

## Cross-cutting: bench commands you will run

```
bench --site <site> migrate              # apply schema + fixtures + patches
bench --site <site> clear-cache          # cache busts after permission/role/workflow edits
bench --site <site> console              # python REPL with frappe imported and site set
bench --site <site> execute <dotted.path>  # run a function once
bench --site <site> export-fixtures      # write fixtures to disk for commit
bench --site <site> reload-doctype "Sales Order"  # reload a single doctype's JSON
bench restart                            # required after hooks.py / .py code changes
bench start                              # foreground dev mode (Procfile-driven)
bench build                              # rebuild JS/CSS bundles after .js changes
bench --site <site> set-config server_script_enabled 1   # enable Server Scripts
```

Always tell the user *which* commands to run after your edits. Edits to `.py` files in `hooks.py` need `bench restart`. Edits to `.js` may need `bench build` (in production mode). Edits to DocType JSON need `bench migrate` or `bench reload-doctype`. Don't make them guess.

See `references/bench-cheatsheet.md` for the full list including site management, app install/uninstall, and common one-liners.

---

## Cross-cutting: permissions & workflow

Workflow transitions interact with role permissions in non-obvious ways:
- The `allowed` field on a transition is a **role**, not a user. The acting user must have that role.
- The user **also** needs permission on the underlying DocType (submit perm to advance a Submitted-state doc, etc.) — workflow doesn't override docperms, it adds another layer.
- `allow_self_approval` defaults to 1. Set to 0 to prevent the doc's creator from approving their own document. This is almost always what compliance teams want — surface it.

See `references/permissions.md` for the interaction with User Permissions, Permission Levels, and how to debug "user can't see the Approve button" cases.

---

## v15-specific gotchas (the important ones)

These are inline because they bite often. The rest are in `references/gotchas.md`.

- **`on_submit` cannot modify the doc and have it persist normally.** Use `doc.db_set("field", value)` to write through. Mutating `doc.field = x` in `on_submit` is silently lost.
- **Workflow auto-creates a Custom Field** named `workflow_state` on the target DocType. Don't pre-create it yourself, and don't try to rename it via Property Setter — change it via the Workflow's `workflow_state_field` if you must.
- **Fixtures are loaded in dependency order Frappe guesses at.** If your Workflow references a Workflow State or Workflow Action Master that doesn't exist yet, the migrate fails. Always fixture states + actions alongside the workflow.
- **`bench restart` is required** after any change to Python code in your app (including `hooks.py`). The dev server doesn't auto-reload Python.
- **Server Scripts are off by default in v15.** `bench --site <site> set-config server_script_enabled 1` to enable. Without it, Server Scripts silently don't run.
- **Background jobs (`frappe.enqueue`)** need the worker process running (`bench start` or production supervisor). If jobs queue but never run, the worker is the suspect.
- **Naming series + workflow** interaction: if your DocType uses a naming series, the name is assigned at insert. Workflows don't change the name. Don't try to encode workflow state in the name.

---

## How to use this skill

When you receive a request:

1. **Diagnose** which mechanism(s) apply, using the decision tree above. State your diagnosis to the user in one sentence ("This is a Frappe Workflow with conditional routing, plus a Notification on the Approved transition.") so they can correct you before you write code.

2. **Read the relevant reference file(s)** in `references/` for the patterns you'll use. Don't try to recall everything from this top-level doc — the references have schemas and gotchas you need.

3. **Plan the files you'll touch** before editing: hooks.py, controller .py, .js, fixtures JSON. Tell the user the plan.

4. **Make the edits** at the right paths. Use the user's actual app name (read `hooks.py` or ask if unclear). Don't write `<app>` as a literal in delivered code.

5. **Always end with the bench commands the user needs to run** and what each does. This is non-negotiable — a v15 user who doesn't run `bench restart` after a hooks.py change will think your code is broken.

6. **Surface non-obvious choices** as you go: permission bypass (`ignore_permissions=True`), self-approval flag, sync vs background, transaction boundaries. Don't bury these in code.

---

## References

| File | When to read |
|------|-------------|
| `references/workflows.md` | Building Frappe Workflows: schema, states, transitions, conditions, programmatic transitions |
| `references/doc-events.md` | hooks.py doc_events list, scheduler_events, ordering, Server Script equivalents |
| `references/client-scripts.md` | frm events, custom buttons, dialogs, set_query, fetch_from |
| `references/notifications.md` | Notification doctype, jinja context, recipients, scheduling, programmatic sendmail |
| `references/fixtures.md` | Exporting/importing fixtures, filter recipes, debugging load failures |
| `references/bench-cheatsheet.md` | Full bench command list with usage |
| `references/permissions.md` | Workflow ↔ role permission interactions, debugging missing buttons |
| `references/custom-doctype-with-workflow.md` | End-to-end: new doctype + workflow + notifications + fixtures |
| `references/multi-doctype-orchestration.md` | Spawning docs from doc_events, transactions, enqueue |
| `references/gotchas.md` | The long tail of v15 surprises |

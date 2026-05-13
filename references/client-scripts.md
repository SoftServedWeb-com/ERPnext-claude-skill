# Client Scripts

JS that runs in the desk UI when a user opens or interacts with a form. Two delivery paths: shipped as `.js` files in the app, or stored as `Client Script` DocType records in the DB.

## Table of contents
- [The two delivery paths](#the-two-delivery-paths)
- [Event reference](#event-reference)
- [Form-wide events](#form-wide-events)
- [Field events](#field-events)
- [Child table events](#child-table-events)
- [Reading and writing values](#reading-and-writing-values)
- [Custom buttons](#custom-buttons)
- [Dialogs](#dialogs)
- [Filtering Link fields with set_query](#filtering-link-fields-with-set_query)
- [fetch_from for auto-population](#fetch_from-for-auto-population)
- [Common pitfalls](#common-pitfalls)

## The two delivery paths

**App-shipped:** `apps/<app>/<app>/<module>/doctype/<doctype>/<doctype>.js`. Lives alongside the DocType files. Loaded automatically when the form opens. Use this for behavior that's part of the app's contract — version-controlled, code-reviewed, deploys with the app.

**Client Script DocType:** Records in the `Client Script` table. Edited in the Desk UI under "Customize Form" → Client Script (or directly). Apply To = "Form" or "List". Use this for site-local tweaks or per-site customization. Fixture if you want it to deploy: `{"doctype": "Client Script", "filters": [["name", "in", ["My Script"]]]}`.

You can also load arbitrary JS bundles via `app_include_js` in `hooks.py`:
```python
app_include_js = ["/assets/<app>/js/my_bundle.js"]
```
But this is rarely the right choice for form-specific behavior — it pollutes every page. Prefer per-DocType `.js`.

After editing app-shipped `.js`:
- Dev mode: hard-refresh the browser (Ctrl+Shift+R). The dev server serves the file directly.
- Production mode: `bench build` and then `bench --site <site> clear-cache`.

## Event reference

Hooks fire on a `Frm` (form) object. Pattern:

```javascript
frappe.ui.form.on("Sales Order", {
    event_name: function(frm) {
        // frm.doc is the document
    },
    field_name: function(frm) {
        // fires when this field changes
    },
});
```

## Form-wide events

| Event | Fires when |
|-------|-----------|
| `setup` | Once, before doc is loaded — wire one-time form setup |
| `onload` | Every time form opens, before render |
| `onload_post_render` | After form has rendered |
| `refresh` | After every save and on every form re-render. **Run almost everything here.** |
| `validate` | Client-side validation before save. Return false / throw to block. |
| `before_save` | Right before submit-to-server save |
| `after_save` | After server has saved |
| `before_submit` | Before docstatus → 1 |
| `on_submit` | After docstatus → 1 |
| `before_cancel` | Before docstatus → 2 |
| `after_cancel` | After docstatus → 2 |

The most common mistake: putting one-time setup in `refresh` and seeing it run on every save. If you only want it once, gate with `frm.is_new()` or a flag.

## Field events

Use the fieldname directly:

```javascript
frappe.ui.form.on("Sales Order", {
    customer: function(frm) {
        // fires when customer field changes (including initial load if set)
        if (!frm.doc.customer) return;
        frappe.db.get_value("Customer", frm.doc.customer, "customer_group").then(r => {
            console.log("group:", r.message.customer_group);
        });
    },
});
```

The handler fires:
- On user-driven change
- On programmatic `frm.set_value(...)` change
- On initial form load if the field has a value (this catches people off-guard — gate with `!frm.doc.x` guards if you only want the user-action case)

## Child table events

For a child table named "items" of doctype "Sales Order Item":

```javascript
frappe.ui.form.on("Sales Order Item", {
    qty: function(frm, cdt, cdn) {
        const row = locals[cdt][cdn];
        frappe.model.set_value(cdt, cdn, "amount", row.qty * row.rate);
    },
    items_add: function(frm, cdt, cdn) {
        // a row was added
    },
    items_remove: function(frm) {
        // a row was removed
    },
});
```

`cdt` is the child DocType name, `cdn` is the child row's name (a temp id for unsaved rows). `locals[cdt][cdn]` is the row object.

Use `frappe.model.set_value(cdt, cdn, fieldname, value)` to update a child row field — this fires events and refreshes the UI. Direct mutation of `row.fieldname = x` does not refresh the cell.

## Reading and writing values

| Operation | API |
|-----------|-----|
| Read | `frm.doc.fieldname` |
| Write (with events) | `frm.set_value("fieldname", value)` — returns a Promise |
| Write (silent) | `frm.doc.fieldname = value; frm.refresh_field("fieldname");` |
| Reload from server | `frm.reload_doc()` |
| Get a value from another doc | `frappe.db.get_value("DocType", name, "field").then(r => r.message.field)` |
| Get multiple values | `frappe.db.get_value("DocType", name, ["f1", "f2"])` returns `{f1, f2}` |

Prefer `frm.set_value(...)` over direct assignment — it fires field-change events, which is usually what you want. Use direct assignment only when you explicitly want to avoid the event cascade (e.g., to prevent recursion).

## Custom buttons

```javascript
frappe.ui.form.on("Sales Order", {
    refresh: function(frm) {
        if (frm.doc.docstatus === 1 && !frm.doc.custom_compliance_checked) {
            frm.add_custom_button("Run Compliance Check", () => {
                run_compliance_check(frm);
            }, "Actions");  // optional group name
        }
    },
});
```

The third arg groups buttons under a dropdown. Without it, the button sits inline.

To make the button primary-colored: `frm.page.set_inner_btn_group_as_primary("Actions")` or use `frm.change_custom_button_type("Run Compliance Check", null, "primary")`.

Buttons are reset on every form render — that's why `add_custom_button` belongs in `refresh`. Gate the call with conditions, otherwise it appears on every doc regardless of state.

## Dialogs

For collecting input from the user:

```javascript
function run_compliance_check(frm) {
    const d = new frappe.ui.Dialog({
        title: "Compliance Check",
        fields: [
            {fieldname: "checker_name", fieldtype: "Data", label: "Checker", reqd: 1},
            {fieldname: "passed", fieldtype: "Check", label: "Passed?", default: 1},
            {fieldname: "notes", fieldtype: "Text", label: "Notes"},
        ],
        primary_action_label: "Save",
        primary_action(values) {
            frm.set_value("custom_compliance_checked", values.passed);
            frm.set_value("custom_compliance_notes", values.notes);
            frm.save();
            d.hide();
        },
    });
    d.show();
}
```

For a quick one-field prompt, `frappe.prompt(...)`:
```javascript
frappe.prompt(
    {fieldname: "reason", fieldtype: "Text", label: "Reason", reqd: 1},
    (values) => { console.log(values.reason); },
    "Reason for Rejection",
    "Submit",
);
```

## Filtering Link fields with set_query

To restrict what a user can pick in a Link field:

```javascript
frappe.ui.form.on("Sales Order", {
    setup: function(frm) {
        frm.set_query("customer", () => ({
            filters: {disabled: 0, customer_group: "Corporate"},
        }));
    },
});
```

For a Link in a child table:
```javascript
frm.set_query("item_code", "items", (doc, cdt, cdn) => {
    const row = locals[cdt][cdn];
    return {filters: {is_sales_item: 1, item_group: row.item_group || ""}};
});
```

`set_query` belongs in `setup` (or `onload`) — not `refresh`. Setting it on every refresh is wasted work.

## fetch_from for auto-population

For "when X is picked, copy Y from X's record into this form" — most often, this is *not* a client script. It's a `fetch_from` setting on the field in the DocType JSON:

```json
{
    "fieldname": "customer_name",
    "label": "Customer Name",
    "fieldtype": "Data",
    "fetch_from": "customer.customer_name",
    "read_only": 1
}
```

This is declarative, fires automatically, and survives without any JS. Reach for client script only when the logic is conditional ("fetch only if X is set").

## Common pitfalls

- **Event handler doesn't fire** → 90% of the time: typo in the fieldname, or the script is bound to the wrong DocType. Open the JS console — Frappe logs script binding errors.
- **`refresh` fires "too much"** → it does, by design. Gate everything with conditions (`if (frm.doc.docstatus === 1) {...}`). Don't try to make `refresh` "fire once".
- **`onload` not enough — script fires before fields are visible** → use `onload_post_render` or `refresh`.
- **Promise from `frappe.db.get_value` resolves after `refresh` returns** → that's expected; the JS continues async. Don't put `frm.add_custom_button` inside the `.then()` of a get_value call without recognizing that subsequent re-renders may add the button twice. Check `frm.custom_buttons` first.
- **`frm.set_value` returns a Promise** → if you have multiple set_values and need them in order, chain with `.then` or `await`. Synchronous calls work but values may not be fully applied before next read.
- **Editing app-shipped `.js` doesn't take effect** → in production mode, you need `bench build`. In dev mode, hard refresh. Sometimes there's a service worker cache — clear it.
- **Custom button appears on draft AND submitted** when you only wanted submitted → gate with `frm.doc.docstatus === 1` in your `refresh` handler.

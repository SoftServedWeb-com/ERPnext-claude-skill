# Custom DocType with built-in workflow

End-to-end walkthrough for creating a new business object: schema, controller, workflow, notifications, and fixtures. Use this when the user describes a brand-new entity that doesn't exist in stock ERPnext ("Vendor Onboarding", "Project Approval Request", "Compliance Review").

## Decision: ship as code, or as Custom DocType?

- **Code-shipped DocType** — lives at `apps/<app>/<app>/<module>/doctype/<snake_name>/<snake_name>.json`. Loads at app install. Has a controller `.py` and `.js`. Use this when the DocType is part of the app's purpose and will deploy with it.
- **Custom DocType** — created via UI in the "Custom" module, fixtured. Use this for site-local DocTypes that you don't want to ship in code, or when you need to make the schema mutable by site admins.

Almost always for new business objects, the code-shipped path is the right one. The rest of this doc assumes code-shipped.

## Step 1: Create the DocType

Best practical workflow:
1. Open dev site → DocType list → New DocType.
2. Set module to your app's module (this controls where the JSON ends up).
3. Add fields, set field types, set required/in_list_view/etc.
4. Save.

This writes the JSON to `apps/<app>/<app>/<module>/doctype/<snake_name>/<snake_name>.json` automatically (because the module is in your app and `developer_mode: 1` is set).

If `developer_mode` is 0, the DocType is stored as a row in the DocType table instead — and you'd have to fixture it. To convert: set `custom: 0` in the JSON and move it into the app folder.

Alternative: hand-write the JSON. The shape is:

```json
{
 "doctype": "DocType",
 "name": "Project Approval Request",
 "module": "My Module",
 "is_submittable": 1,
 "track_changes": 1,
 "autoname": "format:PAR-{YYYY}-{####}",
 "fields": [
  {"fieldname": "project", "label": "Project", "fieldtype": "Link", "options": "Project", "reqd": 1, "in_list_view": 1},
  {"fieldname": "requested_by", "label": "Requested By", "fieldtype": "Link", "options": "User",
   "default": "user", "read_only": 1},
  {"fieldname": "amount", "label": "Amount", "fieldtype": "Currency", "reqd": 1, "in_list_view": 1},
  {"fieldname": "justification", "label": "Justification", "fieldtype": "Text Editor", "reqd": 1},
  {"fieldname": "section_status", "label": "Status", "fieldtype": "Section Break"},
  {"fieldname": "workflow_state", "label": "Workflow State", "fieldtype": "Link",
   "options": "Workflow State", "read_only": 1, "in_list_view": 1, "in_standard_filter": 1},
  {"fieldname": "reviewer_notes", "label": "Reviewer Notes", "fieldtype": "Text"}
 ],
 "permissions": [
  {"role": "Requester", "read": 1, "write": 1, "create": 1, "submit": 1, "if_owner": 1},
  {"role": "Reviewer", "read": 1, "write": 1},
  {"role": "Manager", "read": 1, "write": 1, "submit": 1, "cancel": 1}
 ]
}
```

A couple of things people forget:
- `is_submittable: 1` if you want docstatus transitions (which workflow needs).
- `track_changes: 1` to populate the audit log (Version DocType).
- `autoname` controls the doc's `name` — use a format string, a field name, or "hash".
- `workflow_state` field: don't add it yourself. The Workflow auto-creates it as a Custom Field when activated. If you must include it (e.g., to control its position), set the field type as `Data` and Frappe will adapt.

## Step 2: The controller

`apps/<app>/<app>/<module>/doctype/project_approval_request/project_approval_request.py`:

```python
import frappe
from frappe.model.document import Document

class ProjectApprovalRequest(Document):
    def validate(self):
        if self.amount <= 0:
            frappe.throw("Amount must be positive")
        if not self.justification or len(self.justification.strip()) < 20:
            frappe.throw("Justification must be at least 20 characters")

    def on_submit(self):
        # Doc is now docstatus 1. Use db_set for any persisted changes.
        self.db_set("submitted_at", frappe.utils.now())

    def on_cancel(self):
        self.db_set("submitted_at", None)
```

For lifecycle events on a custom DocType, you have two choices:
- Define methods on the class (`validate`, `on_submit`, etc.) — cleanest for behavior tied to this DocType
- Register doc_events in `hooks.py` — better for cross-cutting concerns (e.g., audit logging)

For *this* DocType's own logic, the class method path is idiomatic.

## Step 3: The client script

`apps/<app>/<app>/<module>/doctype/project_approval_request/project_approval_request.js`:

```javascript
frappe.ui.form.on("Project Approval Request", {
    refresh: function(frm) {
        // Custom button only for reviewers in the right state
        if (frm.doc.workflow_state === "Pending Review"
            && frappe.user.has_role("Reviewer")
            && !frm.doc.__islocal) {
            frm.add_custom_button("Request More Info", () => {
                frappe.prompt(
                    {fieldname: "info", fieldtype: "Text", label: "What do you need?", reqd: 1},
                    (values) => {
                        frappe.db.set_value(frm.doctype, frm.docname, "reviewer_notes", values.info)
                            .then(() => frm.reload_doc());
                    },
                    "Request More Info",
                );
            });
        }
    },

    project: function(frm) {
        if (!frm.doc.project) return;
        frappe.db.get_value("Project", frm.doc.project, "project_name").then(r => {
            frm.set_value("project_name_cached", r.message.project_name);
        });
    },
});
```

## Step 4: The workflow

Create via UI (see [workflows.md](workflows.md)) or hand-author the fixture:

States: Draft → Pending Review → Pending Manager → Approved (or Rejected)

```json
[{
  "doctype": "Workflow",
  "workflow_name": "Project Approval Request Flow",
  "document_type": "Project Approval Request",
  "is_active": 1,
  "send_email_alert": 1,
  "states": [
    {"state": "Draft", "doc_status": "0", "allow_edit": "Requester"},
    {"state": "Pending Review", "doc_status": "0", "allow_edit": "Reviewer"},
    {"state": "Pending Manager", "doc_status": "0", "allow_edit": "Manager"},
    {"state": "Approved", "doc_status": "1", "allow_edit": "Manager"},
    {"state": "Rejected", "doc_status": "2", "allow_edit": "Requester"}
  ],
  "transitions": [
    {"state": "Draft", "action": "Submit for Review", "next_state": "Pending Review",
     "allowed": "Requester"},
    {"state": "Pending Review", "action": "Send to Manager", "next_state": "Pending Manager",
     "allowed": "Reviewer", "allow_self_approval": 0, "send_email_to_creator": 1},
    {"state": "Pending Review", "action": "Reject", "next_state": "Rejected",
     "allowed": "Reviewer", "allow_self_approval": 0, "send_email_to_creator": 1},
    {"state": "Pending Manager", "action": "Approve", "next_state": "Approved",
     "allowed": "Manager", "allow_self_approval": 0, "send_email_to_creator": 1},
    {"state": "Pending Manager", "action": "Reject", "next_state": "Rejected",
     "allowed": "Manager", "allow_self_approval": 0, "send_email_to_creator": 1}
  ]
}]
```

## Step 5: Notifications

For each state transition you want to surface:

```json
[
 {
  "doctype": "Notification",
  "name": "PAR - Submitted to Reviewer",
  "enabled": 1, "channel": "Email", "send_system_notification": 1,
  "document_type": "Project Approval Request",
  "event": "Value Change", "value_changed": "workflow_state",
  "condition": "doc.workflow_state == 'Pending Review'",
  "subject": "Project Approval Request {{ doc.name }} needs your review",
  "recipients": [{"receiver_by_role": "Reviewer"}],
  "message": "<p>A new request ({{ doc.project }} — {{ frappe.utils.fmt_money(doc.amount) }}) is awaiting your review.</p>"
 },
 {
  "doctype": "Notification",
  "name": "PAR - Approved",
  "enabled": 1, "channel": "Email", "send_system_notification": 1,
  "document_type": "Project Approval Request",
  "event": "Value Change", "value_changed": "workflow_state",
  "condition": "doc.workflow_state == 'Approved'",
  "subject": "Your request {{ doc.name }} was approved",
  "recipients": [{"receiver_by_document_field": "requested_by"}],
  "message": "<p>Your approval request has been approved. You may proceed.</p>"
 },
 {
  "doctype": "Notification",
  "name": "PAR - Rejected",
  "enabled": 1, "channel": "Email", "send_system_notification": 1,
  "document_type": "Project Approval Request",
  "event": "Value Change", "value_changed": "workflow_state",
  "condition": "doc.workflow_state == 'Rejected'",
  "subject": "Your request {{ doc.name }} was rejected",
  "recipients": [{"receiver_by_document_field": "requested_by"}],
  "message": "<p>Your approval request has been rejected. See reviewer notes: {{ doc.reviewer_notes }}</p>"
 }
]
```

## Step 6: Wire fixtures in hooks.py

```python
fixtures = [
    {"doctype": "Role", "filters": [["name", "in", ["Requester", "Reviewer", "Manager"]]]},
    {"doctype": "Workflow State", "filters": [["name", "in",
        ["Pending Review", "Pending Manager", "Approved", "Rejected"]]]},
    {"doctype": "Workflow Action Master", "filters": [["name", "in",
        ["Submit for Review", "Send to Manager", "Approve", "Reject"]]]},
    {"doctype": "Workflow", "filters": [["name", "in", ["Project Approval Request Flow"]]]},
    {"doctype": "Notification", "filters": [["name", "in",
        ["PAR - Submitted to Reviewer", "PAR - Approved", "PAR - Rejected"]]]},
]
```

Order matters — see [fixtures.md](fixtures.md#load-order). Roles first, then states/actions, then workflow, then notifications.

## Step 7: Install + verify

```
bench --site <site> migrate
bench --site <site> clear-cache
bench restart
```

Then on the site:
1. Go to /app/project-approval-request/new and create a doc as a Requester.
2. Submit for Review — confirm the Reviewer gets a notification.
3. As Reviewer, take "Send to Manager" — confirm Manager gets pinged.
4. As Manager, Approve — confirm Requester gets the approved email.

If any step fails, run through [permissions.md](permissions.md#debug-recipe).

## On a fresh site

After all the fixturing, installing the app on a fresh site should set everything up automatically:
```
bench --site newsite.local install-app myapp
bench --site newsite.local migrate
```

This is the test that proves your fixtures are complete. If a state name or role is missing, install will fail on the fresh site even though it works on dev. Always test on a freshly-created site before considering the work done.

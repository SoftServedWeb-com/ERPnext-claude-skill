# Permissions & workflow

Frappe has three overlapping permission layers, and workflow adds a fourth. Knowing how they interact is essential for debugging "user can't see the button" cases.

## The layers

1. **Role Permission Manager** (DocType permissions) — what *roles* can read/write/create/delete/submit/cancel/amend a DocType. Set per role per DocType. The most coarse-grained.
2. **Permission Level (`permlevel`)** — within a single DocType, fields can be grouped under a permlevel (0, 1, 2...). Permissions are then granted *per permlevel*. So you can let Sales User read a doc but only Sales Manager edit specific high-sensitivity fields.
3. **User Permissions** — restricts what records of a Linked DocType a specific user can see. E.g., "User X can only see Customers in the 'Corporate' customer_group". Operates as a data-row filter.
4. **Workflow `allowed`** — only members of the named role can take a given transition. Layered on top of (1).

A user must satisfy **all** layers to take an action. Workflow doesn't bypass DocType perms; it adds another gate.

## DocType permissions in plain English

For each DocType, each role has a row in the permissions table with checkboxes:

| Field | Meaning |
|-------|---------|
| `read` | See the doc in the list view and open it |
| `write` | Edit fields (in editable states) |
| `create` | Make a new doc |
| `delete` | Delete drafts (cancelled docs are deleted differently) |
| `submit` | Move docstatus 0 → 1 |
| `cancel` | Move docstatus 1 → 2 |
| `amend` | Create an amended version of a cancelled doc |
| `print`, `email`, `export`, `report`, `share`, `import` | What it says |
| `if_owner` | Restrict this row to the doc's owner (the user who created it) |

For workflow approvals to work, the approver role typically needs:
- `read` on the DocType (to see the doc)
- `write` on the DocType (to take an action that updates the workflow_state field)
- `submit` if any transition flips docstatus 0 → 1

A common bug: you give "Purchase Manager" only `read` permission and they can't take the Approve transition. They need `write` at minimum.

## How Workflow `allow_edit` interacts

A workflow state can specify `allow_edit: <role>`. This **adds** edit permission for that role *only in this state*. It does not subtract perms. So if `allow_edit: "Purchase Manager"`, that role can edit the doc when it's in this state — even if their default DocType permission lacks `write`.

This is how workflows give different roles edit rights at different stages without permanent DocType-level grants.

## Why the Approve button isn't showing

Most common causes, in rough order of frequency:

1. **User lacks the role** in the transition's `allowed`. Check User → Roles tab.
2. **`allow_self_approval: 0`** and the user is the doc's owner.
3. **The transition's `condition` is falsy** for this doc. Test in `bench console`:
   ```python
   doc = frappe.get_doc("Purchase Order", "PO-0001")
   eval("doc.grand_total >= 5000", {"doc": doc, "frappe": frappe})
   ```
4. **User lacks `write` permission** on the DocType. Workflow can't grant it on its own.
5. **Workflow isn't active** (`is_active: 0`) or the wrong workflow is active.
6. **Cache is stale.** `bench --site <site> clear-cache`. Especially after workflow edits.
7. **DocType is in a state with no transitions** from the user's role — verify the current state by checking `doc.workflow_state`.
8. **User has a User Permission** that restricts them away from this doc's value of some Linked field.

## Debug recipe

```python
# bench --site mysite.local console
from frappe.permissions import has_permission
from frappe.model.workflow import get_workflow, get_transitions

frappe.set_user("user@example.com")     # impersonate

doc = frappe.get_doc("Purchase Order", "PO-0001")
print("read perm:", has_permission(doc.doctype, "read", doc=doc))
print("write perm:", has_permission(doc.doctype, "write", doc=doc))
print("submit perm:", has_permission(doc.doctype, "submit", doc=doc))

wf = get_workflow(doc.doctype)
print("active workflow:", wf.name if wf else None)
print("current state:", doc.workflow_state)
print("available transitions:", get_transitions(doc))
```

`get_transitions(doc)` returns the transitions the current user can take. Empty list → permission/condition issue. Falsy condition is silent; an empty list might mean "no condition matched" rather than "no role allowed."

## permission_query_conditions and has_permission hooks

For document-level filtering that goes beyond User Permissions (e.g., "users can only see their department's records"), wire a custom permission hook:

```python
# hooks.py
permission_query_conditions = {
    "Purchase Order": "myapp.myapp.permissions.purchase_order_query",
}
has_permission = {
    "Purchase Order": "myapp.myapp.permissions.purchase_order_has_permission",
}
```

```python
# myapp/myapp/permissions.py
import frappe

def purchase_order_query(user=None):
    """Return a SQL WHERE clause fragment to filter the list view."""
    if not user:
        user = frappe.session.user
    if "System Manager" in frappe.get_roles(user):
        return ""  # see everything
    dept = frappe.db.get_value("Employee", {"user_id": user}, "department")
    return f"`tabPurchase Order`.department = {frappe.db.escape(dept)}"

def purchase_order_has_permission(doc, ptype, user=None):
    """Return True/False for a specific doc instance."""
    if not user:
        user = frappe.session.user
    if "System Manager" in frappe.get_roles(user):
        return True
    return doc.department == frappe.db.get_value("Employee", {"user_id": user}, "department")
```

`permission_query_conditions` filters lists and reports (efficient, SQL-level). `has_permission` filters individual gets (called per doc). For consistency, write both.

The return from `permission_query_conditions` is injected into the SQL WHERE clause — escape user values rigorously to avoid SQL injection.

## Role inheritance and special roles

Some roles have semantics beyond the docperm grid:

- **System Manager** — bypasses most permission checks. Use sparingly.
- **Administrator** (user, not role) — bypasses everything. Use only for setup and migrations.
- **Guest** — for public-facing endpoints. Almost never the right answer for ERPnext business logic.
- **All** (role) — every user has it implicitly. Useful for "everyone can read" rules.

If you create a custom role (e.g., "Compliance Officer"), fixture it, then assign it via Role Permission Manager. Don't assume new roles inherit anything.

## A debugging shortcut

If a user reports "I can't do X" and you've checked everything:

```python
# bench console as the affected user
frappe.set_user("user@example.com")
doc = frappe.get_doc("DocType", "name")
try:
    doc.run_method("validate")  # or whatever they're trying
except Exception as e:
    frappe.log_error("debug")
    print(repr(e))
```

The traceback usually names the missing perm or the failed condition.

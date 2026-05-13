# Fixtures

Fixtures are how you ship configuration (Workflows, Notifications, Custom Fields, Property Setters, Roles, custom DocTypes) inside an app. On `bench migrate`, Frappe reads the JSON files in `apps/<app>/<app>/fixtures/` and upserts records into the DB.

This is the difference between "it works on my dev site" and "it deploys to staging cleanly".

## Table of contents
- [Declaring fixtures in hooks.py](#declaring-fixtures-in-hookspy)
- [Filter recipes](#filter-recipes)
- [Exporting](#exporting)
- [What you must always fixture together](#what-you-must-always-fixture-together)
- [Load order](#load-order)
- [Debugging](#debugging)

## Declaring fixtures in hooks.py

The `fixtures` variable is a list. Each entry is either:
- A **bare string** = the doctype name (exports all records of that type owned by the app)
- A **dict** with `doctype` + `filters` (exports only records matching the filters)

```python
fixtures = [
    "Custom DocType",                   # bare string

    {"doctype": "Workflow",             # dict with filters
     "filters": [["name", "in", ["PO Approval", "Vendor Onboarding"]]]},

    {"doctype": "Workflow State",
     "filters": [["name", "in", ["Pending Manager", "Pending CFO"]]]},

    {"doctype": "Workflow Action Master",
     "filters": [["name", "in", ["Send to CFO", "Approve", "Reject"]]]},

    {"doctype": "Notification",
     "filters": [["name", "in", ["PO Approved Email", "PO Rejected Email"]]]},

    {"doctype": "Custom Field",
     "filters": [["name", "like", "%-custom_compliance_%"]]},

    {"doctype": "Property Setter",
     "filters": [["module", "=", "My Module"]]},

    {"doctype": "Role",
     "filters": [["name", "in", ["Compliance Officer"]]]},

    {"doctype": "Client Script",
     "filters": [["name", "in", ["PO Compliance Banner"]]]},

    {"doctype": "Server Script",
     "filters": [["name", "in", ["Auto-Set Cost Center"]]]},
]
```

## Filter recipes

The `filters` value is a list of Frappe filter triplets `[field, operator, value]`.

```python
# By exact name match
{"doctype": "Notification", "filters": [["name", "in", ["A", "B"]]]}

# By name pattern (for systematically-named Custom Fields)
# Custom Field names have shape "<DocType>-<fieldname>"
{"doctype": "Custom Field", "filters": [["name", "like", "%-my_app_%"]]}

# By module (clean if all your customizations belong to one module)
{"doctype": "Property Setter", "filters": [["module", "=", "My Module"]]}

# By doc_type for Custom Fields
{"doctype": "Custom Field", "filters": [["dt", "in", ["Sales Invoice", "Purchase Order"]]]}

# By any field
{"doctype": "Workflow", "filters": [["document_type", "=", "Sales Order"]]}
```

**Naming discipline matters.** If you prefix all your custom field names with `custom_my_app_` (or whatever), the `like` filter captures them cleanly. Without a prefix, you'll either over-capture (random fields from your dev site) or have to enumerate by name.

## Exporting

```
bench --site <site> export-fixtures --app <app>
```

This reads `hooks.py`'s `fixtures` list, queries each, and writes one JSON file per doctype to `apps/<app>/<app>/fixtures/`. Filenames are snake_case of the doctype (`workflow.json`, `workflow_state.json`, `notification.json`, `custom_field.json`, etc.).

If `--app` is omitted, it exports for every app in `apps.txt` — usually not what you want.

After exporting, **commit the JSON files**. They are the source of truth.

You can also hand-write the JSON. The shape is "a JSON array of dicts, each dict has `doctype` plus the fields". For most doctypes, the easiest workflow is: build on dev UI, export, commit. For Workflow State / Workflow Action Master (which are trivially small) hand-writing is fine.

## What you must always fixture together

These are dependency clusters. Forgetting any one breaks `bench migrate` on a fresh site.

**Workflow cluster:**
- `Workflow`
- `Workflow State` (for any state name not in defaults: Draft, Approved, Rejected, Pending)
- `Workflow Action Master` (for any action name)
- Any `Role` that doesn't ship with ERPnext (the workflow references roles in `allowed`)
- Any `Custom Field` referenced by `update_field` or that the workflow auto-creates

**Notification cluster:**
- `Notification`
- The Custom Field if your `value_changed` is a custom field
- Any Email Template referenced (if you use one)
- Any Role mentioned in `receiver_by_role`

**Custom DocType cluster:**
- `Custom DocType` is the legacy name for DocTypes created via UI. If your app *owns* the DocType (i.e., its module is your app's module), you don't fixture it — it ships as code in `<app>/<module>/doctype/<name>/<name>.json`. If your DocType is in a system module like "Custom" — fixture it.
- `Custom Field` for any field added to a system DocType
- `Property Setter` for any UI tweak (label change, field order, hidden)

## Load order

`bench migrate` loads fixtures in the order they appear in your `fixtures` list. **Order matters** when records reference each other:

1. `Role` (referenced by everything)
2. `Workflow State`, `Workflow Action Master` (referenced by Workflow)
3. `Custom Field` (referenced by anything using the field)
4. `Property Setter` (cosmetic, late is fine)
5. `Workflow` (after states + actions)
6. `Notification` (after Custom Fields it references)
7. `Client Script`, `Server Script` (after everything else)

If you see "DocType X / record Y does not exist" during migrate, the offender is almost always order — record Y is fixtured later in the list than the record that references it. Move it earlier.

## Debugging

**Fixture file written but not loading**:
- `bench --site <site> migrate` output mentions the file? If not, the file is in the wrong directory. Must be `apps/<app>/<app>/fixtures/`.
- The JSON has a syntax error. Validate: `python -m json.tool apps/<app>/<app>/fixtures/workflow.json`.
- The records exist with different names. Frappe upserts by `name` — if the dev export has `name: "MY-PO-Approval-Workflow"` and the prod DB already has `name: "PO Approval"`, you'll get duplicates instead of updates.

**Record loads but a referenced thing is missing**:
- "Could not find Workflow State 'Pending Manager'" → you didn't fixture the Workflow State, or it's listed *after* the Workflow in `fixtures`. Move it before.

**Migrate succeeds but the workflow/notification doesn't appear**:
- `bench --site <site> clear-cache` — Frappe caches workflow lookups per session.
- Check the `is_active` / `enabled` flag on the record. Defaults are off.

**Re-exporting overwrites your local edits**:
- Yes, that's how export-fixtures works. Treat the fixtures JSON as generated artifacts. Make changes in the UI on dev, then export. Don't hand-edit the exported JSON for ongoing changes.

**Two devs export different selections of fixtures and the JSON files churn**:
- Use tight filters (`name in [...]` with explicit names) rather than broad ones (`module = "Foo"`). Then each dev only re-exports their owned records.
- Or: only one dev exports, and others let the dev branch override.

**A custom field shows up in fixtures even though I didn't add it**:
- Someone (or some installed app) created a Custom Field on a DocType your filter captures. Tighten the `name like` filter.

## A note on production deploys

`bench --site <site> migrate` runs fixtures *every* migrate. Records are upserted by `name`. This means:
- Edits to a fixtured record in the prod UI **get overwritten on next migrate**. If users complain that their changes vanished, this is why. Solution: make the field unfixtured, or stop letting prod users edit fixtured records.
- Deleted records in prod **come back on next migrate** (they're recreated from the fixture). To actually remove, delete from the fixture JSON AND delete from prod manually.

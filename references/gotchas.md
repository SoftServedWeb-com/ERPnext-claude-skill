# v15 gotchas (the long tail)

The most-bitten-by surprises in Frappe / ERPnext v15. Skim this when you're stuck on something that "should just work."

## Lifecycle / hooks

- **`on_submit` cannot mutate `doc`** — use `doc.db_set("field", value)`. Same for all `on_*` events.
- **`bench restart` is mandatory after any Python change** — `hooks.py`, controllers, utilities. Dev server doesn't auto-reload Python.
- **Modifying `doc` in `validate` mutates the about-to-be-saved record, which is desired** — but if a later validator throws, your earlier mutation is rolled back. That's correct, just be aware.
- **`*` doctype hook fires on system doctypes** like Communication, ToDo, File, Version. Filter the doctype inside the handler if you only care about business doctypes.
- **`before_insert` fires before autoname runs in some cases**, so `doc.name` may not be set yet. Use `after_insert` if you need a guaranteed name.
- **Setting `doc.flags.ignore_validate = True` does not skip `validate` doc_events** registered via hooks.py — only the controller's own `validate` method. To skip a hook, gate inside the handler with a flag.

## Workflows

- **One active workflow per DocType.** Activating a new one without deactivating the old one fails migrate. Set `is_active: 0` on the old workflow first.
- **`workflow_state` field is auto-created** — don't pre-create a Custom Field with the same name.
- **Workflow State and Workflow Action Master names must exist** before the Workflow loads. Fixture them earlier in the `fixtures` list.
- **Conditions can't import or do I/O.** Push complex routing logic into `before_validate` that sets a field, then condition on that field.
- **`apply_workflow(doc, action)`** is the programmatic API. Setting `doc.workflow_state` directly skips all validation, role checks, and notifications — only use for data migration.
- **Self-approval defaults to allowed.** Almost no compliance team wants this. Set `allow_self_approval: 0` on Approve transitions.
- **Workflow doesn't override docperm.** Approver also needs `write` (or `submit`) at the DocType level. Workflow's `allow_edit` adds permission *for the state*, not generally.
- **Doc cancel reaches via workflow transitions with `doc_status: 2`**, not via the normal Cancel button (which is hidden). If users complain "I can't cancel", they need an action that transitions to a doc_status: 2 state.
- **Override existing built-in status field** — set `override_status: 1` on the workflow only if you want the workflow state to *replace* ERPnext's `status` field on the doc. Usually leave this 0.

## Permissions

- **Roles vs User Permissions are different things.** Role = "Sales Manager can read Customer". User Permission = "User X can only see Customers in CG=Corporate". Both apply.
- **`if_owner` is checked against `doc.owner`, the creator** — not the current user. Owners can't change.
- **System Manager bypasses most checks.** Don't develop or debug as System Manager; you'll miss real-user breakage.
- **`permission_query_conditions` returns SQL** — escape user-supplied values rigorously (`frappe.db.escape`), or you have SQL injection.
- **`has_permission` hook returning False blocks read/write, but the user still sees the doc in list views unless `permission_query_conditions` also filters it.** Always pair them.
- **Permission Level (permlevel)** — fields with `permlevel: 1` need their own permission row. Common surprise: you set up a "Manager" role with full perms at level 0 but they still can't edit a permlevel-1 field. Add a level-1 permission row.

## Fixtures

- **Filters must be specific or you over-capture.** Bare `"Custom Field"` exports every custom field on the dev site, including ones from other apps.
- **Order matters in `fixtures` list.** Roles, then Workflow State / Action Master, then Workflow, then Notifications.
- **Re-export blows away your edits.** Fixture JSON files are generated artifacts. Edit in the UI on dev, then re-export.
- **`bench migrate` upserts fixtures.** A fixtured record edited in prod gets overwritten on next migrate. If users complain about lost changes, this is why.
- **Custom DocType vs DocType in module.** If your DocType lives in a system module like "Custom", fixture it. If it's in your app's own module, it ships as code (don't double-fixture).

## DocType schema

- **DocType JSON is rewritten by Frappe on every save in dev mode.** Hand-edits get reformatted. Use the UI to add fields, then commit the auto-generated JSON.
- **Field renames break data.** Frappe doesn't migrate column names. Use a patch.
- **`is_submittable: 1` is required** for workflows (workflows need docstatus transitions).
- **`autoname: "field:fieldname"`** uses a field's value as the doc name. If the field is unset or empty at insert, autoname fails — usually with a confusing error. Set a `default` on the field or use `format:` instead.
- **`track_changes: 1`** populates the Version doctype on every save. Useful for audit; consumes DB rows.
- **Removing a field** from the DocType JSON doesn't drop the DB column. The data stays orphan. Use a patch + manual DROP if you want it gone.
- **Child table renames** require careful coordination — the child's `parenttype` field references the parent doctype name.

## Notifications

- **Notification fires under "Value Change" only if the field actually changed**, not on initial set. For "fire when doc is created with X", use "New" event with a condition.
- **Days After/Before notifications require the scheduler.** If it's not running, they never fire.
- **Outgoing emails need an Email Account configured.** Without one, Notification Log records a row but no email goes out. Check Email Domain and Email Account.
- **Jinja errors in subject/message** are silent — logged to Error Log. Always test render with `frappe.render_template` before fixturing.
- **`receiver_by_role` skips users without an email address.** Disabled users are also skipped.

## Client scripts

- **`refresh` fires on every save and every navigation.** Don't put one-time setup there without a guard.
- **App-shipped `.js` edits don't show up in prod mode without `bench build`.**
- **`frm.set_value` is async (returns a Promise).** Sequence dependent calls with `.then` or `await`.
- **Field handler fires on initial load with a value**, not only on user-driven changes. Add guards if you only want user-driven cases.
- **`add_custom_button` is reset on each refresh** — that's why you re-add in `refresh`. But check `frm.custom_buttons` to avoid double-adding when triggered async.

## Background jobs / scheduler

- **Worker process must be running.** `bench start` runs it in dev; supervisor in prod. `bench doctor` diagnoses.
- **`enqueue_after_commit=True`** is needed when enqueuing from a doc_event, or the job runs before the original txn commits and the spawned doc-load fails.
- **Long-running jobs default to a 5-minute timeout.** Pass `timeout=` explicitly for anything that might exceed it.
- **Scheduler runs at minute boundaries, not exactly on the cron expression.** A `*/5` cron may fire 0–60s late.
- **`bench --site X disable-scheduler`** for data migrations — otherwise the scheduler may fire mid-migration and corrupt state.

## Bench / deployment

- **`bench update` without `--no-backup` is slow** and may fail on disk-full sites. For dev iterations, prefer targeted commands.
- **`bench restart`** is necessary after editing any Python in any installed app. The dev server is a Werkzeug autoreloader for app code but not for hooks (which are registered at boot).
- **`bench build` is for production mode.** In dev (`developer_mode: 1`), JS is served unbuilt — don't run `bench build` unnecessarily, it's slow.
- **`bench --site X migrate` is safe to re-run** — idempotent. If a migrate fails partway, fix the issue and rerun.
- **Patches run once per site.** Their names are recorded in the `tabPatch Log` table. If you need to re-run a patch, delete its row from `tabPatch Log` first.

## Naming and conventions

- **DocType names are case-sensitive in some places.** `frappe.get_doc("Sales Order", ...)` is canonical; `frappe.get_doc("sales order", ...)` may or may not work depending on the call path.
- **Module names with spaces** become snake_case in the filesystem. "My Module" → `my_module`. The JSON keeps the spaced version.
- **App names are snake_case.** "Ahoy Accounts" → `ahoy_accounts`. Hooks.py references use the snake_case form.
- **Custom Field name format is `<DocType>-<fieldname>`.** A Custom Field named `Customer-loyalty_tier` is the `loyalty_tier` field on Customer. Use this in `name like %-loyalty_tier%` filters.

## Migrations and data

- **Patches don't have access to the latest DocType JSON until *after* `frappe.reload_doctype`.** If your patch reads a new field, call `frappe.reload_doc("module", "doctype", "doctype_name")` first.
- **Bulk operations should use `frappe.db.sql(...)`** or `frappe.db.set_value(...)`, not `frappe.get_doc(...).save()` in a loop. The latter is 100x slower for large datasets.
- **Don't disable the scheduler permanently** — re-enable after data migrations.
- **`frappe.db.commit()`** is rarely needed. The request-level transaction handles it. Only commit explicitly in long-running scripts where you want partial progress saved (e.g., a patch processing 10M rows).

## Misc

- **Test on a fresh site** before declaring work done. Many bugs only show up there: missing fixtures, role assumptions, hardcoded site names.
- **Error Log is your friend.** When something silently fails (notifications, scheduler, hooks), check `/app/error-log` for the traceback.
- **`bench --site <site> show-pending-jobs`** when "background things aren't happening".
- **Cache busts happen in two layers** — `bench --site <site> clear-cache` (Frappe Python cache) and `bench --site <site> clear-website-cache` (Redis-backed website cache). Sometimes you need both.
- **Production mode vs developer mode** — set in site_config.json. Most "works in dev, fails in prod" bugs are JS build issues or developer_mode-only features. Verify behavior in production mode before shipping.

# bench command cheatsheet

`bench` is the Frappe CLI. Almost everything you do as a Frappe developer goes through it. These commands assume you are inside a `frappe-bench/` directory.

## Lifecycle

```
bench start                  # foreground dev mode — web + worker + scheduler + redis (reads Procfile)
bench restart                # restart all processes (supervisor in prod, foreground devs Ctrl-C and rerun)
bench update                 # pull, install deps, migrate, build, restart
bench update --no-backup     # same, skip the safety backup
bench update --reset         # discard local changes in apps (destructive — confirm intent first)
```

In dev mode you usually run `bench start` in one terminal and other commands in another. In prod, supervisor manages everything and you use `bench restart` after any code change.

## Site management

```
bench new-site mysite.local                                    # create a new site
bench --site mysite.local install-app erpnext                  # install an app on a site
bench --site mysite.local uninstall-app myapp                  # uninstall (destructive on data)
bench --site mysite.local migrate                              # apply schema, fixtures, patches
bench --site mysite.local clear-cache                          # bust frappe's in-memory caches
bench --site mysite.local clear-website-cache                  # bust website cache
bench --site mysite.local set-config <key> <value>             # write to site_config.json
bench --site mysite.local set-admin-password <password>        # reset Administrator password
bench --site mysite.local backup                               # snapshot DB + files
bench --site mysite.local backup --with-files                  # include file attachments
bench --site mysite.local restore <sql-dump>                   # restore from a backup
bench --site mysite.local drop-site                            # delete the site entirely (destructive)
bench use mysite.local                                         # set as default site for unflagged commands
```

`--site` is required for anything that touches a specific tenant. After `bench use`, you can omit it.

## App management

```
bench new-app myapp                          # scaffold a new app at apps/myapp/
bench get-app https://github.com/owner/repo  # fetch an app from git
bench get-app --branch develop owner/repo    # specific branch
bench remove-app myapp                       # remove from apps.txt + delete folder
```

After `bench new-app`, the new app is on disk but not installed on any site. `bench --site <site> install-app myapp` actually installs it.

## DocType + data

```
bench --site mysite.local reload-doctype "Sales Order"   # reload a single doctype's JSON without full migrate
bench --site mysite.local reload-doc <module> <doctype> <name>  # reload a specific record JSON
bench --site mysite.local export-doc <doctype> <name>    # write a specific doc to JSON for fixturing
bench --site mysite.local export-fixtures                # write all configured fixtures
bench --site mysite.local export-fixtures --app myapp    # only one app's fixtures
bench --site mysite.local export-csv <doctype> <path>    # dump a doctype to CSV
bench --site mysite.local import-csv <path>              # bulk-import CSV
```

## Running Python

```
bench --site mysite.local console
# Drops you into a Python REPL with `frappe` imported and the site set.
# Useful for poking at the DB, calling functions, debugging.

bench --site mysite.local execute myapp.myapp.tasks.send_daily_reminder
# Runs a function once. The function must take no required args (or pass via --args/--kwargs).
bench --site mysite.local execute myapp.myapp.tasks.process_invoice --kwargs "{'invoice':'SI-0001'}"

bench --site mysite.local mariadb            # MySQL shell on the site's DB
bench --site mysite.local postgres           # Postgres shell (if using PG)
```

For ad-hoc data fixes, `bench console` is your friend. Don't write throwaway scripts; just exec it.

## Background jobs

```
bench --site mysite.local show-pending-jobs              # what's queued
bench --site mysite.local doctor                         # diagnose worker / scheduler / cache health
bench worker --queue default                             # run a worker in foreground (debug)
bench schedule                                           # run the scheduler in foreground (debug)
bench --site mysite.local enable-scheduler               # toggle scheduler on
bench --site mysite.local disable-scheduler              # toggle off (data migrations: do this)
```

In dev mode, `bench start` runs the worker and scheduler. If background jobs aren't firing, run `bench doctor` first.

## Build + assets

```
bench build                                  # rebuild JS/CSS bundles for all apps (production mode)
bench build --app myapp                      # rebuild just one app
bench build --watch                          # watch + rebuild on change (dev)
bench setup requirements                     # reinstall Python deps
bench setup requirements --node              # reinstall Node deps
```

In dev mode (`developer_mode: 1` in site_config), JS is served unbuilt — no rebuild needed. In production mode, you must `bench build` after any `.js` change.

## Config

```
bench --site mysite.local set-config developer_mode 1
bench --site mysite.local set-config allow_tests 1
bench --site mysite.local set-config server_script_enabled 1
bench --site mysite.local set-config maintenance_mode 1      # block all requests
bench --site mysite.local set-config disable_async true      # synchronous mode for debugging
```

`set-config` writes to `sites/<site>/site_config.json`. To set a global default (across all sites), edit `sites/common_site_config.json` directly.

## Patches

```
bench --site mysite.local migrate                                  # runs unrun patches
bench --site mysite.local execute frappe.recorder.patches.run      # specific patch path
```

Patches live in `apps/<app>/<app>/patches/<version>/<patch_name>.py`. List them in `apps/<app>/<app>/patches.txt`. They run once per site. Use them for data migrations that aren't pure schema changes.

## Testing

```
bench --site mysite.local run-tests --app myapp                # run all tests in the app
bench --site mysite.local run-tests --doctype "Sales Invoice"  # tests for one doctype
bench --site mysite.local run-tests --module myapp.tests.test_workflow  # a specific module
```

Tests run inside a test site automatically (Frappe creates one with `_test` suffix). Set `allow_tests: 1` first.

## When to run what after your edits

| You edited | Run |
|------------|-----|
| `hooks.py` (any) | `bench --site <site> migrate && bench restart` |
| `apps/<app>/<app>/<module>/doctype/<dt>/<dt>.py` (controller) | `bench restart` |
| `apps/<app>/<app>/<module>/doctype/<dt>/<dt>.js` | dev: hard refresh browser. prod: `bench build` |
| `apps/<app>/<app>/<module>/doctype/<dt>/<dt>.json` (schema) | `bench --site <site> migrate` (or `reload-doctype` for a single DT) |
| `fixtures/*.json` | `bench --site <site> migrate` |
| `patches.txt` | `bench --site <site> migrate` |
| Anything Python (utilities, hooks, controllers) | `bench restart` is mandatory |

Skipping `bench restart` is the #1 cause of "my changes don't work." Always include it in your delivered instructions.

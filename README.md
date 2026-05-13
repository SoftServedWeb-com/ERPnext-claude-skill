# erpnext-workflow-automation

A Claude skill for building **ERPnext v15 / Frappe** workflow automation: approval state machines, `doc_events` hooks, client scripts, Notification doctype alerts, scheduled jobs, fixtures, and multi-doctype business processes.

The skill teaches Claude *which* Frappe mechanism to reach for, the file paths and schemas it needs to touch, and the `bench` commands that have to run afterward — all the things a v15 developer keeps in their head but Claude doesn't, by default.

## What it covers

- **Frappe Workflows** — state machines on any DocType, with conditional routing and role-based transitions
- **`doc_events` in `hooks.py`** — `validate`, `before_save`, `on_submit`, `on_cancel`, `on_update_after_submit`, etc.
- **Client Scripts** — form behavior in the desk UI (`frappe.ui.form.on(...)`)
- **Notification doctype** — declarative email/system/Slack alerts
- **Server Scripts** — site-local Python automation in the sandbox
- **Fixtures** — exporting customizations as JSON for version control
- **Multi-doctype orchestration** — spawning related documents, transaction boundaries, background jobs
- **Custom DocTypes with built-in workflow** — end-to-end new business objects
- **v15-specific gotchas** — the things that bite people

## Installation

Skills live in a directory that Claude Code (or another Claude harness) discovers at startup. The simplest install is to clone this repo into your user-level skills directory:

### Claude Code

```bash
# clone into your user skills directory
git clone https://github.com/<your-username>/erpnext-workflow-automation \
    ~/.claude/skills/erpnext-workflow-automation
```

On Windows, the equivalent is `%USERPROFILE%\.claude\skills\erpnext-workflow-automation`.

Restart Claude Code (or start a new conversation) and the skill becomes available. You can verify by asking Claude "what skills do you have?" — `erpnext-workflow-automation` should appear in the list.

### As a Claude plugin

If you want to bundle this with other skills as a plugin, place this directory inside your plugin's `skills/` folder. See [Claude plugins documentation](https://docs.anthropic.com/) for the manifest format.

### Per-project

To make the skill available only inside a specific project, clone into the project's `.claude/skills/` directory instead.

## How Claude uses it

When you ask Claude something that matches the skill's triggers — anything from "set up an approval workflow for Purchase Order" to "auto-create a Delivery Note when Sales Order submits" — Claude reads `SKILL.md`, follows its decision tree to pick the right Frappe mechanism, then dives into the relevant file in `references/` for the schema or pattern details. It produces:

- a one-sentence diagnosis ("This is a Frappe Workflow with conditional routing, plus a Notification on Approved")
- the exact file paths and edits in your app
- JSON fixtures wired into `hooks.py`
- the `bench` commands to run, with an explanation of what each does
- a flag for any non-obvious tradeoff (permission bypass, self-approval, sync vs background)

## Repo layout

```
erpnext-workflow-automation/
├── SKILL.md                      # the skill Claude reads
├── references/                   # progressive-disclosure deep-dives
│   ├── workflows.md
│   ├── doc-events.md
│   ├── client-scripts.md
│   ├── notifications.md
│   ├── fixtures.md
│   ├── bench-cheatsheet.md
│   ├── permissions.md
│   ├── custom-doctype-with-workflow.md
│   ├── multi-doctype-orchestration.md
│   └── gotchas.md
├── evals/
│   └── evals.json                # test prompts used to validate the skill
├── LICENSE
├── README.md
└── .gitignore
```

## Contributing

Issues and PRs welcome. The skill targets **ERPnext / Frappe v15** specifically — v14 and earlier have meaningful differences (Server Script defaults, workflow internals, fixture loading order) and aren't covered here.

If you find a Frappe gotcha that bit you, adding it to `references/gotchas.md` with the symptom and the cause is the most valuable contribution.

## License

[MIT](LICENSE).

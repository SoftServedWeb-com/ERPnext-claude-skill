# Notifications

The Notification DocType is the declarative way to send emails, system alerts, or Slack/SMS messages on document events or schedules. For anything beyond what it can express, fall back to `frappe.sendmail(...)` from a doc_event or Server Script.

## Table of contents
- [The Notification DocType](#the-notification-doctype)
- [Event types](#event-types)
- [Channels](#channels)
- [Recipients](#recipients)
- [Jinja context](#jinja-context)
- [Fixture example](#fixture-example)
- [Programmatic emails](#programmatic-emails)
- [Testing without spamming inboxes](#testing-without-spamming-inboxes)
- [Common pitfalls](#common-pitfalls)

## The Notification DocType

Key fields:

| Field | Purpose |
|-------|---------|
| `subject` | Jinja-rendered subject line |
| `document_type` | The DocType to watch |
| `event` | When to fire (see below) |
| `value_changed` | For "Value Change" events, the field to watch |
| `condition` | Optional Jinja or Python expression — only send if truthy |
| `days_in_advance` / `date_changed` | For date-based events |
| `channel` | "Email", "System Notification", "Slack", "SMS" |
| `recipients` | Child table of recipient rules |
| `message` | Jinja-rendered HTML body |
| `attach_print` | Attach a print of the doc as PDF |
| `print_format` | Which print format to attach |
| `send_system_notification` | Also send a bell-icon notification |
| `enabled` | 1 / 0 |

## Event types

| Event | Fires |
|-------|-------|
| `New` | When a doc is inserted |
| `Save` | On every save (incl. updates) |
| `Submit` | When docstatus → 1 |
| `Cancel` | When docstatus → 2 |
| `Value Change` | When `value_changed` field changes to anything (or to a specific value via `condition`) |
| `Days After` | N days after `date_changed` field |
| `Days Before` | N days before `date_changed` field |
| `Date Change` | On a date field change |
| `Method` | Manually triggered via `frappe.get_doc("Notification", "<name>").send(doc)` |
| `Custom` | For Slack/system notifications with custom triggers |

For workflow transitions, use **Value Change** with `value_changed: "workflow_state"` and `condition: "doc.workflow_state == 'Approved'"`. This fires once on the transition.

## Channels

- **Email** — uses the site's outgoing email account. Configure under Email Account.
- **System Notification** — the bell icon. Also lands in the user's inbox in the Frappe UI.
- **Slack** — posts to a configured Slack webhook URL. Set up under Slack Webhook URL.
- **SMS** — uses the SMS Settings doctype to dispatch via an SMS gateway.

Set `send_system_notification: 1` on an Email notification to *also* fire a bell notification — useful for in-app awareness without making people open email.

## Recipients

The `recipients` child table accepts these patterns:

```json
{"recipients": [
    {"receiver_by_role": "Accounts Manager"},                      // role → all users with role
    {"receiver_by_document_field": "owner"},                       // a user field on the doc
    {"receiver_by_document_field": "custom_approver"},             // your custom Link to User field
    {"receiver_by_document_field": "supplier.supplier_primary_contact.email_id"}, // dotted = follow links
    {"email": "finance-team@example.com"},                         // hardcoded address
    {"cc": "audit@example.com"},                                   // CC instead of To
    {"bcc": "archive@example.com"}                                 // BCC
]}
```

The dotted-path syntax for `receiver_by_document_field` follows Link fields. `supplier.supplier_primary_contact.email_id` reads the doc's Supplier, follows to that supplier's primary contact, and uses the contact's email_id.

## Jinja context

The `subject`, `message`, and `condition` fields are rendered with Jinja. Available variables:

- `doc` — the document
- `user` — the user the notification fires for (the recipient, not necessarily the actor)
- `frappe` — the frappe module (limited surface)
- `frappe.utils` — utility functions (fmt_money, format_date, etc.)
- `_` — translator function

Common patterns:

```
Subject: Purchase Order {{ doc.name }} ({{ frappe.utils.fmt_money(doc.grand_total, currency=doc.currency) }})
```

```html
<p>Hi {{ user.first_name }},</p>
<p>The PO <a href="{{ frappe.utils.get_url() }}/app/purchase-order/{{ doc.name }}">{{ doc.name }}</a>
   for <b>{{ doc.supplier_name }}</b> has been approved.</p>
<p>Total: {{ frappe.utils.fmt_money(doc.grand_total, currency=doc.currency) }}</p>
```

```
Condition: doc.workflow_state == "Approved" and doc.grand_total > 10000
```

The condition is evaluated as Python (not Jinja). It has access to `doc` and a small sandboxed subset.

## Fixture example

`apps/<app>/<app>/fixtures/notification.json`:

```json
[
 {
  "doctype": "Notification",
  "name": "PO Approved - Notify Requester",
  "enabled": 1,
  "channel": "Email",
  "send_system_notification": 1,
  "document_type": "Purchase Order",
  "event": "Value Change",
  "value_changed": "workflow_state",
  "condition": "doc.workflow_state == 'Approved'",
  "subject": "Purchase Order {{ doc.name }} approved",
  "recipients": [
   {"receiver_by_document_field": "owner"},
   {"cc": "finance@example.com"}
  ],
  "message": "<p>Hi {{ user.first_name }},</p><p>Your PO <b>{{ doc.name }}</b> for <b>{{ doc.supplier_name }}</b> ({{ frappe.utils.fmt_money(doc.grand_total, currency=doc.currency) }}) has been approved.</p>"
 },
 {
  "doctype": "Notification",
  "name": "PO Rejected - Notify Requester",
  "enabled": 1,
  "channel": "Email",
  "document_type": "Purchase Order",
  "event": "Value Change",
  "value_changed": "workflow_state",
  "condition": "doc.workflow_state == 'Rejected'",
  "subject": "Purchase Order {{ doc.name }} rejected",
  "recipients": [
   {"receiver_by_document_field": "owner"}
  ],
  "message": "<p>Your PO <b>{{ doc.name }}</b> has been rejected. Please review and resubmit if needed.</p>"
 }
]
```

Register in `hooks.py`:
```python
fixtures = [
    {"doctype": "Notification", "filters": [["name", "in",
        ["PO Approved - Notify Requester", "PO Rejected - Notify Requester"]]]},
]
```

## Programmatic emails

When the Notification doctype can't express what you need (e.g., dynamic recipient lists, conditional attachments, custom templating logic), use `frappe.sendmail` from a doc_event:

```python
import frappe

def on_submit(doc, method):
    recipients = get_dynamic_recipients(doc)  # your logic
    frappe.sendmail(
        recipients=recipients,
        subject=f"PO {doc.name} approved",
        message=frappe.render_template(
            "templates/emails/po_approved.html",
            {"doc": doc},
        ),
        reference_doctype=doc.doctype,
        reference_name=doc.name,
        attachments=[
            frappe.attach_print(doc.doctype, doc.name, print_format="Standard")
        ],
    )
```

`reference_doctype` and `reference_name` make the email show up in the doc's communication timeline — useful for audit.

For high-volume or slow emails, push to a background job:
```python
frappe.enqueue(
    "frappe.sendmail",
    recipients=recipients,
    subject=...,
    message=...,
    queue="long",
)
```

## Testing without spamming inboxes

A few options:
1. **Email Domain → Mock** — set up an Email Account that points to a catchall like Mailtrap or MailHog. Configure once, all outgoing mail captures there.
2. **`frappe.flags.in_test`** — when set, Frappe skips outbound email. Set in `bench --site <site> console` for ad-hoc tests.
3. **Disable Notifications** — flip `enabled: 0` on a specific Notification when iterating.
4. **Notification Log** — the `Notification Log` DocType records every notification fired (system + email). Useful for verifying without checking actual inboxes.

## Common pitfalls

- **Notification doesn't fire** → check `enabled: 1`, then check the `event` matches (e.g., "Value Change" needs the field to actually change — initial set on a new doc is the "New" event, not "Value Change"). Check the `condition` — Frappe silently skips on falsy.
- **Email doesn't send but notification log shows it** → outbound Email Account not configured for the relevant user/domain. Check Email Domain + Email Account.
- **Recipient list is empty** → `receiver_by_role` filters out users without an email address. Roles assigned to disabled or no-email users won't get pinged.
- **Jinja error in subject** → the notification fails silently but logs to `Error Log`. Always test with a `frappe.render_template` call in `bench console` before fixturing.
- **Notification fires too often** → for "Save" event on a frequently-saved doc, you'll spam. Switch to "Value Change" on the field that actually matters.
- **`value_changed` and `condition` interact** → the notification fires if `value_changed` changed AND `condition` is truthy. To only send when transitioning *to* a state, use `condition: doc.workflow_state == 'Target'`. If you want "any change" without `condition`, leave `condition` empty.
- **Days After/Before relies on the scheduler** — these aren't real-time. They fire when the daily scheduler runs. If your worker isn't running, they never fire.

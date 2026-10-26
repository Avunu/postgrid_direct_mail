---
title: How it works
description: The design behind Direct Mail via PostGrid, from the print view and the Notification extension to mailing a letter, the webhook and the install hooks.
nav_title: How it works
order: 10
updated: 2026-10-26
---

This page describes the design behind the code, for developers. For commands and tests, see [Develop Direct Mail via PostGrid](README.md).

## The pieces

| Piece                  | Hook                                                               | Code                                                                       |
| ---------------------- | ------------------------------------------------------------------ | -------------------------------------------------------------------------- |
| Mail button and dialog | `page_js` for `print`                                              | `public/js/print_view.js`                                                  |
| Notification form      | `doctype_js` for Notification                                      | `public/js/notification.js`                                                |
| Mailed Letter channel  | `extend_doctype_class` for Notification                            | `postgrid_integration/hooks/notification.py`                               |
| Mailing and webhook    | Whitelisted methods                                                | `postgrid_integration/doctype/postgrid_settings/__init__.py`               |
| Settings and webhooks  | The PostGrid Settings controller and its form action               | `postgrid_integration/doctype/postgrid_settings/postgrid_settings.py`      |
| Timeline card          | `additional_timeline_content` for every DocType                    | `get_timeline_content` and `public/js/templates/direct_mail_timeline.html` |
| Install and uninstall  | `after_install`, `after_sync`, `after_migrate`, `before_uninstall` | `install.py` and `setup/custom_fields.py`                                  |

The paths are relative to the `postgrid_direct_mail` package.

## The print view

`print_view.js` subclasses Frappe's `frappe.ui.form.PrintView`, which exists only on the print page. On `make` it fetches the defaults (`get_postgrid_defaults`, which maps the settings to PostGrid's terms such as `insert_blank_page` and `standard_double_window`) and asks whether ERPNext is installed. On `show` it adds the **Mail** button when the document's `docstatus` is not 0.

The dialog's defaults come from the document's `company_address` (the sender; with ERPNext installed and no `company_address`, the default address of the document's company, the user's default company or the global default company, from `erpnext.setup.doctype.company.company.get_default_company_address`), its `customer_address`, `shipping_address`, `billing_address`, `supplier_address` or `address` (the recipient, the first one set), and its `contact_person`. **Mail via PostGrid** calls `mail_letter` with the print format selected in the print view.

## Mailing a letter

`mail_letter` takes the document, the print format, the sender and recipient Address, an optional Contact, the options (`pg_parameters`, or the settings defaults when there are none) and an optional cover letter print format.

1. It renders the print format to PDF with `frappe.get_print`, so Frappe's print permission and docstatus rules apply. With a cover letter it renders that too and merges them with pypdf: cover letter, a blank page when the letter is double-sided, then the document.
2. It sends the letter with PostGrid's Python client (`postgrid-python` 1.x), using the API key of the current **Mode**. The Address Title goes as the company name and the Contact's first and last name with the recipient. The ids PostGrid gives the two addresses are stored in their `postgrid_id` fields.
3. It records the request and PostGrid's response in an Integration Request (service PostGrid, status Queued, or Failed with the error).
4. On success it adds a Communication of type **Mailed Letter** to the document, with PostGrid's letter id as `message_id` and the delivery status Scheduled, and returns the alert the dialog shows.

A PostGrid error is caught and reported in the Integration Request and the Error Log; errors before the call (a missing key, an address without a country, a refused print) are raised.

## Mailed Letter notifications

`DirectMailNotification` extends Frappe's Notification controller. Frappe runs every enabled notification itself, on the document events (`Document.run_notifications`) and daily for Days Before and Days After, and sends each through `send_notification_by_channel`. The extension mails the document there when the channel is Mailed Letter and the event is one of New, Save, Submit, Cancel, Days Before or Days After, or Method with `after_insert`, `on_update`, `on_submit` or `on_cancel`. It then calls Frappe's own method, which sends nothing on a channel it does not know but still sends the system notification. Value Change, Custom and the Minutes events are left out on purpose: before the app moved onto Frappe's notification run it never mailed on them, and turning them on would start mailing, and paying for, letters on sites that have such notifications.

The address comes from the document field the notification names, the options from the notification's own fields. Every failure is caught and written to the Error Log, so a notification never stops the save that fired it.

## The webhook

Saving PostGrid Settings generates the webhook secret on the first save and registers a webhook for the current mode when that mode has no id yet. The webhook subscribes to the created and updated events of letters, postcards, cheques and return envelope orders, and points at the guest method `webhooks`.

PostGrid posts each event as a JWT signed with the secret. `webhooks` refuses the call (`AuthenticationError`) when there is no secret or the signature does not verify, before reading anything. It maps the letter's status (`ready`, `printing`, `processed_for_delivery`, `completed`, `cancelled`) to an Integration Request status and a Communication delivery status, found by the letter id; an event with any other status, such as a return envelope order's, is acknowledged and ignored. With **Save PDF Files** it attaches PostGrid's PDF to the Communication once, and with **Update Addresses** it writes PostGrid's verified sender and recipient addresses back to the Addresses with the matching `postgrid_id`.

**Reset Webhooks** (`reset_webhooks`, POST only, write permission on the settings) works through each mode that has a key on its own: it deletes the recorded webhook, a 404 counting as already gone, then registers a new one, and stores each mode's outcome with `db_set` so that a failure in one mode never undoes the other.

## Install, migrate and uninstall

- **Install.** `after_install` adds the **Direct Mail** card and PostGrid Settings link to the Integrations workspace, and adds **Mailed Letter** to the options of Notification `channel` and Communication `communication_type` with Property Setters, appending to an existing setter when there is one. Frappe syncs `custom/*.json` after `after_install`, so `after_sync` then marks the shipped custom fields system-generated.
- **Migrate.** `after_migrate` marks the custom fields system-generated again (Frappe's sync inserts a new field with the flag off) and re-adds the options if they are missing.
- **Uninstall.** `before_uninstall` removes the workspace link (and the card when nothing else is under it), takes **Mailed Letter** out of the two option lists (deleting the Property Setter when the options are back to standard), and deletes exactly the shipped custom fields. Frappe then deletes the module, its DocType and the records linked to it, such as the Property Setter on Notification `recipients`.

None of these hooks commits: Frappe's installer, migrator and uninstaller commit the transaction they run in.

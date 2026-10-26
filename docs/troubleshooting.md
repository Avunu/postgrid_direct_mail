---
title: Limitations and troubleshooting
description: What Direct Mail via PostGrid does not do, and the messages you can meet when mailing, saving the settings or resetting webhooks, with their causes and fixes.
nav_title: Troubleshooting
order: 8
updated: 2026-10-26
---

## Limitations

- Only letters are supported. Postcards and cheques are not implemented, although the webhook is registered for their events and ignores them.
- The **Mail** button appears only on submitted or cancelled documents, so a DocType that cannot be submitted can only be mailed by a [Notification](notifications.md).
- A Mailed Letter notification mails on the New, Save, Submit, Cancel, Days Before and Days After events, and on a Method event for `after_insert`, `on_update`, `on_submit` or `on_cancel`. The Value Change, Minutes Before, Minutes After and Custom events don't mail.
- Without ERPNext, when a document has no `company_address`, the **From Address** in the Mail dialog starts empty and has to be picked by hand.
- Delivery updates need the site to be reachable by PostGrid over the internet.
- The **To Address Field** and **To Contact Field** of a notification list only the DocType's own Link fields, not fields of its child tables.

## Troubleshooting

### There is no Mail button on the print view

The document is a draft, or its DocType cannot be submitted. Submit the document, or mail it with a [Notification](notifications.md). For a cancelled document, the print view itself needs **Allow Print for Cancelled** in Print Settings.

### A letter was not sent

The Mail dialog shows "Failed to send letter, please check the Integration Request for details". PostGrid refused the letter or could not be reached. Open the Integration Request the alert links to: its **Error** and **Output** hold PostGrid's reason. The same failure is in the Error Log as "PostGrid Error". Common causes are an invalid API key for the selected **Mode**, or an address PostGrid cannot use. Fix the cause and mail the document again.

### Not allowed to print draft documents

Frappe refuses to print a draft of a submittable DocType unless **Allow Print for Draft** is ticked in Print Settings. A Mailed Letter notification on New or Save meets this. Tick the setting, or move the notification to Submit. "Not allowed to print cancelled documents" is the same for **Allow Print for Cancelled** and the Cancel event.

### Password not found for PostGrid Settings

Saving PostGrid Settings, or mailing a letter, needs the API key of the selected **Mode**. The message names the missing field: `pm_key_test` is the **Test API Key** and `pm_key_live` the **Live API Key**. Enter the key, or switch **Mode** to the one whose key you have, and save.

### PostGrid Settings will not save

The first save in a mode registers the webhook with PostGrid, so the save fails with PostGrid's error when the key is wrong or PostGrid cannot be reached. Check the key in the [PostGrid dashboard](https://dashboard.postgrid.com/dashboard/settings) and that it matches the **Mode**.

### Failed to mail notification

This Error Log comes from a Mailed Letter notification:

- With the message "Could not find address for" and a document name, the notification has no **From Address**, or the document's **To Address Field** is empty. Set the sender in the notification, or fill in the document's address.
- With a traceback, something failed before the letter reached PostGrid: an Address without a **Country**, a print Frappe refused, or a missing API key. The traceback names it.

### The delivery status stays at Scheduled

PostGrid's updates are not reaching the site. Check that:

- the site is reachable from the internet at the address it registered, which uses `host_name` from `site_config.json` when that is set;
- PostGrid Settings has a webhook for the mode in use. If the site's address changed, or the webhook was deleted in PostGrid's dashboard, use **Reset Webhooks** (see [Configuration](configuration.md#reset-webhooks)).

A call the app cannot verify against the webhook secret is refused, so a webhook created by hand in PostGrid's dashboard, without that secret, does not work. Use **Reset Webhooks** instead.

### Reset Webhooks messages

| Message                                                                                                                                                            | Cause and fix                                                                                                                                                      |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| "Reset Webhooks needs an API key. Enter the Test or the Live API key, save, and try again." (**No API Key**)                                                       | Neither key is entered. Enter one, save and run it again.                                                                                                          |
| "Skipped Test: it has no API key." or "Skipped Live: it has no API key."                                                                                           | That mode has no key, so it was left alone. Nothing to fix unless you use that mode.                                                                               |
| "Could not reset the Live webhook: ... Its webhook was not changed."                                                                                               | PostGrid refused the key or the delete of the old webhook. The old webhook and its id stay. Fix the key and run it again.                                          |
| "Could not register a Live webhook: ... Its webhook was not changed."                                                                                              | There was no webhook for that mode and PostGrid could not register one. Fix the cause PostGrid gives and run it again.                                             |
| "Could not register a new Live webhook: ... The old one was deleted, so Live has no webhook until Reset Webhooks succeeds or the settings are saved in Live mode." | The old webhook is gone and no new one was made, so that mode gets no updates. Fix the cause, then run **Reset Webhooks** again or save the settings in that mode. |

The same messages name Test for the test mode. When any mode failed, the report is titled **Webhooks Not Reset** and the failure is also in the Error Log, titled "PostGrid: could not reset the ... webhook" or "PostGrid: could not register a ... webhook".

### An address changed by itself

**Update Addresses** is ticked in PostGrid Settings, and PostGrid verified the address of a letter. See [Verified addresses](track-letters.md#verified-addresses).

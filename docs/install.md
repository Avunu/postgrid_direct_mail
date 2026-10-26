---
title: Install and upgrade
description: Install Direct Mail via PostGrid on a Frappe v16 bench, see what the install adds to your site, and upgrade or uninstall it.
nav_title: Install
order: 1
updated: 2026-10-26
---

Direct Mail via PostGrid installs like any Frappe app. It has no build step of its own beyond Frappe's normal asset build, and nothing to set up before PostGrid Settings.

## Before you start

- A bench with Frappe v16, and a site to install on. ERPNext is optional.
- A System Manager login on the site. Only the System Manager role can open PostGrid Settings.
- A PostGrid Print & Mail account. You enter its API keys after the install: see [Configuration](configuration.md).

## Install

Run these from the bench directory. `<SITE_NAME>` is your site, for example `erp.example.com`.

```bash
bench get-app https://github.com/Avunu/postgrid_direct_mail --branch version-16
bench --site <SITE_NAME> install-app postgrid_direct_mail
```

> [!NOTE]
> The `version-16` branch is created by the first release. Until then, install from `develop`: `bench get-app https://github.com/Avunu/postgrid_direct_mail --branch develop`.

## What installing sets up

Installing the app does these things on its own:

| What                                  | Change                                                                                                                                                                                                 |
| ------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **PostGrid Settings**                 | A new single DocType in the PostGrid Integration module. It starts in Test mode with no API key.                                                                                                       |
| **Integrations** workspace            | A **Direct Mail** card with a link to PostGrid Settings.                                                                                                                                               |
| Notification, **Channel**             | **Mailed Letter** is added to the options, with a Property Setter.                                                                                                                                     |
| Communication, **Communication Type** | **Mailed Letter** is added to the options, with a Property Setter. Each mailed letter is stored as a Communication of this type.                                                                       |
| Notification fields                   | A **Direct Mail Settings** section with the sender, recipient and mailing options, and a Property Setter that makes **Recipients** mandatory only for the Email, SMS and System Notification channels. |
| Address field                         | A hidden **Postgrid ID** field (`postgrid_id`) that links an address to PostGrid's copy of it.                                                                                                         |

The custom fields are marked system-generated, so **Reset to Standard** in Customize Form keeps them. [Configuration](configuration.md#what-the-app-adds-to-your-site) lists every field.

Nothing is sent to PostGrid at install. The webhook for delivery updates is registered the first time you save PostGrid Settings in each mode.

## Upgrade

Update the app the way you update any Frappe app: `bench update`, or pull the new code in `apps/postgrid_direct_mail` and then run this from the bench directory.

```bash
bench --site <SITE_NAME> migrate
```

`bench migrate` re-syncs the custom fields and the Property Setter on Notification, adds **Mailed Letter** to the two option lists again if it is missing, and marks the app's custom fields system-generated. The app ships no data patches.

The app was called `postgrid_integration` before. Its old whitelisted method paths, including the webhook address PostGrid was given before the rename, still answer and forward to the new ones, so webhooks registered under the old name keep delivering updates.

## Uninstall

```bash
bench --site <SITE_NAME> uninstall-app postgrid_direct_mail
```

Uninstalling removes:

- PostGrid Settings, with its API keys and the webhook ids it recorded.
- The **Direct Mail** card and its link in the Integrations workspace. The card stays if you added other links under it.
- **Mailed Letter** from the Notification **Channel** and the Communication **Communication Type** options. The Property Setter goes when the options are back to the standard ones.
- The custom fields on Address and Notification, and the Property Setter on Notification **Recipients**.

These stay behind:

- The webhooks registered at PostGrid. Delete them in the PostGrid dashboard, or PostGrid keeps calling an address that no longer answers.
- The mailed letters: their Communications, their Integration Requests and any saved PDF files.
- The database columns of the removed custom fields and their values, as Frappe leaves the columns of deleted custom fields.
- Notifications that used the **Mailed Letter** channel. They keep the channel value but no longer mail anything.

## Next

- [Configuration](configuration.md) explains each setting and lists everything the app adds.
- [Mail a document](mail-a-document.md) sends the first letter.

---
title: Develop Direct Mail via PostGrid
description: The dev shell, the commands and tests, where the code lives, and what CI, the release flow and the docs check run for this app.
nav_title: Development
order: 9
updated: 2026-10-26
---

This page is for people changing the app. [How it works](architecture.md) explains the design behind the code. The managed files, checks and release flow shared by all Avunu apps are described once in the [app standards documentation](https://docs.avunu.net/frappe-nix/docs/app-standards/).

## Set up

`nix develop --no-pure-eval` (or `direnv allow`) gives you Python, Node, yarn, the git hooks and a dev bench in `.frappe-nix/bench` with the app installed.

```bash
nix develop --no-pure-eval
prek install
frappe-test
```

## Commands

| Command                              | What it does                                                                                               |
| ------------------------------------ | ---------------------------------------------------------------------------------------------------------- |
| `yarn check`                         | Checks the formatting, lints and type-checks the desk scripts                                              |
| `yarn lint` / `yarn format`          | Runs oxlint and oxfmt                                                                                      |
| `yarn typecheck`                     | Type-checks the desk scripts (`public/js`, JSDoc-typed) and the screenshot spec                            |
| `yarn lint:py` / `yarn typecheck:py` | Runs ruff and ty on the Python code                                                                        |
| `frappe-test --reuse-site`           | Runs the Python tests on the dev site, with coverage                                                       |
| `frappe-shots`                       | Rebuilds the demo site (`postgrid_direct_mail/demo.py`) and retakes the screenshots in `docs/screenshots/` |

## Tests

The Python tests are Frappe `IntegrationTestCase`s:

| File                                                                                            | Covers                                                                                                                                                  |
| ----------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `postgrid_direct_mail/postgrid_integration/doctype/postgrid_settings/test_postgrid_settings.py` | The settings (mode, key, secret and webhook on save), every **Reset Webhooks** outcome, the webhook endpoint, `get_postgrid_defaults` and `mail_letter` |
| `postgrid_direct_mail/tests/test_notification.py`                                               | The Notification extension: the events that mail, other channels left to Frappe, the options it sends and the failures it logs                          |
| `postgrid_direct_mail/tests/test_install.py`                                                    | The workspace card, the channel options on install, migrate and uninstall                                                                               |
| `postgrid_direct_mail/tests/test_custom_fields.py`                                              | The shipped custom fields, their system-generated flag and their removal on uninstall                                                                   |
| `postgrid_direct_mail/tests/test_demo.py`                                                       | The demo data                                                                                                                                           |

The tests never reach PostGrid: they replace the client's `Letter.create` and `Webhook.create`, and Frappe's PDF rendering, with stand-ins (`postgrid_direct_mail/tests/utils.py`). The demo does the same. `frappe-test` fails below 80% coverage (`fail_under` in `pyproject.toml`).

There are no JavaScript unit tests. `yarn check` is the front-end gate.

## Where things live

| Path                                                                                       | Holds                                                                                                   |
| ------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------- |
| `postgrid_direct_mail/hooks.py`                                                            | The hooks, and the rename shim                                                                          |
| `postgrid_direct_mail/config/__init__.py`                                                  | `MAIL_TYPES`, the channels the app adds (Mailed Letter only, for now)                                   |
| `postgrid_direct_mail/postgrid_integration/doctype/postgrid_settings/__init__.py`          | The whitelisted methods `mail_letter`, `get_postgrid_defaults` and `webhooks`, and the timeline content |
| `postgrid_direct_mail/postgrid_integration/doctype/postgrid_settings/postgrid_settings.py` | The settings DocType, the PostGrid client and webhook registration, and **Reset Webhooks**              |
| `postgrid_direct_mail/postgrid_integration/hooks/notification.py`                          | The Notification extension that mails on the **Mailed Letter** channel                                  |
| `postgrid_direct_mail/postgrid_integration/custom/`                                        | The custom fields on Address and Notification, and the Property Setter on Notification                  |
| `postgrid_direct_mail/install.py`, `postgrid_direct_mail/setup/custom_fields.py`           | The install, migrate and uninstall hooks                                                                |
| `postgrid_direct_mail/public/js/print_view.js`                                             | The print view's **Mail** button and dialog                                                             |
| `postgrid_direct_mail/public/js/notification.js`                                           | The Notification form for the mail channels                                                             |
| `postgrid_direct_mail/public/js/templates/direct_mail_timeline.html`                       | The timeline card, bundled by `main.bundle.js`                                                          |
| `postgrid_direct_mail/demo.py`                                                             | The demo data for `frappe-demo` and the screenshots                                                     |
| `marketplace/`                                                                             | The marketplace listing and the screenshot spec                                                         |

## The rename

The app was called `postgrid_integration` before. `hooks.py` keeps the old dotted paths of the whitelisted methods working (the block between the `frappe-nix:rename-shim` markers) for callers that still use them, such as webhooks registered before the rename. Keep it for at least one minor release after the rename.

## Continuous integration

`.github/workflows/ci.yml` is managed by frappe-nix and runs the shared quality gates on every pull request, every push to `develop` and once a day. Each gate is a check named `ci / <gate>`: `lint` (the pre-commit hooks, run with prek), `typecheck`, `test` (the Python tests with coverage) and `marketplace` (the checks on `marketplace/listing.toml`, `hooks.py` and the logo). `pr-policy.yml` checks a pull request's base, title and commits. `nightly.yml` runs the integration suites each night and is never a required check.

Pull requests go to `develop`. Commit messages and pull request titles follow Conventional Commits.

The pre-commit hooks in `.pre-commit-config.yaml` are managed by frappe-nix. Run them as CI does with `prek run --all-files` in the dev shell. `validate_copyright` skips `docs/`, so pages here carry no copyright comment above their front matter.

## Releases

`release.yml` runs release-please on `develop`. Merging its release pull request tags the version, writes `CHANGELOG.md` and moves the `version-16` branch forward, which is the branch the install instructions use. Release notes are on the [releases page](https://github.com/Avunu/postgrid_direct_mail/releases).

## Working on these docs

The documentation is Markdown in `docs/`. It is published with the other Avunu projects at [docs.avunu.net/postgrid-direct-mail/docs/](https://docs.avunu.net/postgrid-direct-mail/docs/) by [Avunu/site-docs](https://github.com/Avunu/site-docs), and the `Docs` workflow (`.github/workflows/docs.yml`) checks it on every pull request that changes `docs/`. Follow the site's [writing rules](https://github.com/Avunu/site-docs/blob/main/docs/guide/writing-docs.md): front matter, GitHub alerts and relative links.

To check a change as the pull request check does, clone Avunu/site-docs beside this repository and, from its folder, run:

```bash
npm ci --ignore-scripts && npm run build
CI=true node build/cli.js build --only postgrid-direct-mail --source postgrid-direct-mail=../postgrid_direct_mail
```

Do not edit `docs/screenshots/`: `frappe-shots` writes it.

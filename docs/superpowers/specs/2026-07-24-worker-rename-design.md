# Worker rename to `mail` design

Date: 2026-07-24

## Goal

Shorten the application and API URL from `https://cloudflare_temp_email.cooe.workers.dev` to `https://mail.cooe.workers.dev` while continuing to receive mail for `gmail.dpdns.org`.

## Non-goals

- Do not add `mail.gmail.dpdns.org` or another custom web domain.
- Do not change the receiving domain `gmail.dpdns.org`.
- Do not change D1 data, user accounts, JWT signing material, administrator credentials, or the authentication model.
- Do not clone the repository locally or modify any other repository.

## Current state

- Worker name: `cloudflare_temp_email`.
- Public URL: `https://cloudflare_temp_email.cooe.workers.dev`.
- Receiving domain: `gmail.dpdns.org`.
- Email Routing catch-all target: `cloudflare_temp_email`.
- D1 binding: `temp_mail`.
- Worker Secrets: `JWT_SECRET` and `ADMIN_PASSWORDS`.
- Deployment source: GitHub Actions using the `BACKEND_TOML` repository secret.

## Selected approach

Rename the existing Cloudflare Worker rather than deploy a second Worker. This minimizes secret and binding migration. Cloudflare documents that renaming a Worker removes Email Routing rule bindings, so the catch-all rule must be updated immediately after the rename.

## Migration sequence

1. Preflight-check the current Worker, D1 binding, encrypted Secrets, active Email Routing catch-all, and latest successful GitHub Actions deployment.
2. Update the GitHub `BACKEND_TOML` secret from `name = "cloudflare_temp_email"` to `name = "mail"`, but do not dispatch the workflow yet.
3. Rename the existing Worker to `mail` in the Cloudflare dashboard.
4. Confirm that the renamed Worker still has the `temp_mail` D1 binding and both encrypted Secrets.
5. Update the `gmail.dpdns.org` catch-all rule to send mail to `mail` and confirm that the rule is active.
6. Dispatch the `Deploy Backend` GitHub Actions workflow. The existing `keep_vars = true` setting must remain so encrypted Worker Secrets are preserved.
7. Verify the new URL, authentication, API behavior, and real mail delivery.

## Verification

- `https://mail.cooe.workers.dev/health_check` returns `OK`.
- `https://mail.cooe.workers.dev` loads the bundled frontend.
- Public user registration remains disabled.
- Anonymous address creation remains disabled.
- `owner@gmail.dpdns.org` can log in and create an address.
- Existing address credentials remain valid because `JWT_SECRET` is unchanged.
- A real test message to an address at `gmail.dpdns.org` is received and stored in D1.
- The old `cloudflare_temp_email.cooe.workers.dev` URL is no longer the supported entry point.

## Failure handling and rollback

- Expect a short mail-routing interruption between the Worker rename and catch-all update; perform those steps consecutively.
- If the dashboard indicates that renaming will not preserve bindings or Secrets, stop before confirming the rename.
- If D1 or Secrets are missing after the rename, do not deploy. Rename the Worker back, restore `BACKEND_TOML`, and rebind the catch-all to `cloudflare_temp_email`.
- If the new deployment fails, restore `name = "cloudflare_temp_email"`, rename the Worker back, rebind Email Routing, and run the last known-good workflow configuration.

## Operating model

- Source deployment continues through GitHub Actions.
- Cloudflare Worker rename and Email Routing reassignment are control-plane operations performed in the Cloudflare dashboard.
- No local checkout is created.

# Private user access design

Date: 2026-07-24

## Goal

Make mailbox creation and management private while retaining address-scoped credential access.

## Access model

- Set `DISABLE_ANONYMOUS_USER_CREATE_EMAIL = true`; the Worker returns HTTP 403 when `/api/new_address` has no valid user JWT.
- Keep `ENABLE_USER_CREATE_EMAIL = true` for authenticated users.
- Keep public registration disabled in `user_settings`.
- Create users only from the authenticated admin interface.
- Keep address JWT login available; it grants access only to the corresponding mailbox.
- Admin and normal user authentication remain separate.

## Security

- Use a generated, unique password because upstream stores user passwords directly in D1.
- Keep `JWT_SECRET` and `ADMIN_PASSWORDS` as encrypted Worker secrets.
- Do not put credentials in repository files or Actions logs.

## Verification

1. Anonymous mailbox creation receives HTTP 403.
2. Public registration is disabled.
3. The dedicated user can log in and create a mailbox.
4. Its address JWT opens only the created mailbox.

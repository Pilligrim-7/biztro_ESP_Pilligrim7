# Auth and Billing Deployment Notes

This guide documents the production-sensitive Better Auth and Stripe wiring in
Biztro. Use it when changing auth origins, signup access, organization
invitations, subscription plans, or Stripe webhook configuration.

## Code map

- `src/lib/auth.ts`
  - Creates the Better Auth server instance.
  - Configures trusted origins, Google OAuth, organization fields, invitation
    email sending, and the Better Auth Stripe plugin.
  - Enforces invite-only signup by checking the `Waitlist` table before user
    creation.
- `src/lib/auth-client.ts`
  - Enables organization and Stripe subscription plugins on the client auth
    helper.
- `src/lib/utils.ts`
  - Resolves the canonical app URL with `getBaseUrl()`.
  - Builds public menu links with `getPublishedMenuUrl()`.
  - Sends organization invitation emails through Resend.
- `src/server/actions/subscriptions/*`
  - Reads active subscriptions and creates Stripe billing portal sessions.
- `src/env.mjs`
  - Validates required runtime variables.

## Environment variables

The env schema is the source of truth for validated variables. These variables
are operationally important for auth and billing:

| Variable                                | Used by                                         | Notes                                                                                                     |
| --------------------------------------- | ----------------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| `BETTER_AUTH_URL`                       | `auth.baseURL`, trusted origins, `getBaseUrl()` | Set explicitly in production to the canonical public origin, for example `https://biztro.co`.             |
| `BETTER_AUTH_SECRET`                    | Better Auth runtime                             | Required in production by `src/env.mjs`.                                                                  |
| `AUTH_GOOGLE_ID` / `AUTH_GOOGLE_SECRET` | Google OAuth provider                           | Must match the OAuth client configured for the deployed origins.                                          |
| `RESEND_API_KEY`                        | Organization invitation emails                  | Used by `sendOrganizationInvitation()`.                                                                   |
| `STRIPE_SECRET_KEY`                     | Stripe server SDK                               | Used to initialize the Stripe client for the Better Auth Stripe plugin.                                   |
| `STRIPE_WEBHOOK_SECRET`                 | Better Auth Stripe plugin                       | Must match the signing secret for `/api/auth/stripe/webhook`.                                             |
| `NEXT_PUBLIC_STRIPE_PRICE_PRO_MONTHLY`  | Better Auth Stripe plugin, plan UI              | Pro monthly price ID.                                                                                     |
| `NEXT_PUBLIC_STRIPE_PRICE_PRO_YEARLY`   | Better Auth Stripe plugin, plan UI              | Pro annual price ID.                                                                                      |
| `NEXT_PUBLIC_STRIPE_PRICE_BASIC`        | Better Auth Stripe plugin                       | Basic plan price ID read directly by `src/lib/auth.ts`; this is not currently validated in `src/env.mjs`. |

## Production URL consistency

Set `BETTER_AUTH_URL` explicitly for production and preview environments that
need stable auth, invite, or billing links.

`getBaseUrl()` resolves in this order:

1. `BETTER_AUTH_URL`
2. `https://biztro.co` when `NEXT_PUBLIC_VERCEL_ENV=production`
3. `https://${NEXT_PUBLIC_VERCEL_URL}` when `NEXT_PUBLIC_VERCEL_ENV=preview`
   and `NEXT_PUBLIC_VERCEL_URL` is set
4. `http://localhost:3000`

The resolved base URL is reused for:

- organization invitation links (`/invite/:id`)
- billing portal return URLs (`/dashboard/settings/billing`)
- published menu URLs through `getPublishedMenuUrl()`

`src/lib/auth.ts` also includes `BETTER_AUTH_URL`, `https://biztro.co`,
`https://preview.biztro.co`, matching Vercel preview URLs, and
`http://localhost:3000` in Better Auth trusted origins.

## Signup access and waitlist flow

Signup is invite-only at account creation time:

1. Marketing waitlist forms call `joinWaitlist`.
2. `joinWaitlist` inserts the email into the `Waitlist` table.
3. `Waitlist.enabled` defaults to `false`.
4. Better Auth `databaseHooks.user.create.before` calls
   `isWaitlistEnabled(email)`.
5. If the email is missing from the waitlist or not enabled, user creation is
   blocked with `AccessDenied`.
6. Better Auth errors containing `unable_to_create_user` are redirected to
   `/auth-error?type=access_denied`.

To grant access, update the target waitlist row so `enabled=true` before the
user signs in.

## Organization invitations

Organization invitations use Better Auth's organization plugin:

- `inviteMember` calls `auth.api.createInvitation()`.
- The plugin calls `sendInvitationEmail()` from `src/lib/auth.ts`.
- Invitation links use `${getBaseUrl()}/invite/${id}`.
- Email delivery is handled by Resend in `sendOrganizationInvitation()`.

If invite links point at the wrong host, check `BETTER_AUTH_URL` first.

## Stripe subscription flow

Stripe billing is configured through `@better-auth/stripe` in `src/lib/auth.ts`.

- A Stripe customer is created when a user signs up.
- Plans configured for the plugin:
  - `BASIC` with limits from `src/app/config.ts`
  - `PRO` with monthly and annual price IDs and a 30-day free trial
- Subscription ownership is scoped by organization ID (`referenceId`).
- Billing portal sessions are created by `createStripePortal(referenceId)`.
- Only organization owners or admins can manage a subscription reference.
- `isProMember()` treats active or trialing `PRO` subscriptions as Pro access.
  Organizations with `SPONSORED` status also receive Pro access.

### Local webhook testing

Run the app locally, then forward Stripe webhooks to Better Auth:

```sh
bun run stripe:listen
```

The script forwards events to:

```txt
localhost:3000/api/auth/stripe/webhook
```

Copy the signing secret printed by the Stripe CLI into
`STRIPE_WEBHOOK_SECRET` for the local process.

## Stripe SDK version policy

`src/lib/auth.ts` initializes Stripe with:

<!-- prettier-ignore -->
```ts
apiVersion: Stripe.API_VERSION
```

Keep this pattern unless there is a deliberate Stripe migration that requires a
pinned dated API version. Using `Stripe.API_VERSION` keeps the configured API
version aligned with the installed `stripe` SDK.

## Troubleshooting

### OAuth or auth requests fail in production

- Verify `BETTER_AUTH_URL` matches the public app origin.
- Verify the Google OAuth client allows the deployed origin and callback URLs.
- Check that the request origin is covered by `trustedOrigins` in
  `src/lib/auth.ts`.

### Invitation email links point to localhost or a preview host

- Set `BETTER_AUTH_URL` in the deployment environment.
- Re-send the invitation after the variable is corrected.

### Billing portal returns to the wrong host

- `createStripePortal()` builds the return URL from `getBaseUrl()`.
- Fix `BETTER_AUTH_URL` or the Vercel URL environment values for that
  deployment.

### Stripe webhooks are not processed locally

- Confirm the local app is running on port `3000`.
- Re-run `bun run stripe:listen`.
- Copy the latest Stripe CLI signing secret into `STRIPE_WEBHOOK_SECRET`.
- Confirm the forwarded path is `/api/auth/stripe/webhook`.

### A waitlisted user cannot sign in

- Confirm the email in `Waitlist.email` matches the signup email.
- Confirm `Waitlist.enabled=true`.
- If the user already saw the denied page, have them retry after enabling the
  row.

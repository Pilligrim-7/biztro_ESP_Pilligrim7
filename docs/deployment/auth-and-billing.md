# Auth and Billing Runbook

This runbook covers the production-facing auth, waitlist, organization invite,
and Stripe billing paths. Keep it aligned with `src/lib/auth.ts`,
`src/env.mjs`, and the billing pages under
`src/app/dashboard/settings/billing/`.

## Architecture

- Better Auth is configured in `src/lib/auth.ts` and exposed through the
  catch-all route in `src/app/api/auth/[...all]/route.ts`.
- The Prisma adapter uses the SQLite provider against the LibSQL/Turso adapter
  configured elsewhere in the app.
- Auth is Google-only for normal sign-in; `emailAndPassword.enabled` is `false`.
- The Better Auth organization plugin owns organizations, members, invitations,
  and active organization session state.
- The Better Auth Stripe plugin owns customer creation, subscription checkout,
  subscription reads, billing portal URLs, and Stripe webhook handling.
- The dashboard billing page is available at
  `/dashboard/settings/billing` and only renders subscription controls for
  organization owners while the `FLAGS_ENABLE_SUBSCRIPTIONS=1` flag is enabled.

## Source-of-truth files

| Area | Files |
| --- | --- |
| Auth server config | `src/lib/auth.ts` |
| Auth client plugins | `src/lib/auth-client.ts` |
| Env validation | `src/env.mjs` |
| Base URL helpers | `src/lib/utils.ts` |
| Auth API route | `src/app/api/auth/[...all]/route.ts` |
| Billing actions | `src/server/actions/subscriptions/*` |
| Pro entitlement checks | `src/server/actions/user/queries.ts` |
| Billing UI | `src/app/dashboard/settings/billing/*` |
| Billing/domain types | `src/lib/types/billing.ts`, `prisma/models/auth.prisma` |
| Waitlist writes | `src/server/actions/organization/mutations.ts` |

## Environment configuration

`src/env.mjs` is the source of truth for validated variables. Do not document or
commit real secret values.

Required auth and billing variables:

```env
BETTER_AUTH_SECRET=...
BETTER_AUTH_URL=https://biztro.co
AUTH_GOOGLE_ID=...
AUTH_GOOGLE_SECRET=...
STRIPE_SECRET_KEY=...
STRIPE_WEBHOOK_SECRET=...
NEXT_PUBLIC_STRIPE_PRICE_PRO_MONTHLY=...
NEXT_PUBLIC_STRIPE_PRICE_PRO_YEARLY=...
```

Operational constraints:

- Set `BETTER_AUTH_URL` explicitly in production. `src/lib/auth.ts` uses it for
  Better Auth `baseURL`, and `src/lib/utils.ts` prefers it when generating
  invite links, Stripe portal return URLs, and other absolute app URLs.
- Keep `BETTER_AUTH_URL` in the `trustedOrigins` list by leaving the existing
  `process.env.BETTER_AUTH_URL` entry in `src/lib/auth.ts`.
- `trustedOrigins` also includes `https://biztro.co`,
  `https://preview.biztro.co`, safe Vercel preview URLs matching
  `*.vercel.app`, and `http://localhost:3000`.
- Use `FLAGS_ENABLE_SUBSCRIPTIONS=1` when the billing page should expose
  checkout and portal controls.
- The public billing tier metadata reads the PRO price IDs from
  `NEXT_PUBLIC_STRIPE_PRICE_PRO_MONTHLY` and
  `NEXT_PUBLIC_STRIPE_PRICE_PRO_YEARLY`; keep those IDs aligned with the Stripe
  products configured for the Better Auth Stripe plugin.
- `src/lib/auth.ts` currently also references
  `NEXT_PUBLIC_STRIPE_PRICE_BASIC` for the Stripe plugin's `BASIC` plan, while
  `src/env.mjs` validates only the PRO price IDs. Verify that value explicitly
  when changing Stripe plan setup or env validation.

## Local development workflow

1. Install dependencies and generate Prisma client:

   ```sh
   bun install
   bun run predev
   ```

2. Start the local LibSQL database:

   ```sh
   bun run db:dev
   ```

3. Start the Next.js app:

   ```sh
   bun run dev
   ```

4. In a separate shell, forward Stripe webhooks to the Better Auth Stripe
   webhook endpoint:

   ```sh
   bun run stripe:listen
   ```

   This runs:

   ```sh
   stripe listen --forward-to localhost:3000/api/auth/stripe/webhook
   ```

5. Copy the webhook signing secret printed by the Stripe CLI into
   `STRIPE_WEBHOOK_SECRET` for the local session.

## Waitlist and sign-in behavior

Account creation is invite/waitlist gated:

- Landing-page waitlist submissions call `joinWaitlist` in
  `src/server/actions/organization/mutations.ts`.
- `joinWaitlist` creates a `Waitlist` row with `enabled: false`.
- During Better Auth user creation, `databaseHooks.user.create.before` calls
  `isWaitlistEnabled(email)` in `src/server/actions/user/queries.ts`.
- If the email has no enabled waitlist row, the hook throws `AccessDenied`.
- The auth `before` middleware maps Better Auth
  `unable_to_create_user` errors to `/auth-error?type=access_denied`.
- `src/app/(auth)/auth-error/page.tsx` renders the beta/waitlist message for
  that `access_denied` case.

To allow a beta user to create an account, update the corresponding `Waitlist`
record so `enabled` is `true`.

## Organization invitations

Organization invitations are sent from the Better Auth organization plugin
inside `src/lib/auth.ts`:

- `sendInvitationEmail` builds `/invite/:id` links with `getBaseUrl()`.
- `getBaseUrl()` prefers `BETTER_AUTH_URL`, then production/preview Vercel
  environment variables, then `http://localhost:3000`.
- `sendOrganizationInvitation()` in `src/lib/utils.ts` sends the email with
  Resend and reports send failures to Sentry.

If invite links point at localhost or the wrong preview deployment, verify
`BETTER_AUTH_URL`, `NEXT_PUBLIC_VERCEL_ENV`, and `NEXT_PUBLIC_VERCEL_URL`.

## Stripe billing flow

### Checkout

The Basic plan view starts checkout from the browser with:

```ts
authClient.subscription.upgrade({
  plan: Plan.PRO,
  annual: billingInterval === "yearly",
  referenceId: activeOrganization?.id,
  successUrl: "/dashboard/settings/billing",
  cancelUrl: "/dashboard/settings/billing"
})
```

Constraints:

- `referenceId` must be the organization ID, not the user ID.
- The server-side Stripe plugin allows subscription management only when
  `authorizeReference` finds the current user as an organization `owner` or
  `admin`.
- The billing page itself only displays controls to the organization `owner`.

### Customer portal

`CustomerPortalButton` calls `createStripePortal(referenceId)`, which delegates
to `auth.api.createBillingPortal()` and uses
`${getBaseUrl()}/dashboard/settings/billing` as the return URL. If the portal
returns users to the wrong host, fix `BETTER_AUTH_URL` first.

### Subscription reads

Subscription state is read through Better Auth:

- `getCurrentSubscription(organizationId)` calls
  `auth.api.listActiveSubscriptions({ referenceId: organizationId })` and
  returns the first `active` or `trialing` subscription.
- `isProMember()` performs the same active/trialing lookup for the current
  organization.
- If the active subscription plan differs from the stored organization plan,
  `isProMember()` updates the organization `plan` and `status` through Better
  Auth unless the organization is `SPONSORED`.

## Pro entitlement model

Use `isProMember()` as the server-side source of truth for Pro-gated mutations.
Client-side `organization.plan` checks are only UI hints.

Current server-side Pro gates include:

- menu creation and duplication limits in `src/server/actions/menu/mutations.ts`
- item creation and bulk item creation limits in
  `src/server/actions/item/mutations.ts`
- AI menu import in `src/server/actions/menu-import/mutations.ts`
- translation generation in `src/server/actions/item/translations.ts`
- media upload limits and Pro-only media scopes in `src/app/api/file/route.ts`

Basic plan limits come from `src/app/config.ts`:

- `menuLimit: 1`
- `itemLimit: 10`
- `mediaLimit: 20`

Sponsored organizations bypass Stripe subscription requirements by setting
`Organization.status` to `SPONSORED`.

## Common failure modes

### OAuth, invite, or billing redirects use the wrong host

Cause:

- `BETTER_AUTH_URL` is unset or points at the wrong deployment.

Fix:

- Set `BETTER_AUTH_URL` to the canonical app origin for the environment.
- Confirm `src/lib/auth.ts` still passes it to Better Auth `baseURL`.
- Confirm `getBaseUrl()` in `src/lib/utils.ts` returns the expected host.

### Checkout or portal fails for a non-owner

Cause:

- The billing page requires owner role for UI access.
- The Stripe plugin authorizes organization subscription changes only for owner
  or admin members.

Fix:

- Use an organization owner account for the dashboard billing page.
- If using a custom call site, pass an organization `referenceId` that the user
  belongs to as owner or admin.

### Stripe subscription status does not update locally

Cause:

- Webhooks are not reaching `/api/auth/stripe/webhook`.
- `STRIPE_WEBHOOK_SECRET` does not match the current Stripe CLI listener or
  dashboard webhook endpoint.

Fix:

- Run `bun run stripe:listen` while testing locally.
- Refresh `STRIPE_WEBHOOK_SECRET` from the current Stripe CLI output.
- In production, verify the Stripe dashboard endpoint forwards to
  `https://biztro.co/api/auth/stripe/webhook`.

### Beta user cannot create an account

Cause:

- Better Auth user creation is denied unless the email has an enabled waitlist
  row.

Fix:

- Confirm the email is normalized as lowercase.
- Set the matching `Waitlist.enabled` record to `true`.

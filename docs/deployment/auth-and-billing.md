# Auth and Billing Operations

Biztro uses Better Auth for authentication, organizations, and Stripe-backed subscriptions. This runbook explains how the pieces fit together, which environment variables must be present, and what to check when sign-in, organization access, or billing changes fail.

## Architecture

- `src/lib/auth.ts`
  - Creates the Better Auth server instance with the Prisma adapter.
  - Enables Google sign-in, organizations, Next.js cookies, and the Stripe plugin.
  - Blocks account creation unless the user's email is enabled in the `Waitlist` table.
  - Adds `activeOrganizationId` to new sessions from the first membership found for the user.
- `src/lib/auth-client.ts`
  - Creates the browser auth client with organization and Stripe subscription plugins.
  - Exposes `authClient.subscription.upgrade()` for Checkout and organization helpers for active-org switching.
- `src/app/api/auth/[...all]/route.ts`
  - Mounts Better Auth's `GET` and `POST` handlers for auth, organization, and Stripe plugin endpoints.
- `src/proxy.ts`
  - Protects `/dashboard` by checking the Better Auth session cookie.
  - Leaves `/api/*`, monitoring, ingest, static assets, and image routes outside the proxy matcher.
- `src/server/actions/subscriptions/*`
  - Reads active subscriptions through `auth.api.listActiveSubscriptions()`.
  - Creates Stripe Billing Portal sessions with `auth.api.createBillingPortal()`.
- `src/server/actions/user/queries.ts`
  - `isProMember()` is the server-side entitlement source for Pro features.
  - Synchronizes organization `plan` and `status` from active Stripe subscriptions unless the organization is `SPONSORED`.
- `src/server/actions/menu/*`, `src/server/actions/item/*`, `src/app/api/file/route.ts`
  - Enforce Basic-plan limits and Pro-only features on the server.

## Required Configuration

Environment validation lives in `src/env.mjs`. Keep deployment secrets in the hosting provider, not in source control.

### Better Auth

- `BETTER_AUTH_SECRET`
  - Required in production.
  - Generate with a strong random value, for example `openssl rand -base64 32`.
- `BETTER_AUTH_URL`
  - Canonical application URL.
  - Used by Better Auth and by `getBaseUrl()` for invite and billing return URLs.
- `AUTH_GOOGLE_ID`
- `AUTH_GOOGLE_SECRET`
  - Google OAuth credentials used by `src/lib/auth.ts`.

`trustedOrigins` in `src/lib/auth.ts` currently allows:

- `BETTER_AUTH_URL`
- `https://biztro.co`
- `https://preview.biztro.co`
- valid `NEXT_PUBLIC_VERCEL_URL` preview hosts
- `http://localhost:3000`

Add a new origin there before routing auth traffic through another hostname.

### Stripe

- `STRIPE_SECRET_KEY`
- `STRIPE_WEBHOOK_SECRET`
- `NEXT_PUBLIC_STRIPE_PRICE_PRO_MONTHLY`
- `NEXT_PUBLIC_STRIPE_PRICE_PRO_YEARLY`

The Stripe client in `src/lib/auth.ts` uses `Stripe.API_VERSION`. This intentionally follows the installed Stripe SDK API version instead of pinning a dated string in application code.

The Better Auth Stripe plan config also references `NEXT_PUBLIC_STRIPE_PRICE_BASIC` for the Basic plan. Basic is currently free in product UI, but if the Stripe plugin path for Basic is enabled, make sure this value exists in the runtime environment and is added to validation if it should be required.

### Feature Flag

The billing page is gated by `subscriptionsEnabled()` in `src/flags.ts`.

Set:

```sh
FLAGS_ENABLE_SUBSCRIPTIONS=1
```

Without this flag, `/dashboard/settings/billing` renders an owner-only warning instead of the plan management UI.

## Runtime Flows

### Invite-only account creation

1. A user starts Google sign-in.
2. Before Better Auth creates a `User`, the `databaseHooks.user.create.before` hook normalizes the email from the payload or auth context.
3. `isWaitlistEnabled(email)` checks `Waitlist.enabled`.
4. If the email is not enabled, the hook throws `AccessDenied`.
5. The Better Auth `before` hook maps upstream `unable_to_create_user` errors to `/auth-error?type=access_denied`.

Operational notes:

- The waitlist check only runs at user creation time.
- Existing users are not blocked by this hook on later sign-ins.
- Empty or unresolvable emails are treated as not enabled.

### Organization bootstrap and active sessions

New organizations are created through `bootstrapOrg()` in `src/server/actions/organization/mutations.ts`.

That action:

1. Verifies the requested slug through `auth.api.checkOrganizationSlug()`.
2. Creates the organization with `status: "ACTIVE"` and `plan: "BASIC"`.
3. Sets the created organization as active for the current session.
4. Revalidates organization, membership, permissions, subscription, and settings cache tags.

Session creation also calls `getActiveOrganization(userId)` and stores the first membership's organization id as `activeOrganizationId`.

### Checkout

Owners upgrade from `BasicPlanView` in `src/app/dashboard/settings/billing/basic-plan-view.tsx`.

The client calls:

```ts
authClient.subscription.upgrade({
  plan: "PRO",
  annual: billingInterval === "yearly",
  referenceId: activeOrganization?.id,
  successUrl: "/dashboard/settings/billing",
  cancelUrl: "/dashboard/settings/billing"
})
```

Constraints:

- `referenceId` is the organization id.
- The Stripe plugin authorizes subscription management with `authorizeReference`.
- Only organization owners or admins pass `authorizeReference`, but the billing page only shows the UI to owners.

### Customer portal

`createStripePortal(referenceId)` creates a billing portal session through Better Auth and returns a Stripe URL.

The return URL is:

```txt
${getBaseUrl()}/dashboard/settings/billing
```

After creating the portal URL, the action revalidates:

- `organization-${referenceId}-subscription`
- `subscription-current`

### Webhooks

Better Auth handles Stripe plugin routes under the catch-all auth route. The local helper script forwards Stripe events to:

```sh
bun run stripe:listen
```

which runs:

```sh
stripe listen --forward-to localhost:3000/api/auth/stripe/webhook
```

Use the webhook signing secret from that listener as `STRIPE_WEBHOOK_SECRET` in local development.

## Entitlements and Limits

`isProMember()` is the source of truth for server-side Pro access.

It returns `true` when:

- Better Auth reports an active or trialing subscription whose plan is `PRO`, or
- the organization status is `SPONSORED`.

It also updates the organization record to match the active subscription plan/status when the organization is not sponsored.

Current Basic limits come from `src/app/config.ts`:

- `menuLimit: 1`
- `itemLimit: 10`
- `mediaLimit: 20`

Server-side enforcement exists in:

- `createMenu()` and `duplicateMenu()` for menu count limits.
- item creation/import actions for item count limits.
- translation actions for Pro-only translations.
- `src/app/api/file/route.ts` for media limits and Pro-only upload scopes.

Public menus are subscription-gated too. `getActiveMenuByOrganizationSlug()` only returns menus for organizations with one of these statuses:

- `ACTIVE`
- `TRIALING`
- `SPONSORED`

If an organization's status moves to `CANCELED`, `PAST_DUE`, `UNPAID`, `PAUSED`, or another inactive status, public subdomain rendering returns 404.

## Cache Tags to Revalidate

When changing auth, organization, or billing behavior, keep these tags aligned:

- `organization-current`
- `organizations-list`
- `membership-current`
- `membership-current-role`
- `permissions-all`
- `organization-${organizationId}`
- `organization-${organizationId}-members`
- `organization-${organizationId}-subscription`
- `subscription-current`
- `menus-${organizationId}`
- `translations-${organizationId}`
- `subdomain-${slug}`

Watch for both hyphen and colon tag styles in older actions before adding new invalidation. Prefer the hyphenated tags already used by current query functions unless you are updating the older callers at the same time.

## Local Verification

1. Start dependencies and the app:

```sh
bun install
bun run db:dev
bun run dev
```

2. In another terminal, forward Stripe webhooks:

```sh
bun run stripe:listen
```

3. Configure `.env` with the listener's `STRIPE_WEBHOOK_SECRET`, Stripe keys, Better Auth URL/secret, Google OAuth credentials, and the required Turso/R2 variables from `src/env.mjs`.
4. Enable billing UI with `FLAGS_ENABLE_SUBSCRIPTIONS=1`.
5. Enable the test user's email in `Waitlist`.
6. Sign in, create an organization, and open `/dashboard/settings/billing`.
7. Run a test upgrade and confirm the Stripe webhook updates the subscription status.
8. Confirm Pro-only server actions allow the org after the webhook is processed.
9. Confirm the public subdomain still resolves only for active, trialing, or sponsored organizations.

## Troubleshooting

### User is redirected to `access_denied`

Cause:

- The email is absent from `Waitlist`, or `Waitlist.enabled` is `false`.

Fix:

- Enable the normalized lowercase email in the `Waitlist` table.
- Retry with a new user creation flow. Existing users do not rerun the user-create hook.

### Billing page says only owners can access it

Cause:

- The current active organization member is not `owner`, or `FLAGS_ENABLE_SUBSCRIPTIONS` is not `1`.

Fix:

- Switch to the correct organization.
- Verify membership role through Better Auth organization data.
- Set `FLAGS_ENABLE_SUBSCRIPTIONS=1` in the runtime environment.

### Checkout fails before redirecting to Stripe

Cause:

- Missing or mismatched Stripe price ids.
- `referenceId` is missing because the client has no active organization.
- The current user is not authorized by `authorizeReference`.

Fix:

- Check `NEXT_PUBLIC_STRIPE_PRICE_PRO_MONTHLY` and `NEXT_PUBLIC_STRIPE_PRICE_PRO_YEARLY`.
- Confirm `authClient.useActiveOrganization()` returns the expected organization.
- Confirm the user is an owner or admin of that organization.
- Check Sentry events tagged `billing-checkout`.

### Customer portal does not open

Cause:

- `auth.api.createBillingPortal()` failed, usually because Stripe or Better Auth cannot resolve the organization subscription/customer.

Fix:

- Check Sentry events tagged `billing-portal`.
- Confirm the organization id is passed as `referenceId`.
- Confirm `BETTER_AUTH_URL` produces the expected return URL.
- Confirm Stripe customer/subscription records exist for the organization.

### Organization shows Basic after a successful payment

Cause:

- The Stripe webhook has not reached `/api/auth/stripe/webhook`, or cache still has stale subscription data.

Fix:

- Confirm the Stripe listener or production webhook endpoint is configured.
- Verify `STRIPE_WEBHOOK_SECRET`.
- Revisit `/dashboard/settings/billing`; `isProMember()` also reconciles org plan/status from active subscriptions.
- Revalidate `organization-${organizationId}-subscription` and `subscription-current` if adding new billing mutations.

### Public menu returns 404 for a paid organization

Cause:

- Public rendering only accepts `ACTIVE`, `TRIALING`, or `SPONSORED` organization statuses.
- The organization has no published menu or no published snapshot.

Fix:

- Check the organization's `status`.
- Confirm a menu is `PUBLISHED`.
- Confirm `publishedData` or `serialData` exists.
- Revalidate `subdomain-${slug}` after status or active menu changes.

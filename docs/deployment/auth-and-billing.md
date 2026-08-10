# Auth and Billing Deployment Runbook

Biztro uses Better Auth for authentication, organizations, invitations, and
Stripe-backed subscriptions. This runbook documents the production behavior that
developers and operators need to keep aligned with deployment configuration.

## Architecture

- `src/app/api/auth/[...all]/route.ts` exposes the Better Auth Next.js handler.
- `src/lib/auth.ts` configures the server auth instance, Prisma adapter,
  organization plugin, and Stripe plugin.
- `src/lib/auth-client.ts` configures the client plugins for organizations and
  subscription checkout.
- `src/server/actions/user/queries.ts` is the server-side source for active
  organization, membership, waitlist, and Pro entitlement checks.
- `src/server/actions/subscriptions/*` wraps Better Auth subscription APIs for
  current subscription data and Stripe customer portal creation.
- `src/app/dashboard/settings/billing/*` renders the owner-facing billing UI.

## Required Environment

The required variables are validated in `src/env.mjs`; values should be set in
Vercel and in local `.env` files used for development.

### Auth

- `BETTER_AUTH_SECRET` - required in production.
- `BETTER_AUTH_URL` - canonical application origin. `getBaseUrl()` uses this
  first when building invitation, portal, and callback URLs.
- `AUTH_GOOGLE_ID` and `AUTH_GOOGLE_SECRET` - Google OAuth credentials.
- `RESEND_API_KEY` - sends organization invitation emails.

`src/lib/auth.ts` also trusts these origins:

- `process.env.BETTER_AUTH_URL`
- `https://biztro.co`
- `https://preview.biztro.co`
- `https://${NEXT_PUBLIC_VERCEL_URL}` when it is a `*.vercel.app` host
- `http://localhost:3000`

### Stripe

- `STRIPE_SECRET_KEY`
- `STRIPE_WEBHOOK_SECRET`
- `NEXT_PUBLIC_STRIPE_PRICE_BASIC` - read by the server Stripe plugin for the
  `BASIC` plan price ID.
- `NEXT_PUBLIC_STRIPE_PRICE_PRO_MONTHLY`
- `NEXT_PUBLIC_STRIPE_PRICE_PRO_YEARLY`

The Stripe client is initialized with `apiVersion: Stripe.API_VERSION`, so the
runtime API version follows the installed `stripe` package instead of a pinned
literal in source.

### Feature flags

The billing page checks `subscriptionsEnabled()` from `src/flags.ts`. Set:

```sh
FLAGS_ENABLE_SUBSCRIPTIONS=1
```

The flag key is currently spelled `enable-suscriptions` in code; use the
environment variable above for deployment toggles.

## Signup and Waitlist Flow

New user creation is invite-only through the `Waitlist` table.

1. Marketing waitlist submissions call `joinWaitlist()` in
   `src/server/actions/organization/mutations.ts`.
2. `joinWaitlist()` creates a `Waitlist` row with `enabled = false`.
3. During Better Auth user creation, `databaseHooks.user.create.before` in
   `src/lib/auth.ts` extracts the candidate email from the Better Auth payload
   or context.
4. If an email is present, `isWaitlistEnabled(email)` requires a matching
   `Waitlist` row with `enabled = true`.
5. A denied signup throws `AccessDenied`; the Better Auth hook redirects the
   `unable_to_create_user` error path to `/auth-error?type=access_denied`.

Operationally, approving a waitlist user means setting that email's
`Waitlist.enabled` value to `true` before the user attempts signup.

Constraints:

- Email comparisons are lowercased and trimmed before lookup.
- The guard runs only when an email is found in the Better Auth payload/context.
- The `Waitlist.email` column is unique and indexed in `prisma/schema.prisma`.

## Organization and Session Behavior

The organization plugin exposes these additional Prisma `Organization` fields
through Better Auth:

- `description`
- `banner`
- `status`
- `plan`
- `updatedAt`

Organization creation uses Better Auth server APIs in
`src/server/actions/organization/mutations.ts`:

- new organizations default to `status: "ACTIVE"` and `plan: "BASIC"`;
- the slug is checked with `auth.api.checkOrganizationSlug()`;
- successful bootstrap sets the new organization as active;
- relevant cache tags are invalidated after create, update, and delete actions.

When Better Auth creates a session, `databaseHooks.session.create.before` sets
`activeOrganizationId` to the first organization returned by
`getActiveOrganization(userId)`. Session cookie caching is enabled for five
minutes in `src/lib/auth.ts`.

Organization invitation emails use `sendOrganizationInvitation()` in
`src/lib/utils.ts`, with links built as:

```txt
${getBaseUrl()}/invite/${invitationId}
```

## Billing and Subscription Flow

The Stripe plugin in `src/lib/auth.ts` enables subscriptions with two plans:

- `BASIC`
  - limits: `appConfig.menuLimit` and `appConfig.itemLimit`
- `PRO`
  - monthly and annual price IDs from the public Stripe price env vars
  - limits: 100 menus and 1000 products
  - free trial: 30 days

`authorizeReference()` allows subscription management when the current user is
an `owner` or `admin` member of the organization referenced by Stripe.

Client checkout starts from `BasicPlanView`:

```ts
authClient.subscription.upgrade({
  plan: Plan.PRO,
  annual: billingInterval === "yearly",
  referenceId: activeOrganization?.id,
  successUrl: "/dashboard/settings/billing",
  cancelUrl: "/dashboard/settings/billing"
})
```

Customer portal access goes through the server action
`createStripePortal(referenceId)`, which calls `auth.api.createBillingPortal()`
and returns users to:

```txt
${getBaseUrl()}/dashboard/settings/billing
```

The billing settings page has an additional UI guard: it only renders billing
controls when subscriptions are enabled and the current member role is
`OWNER`.

## Pro Entitlements

Use `isProMember()` from `src/server/actions/user/queries.ts` as the server-side
source of truth for Pro access.

It returns `true` when either:

- Better Auth reports an active or trialing subscription whose plan is `PRO`; or
- the organization status is `SPONSORED`.

When an active subscription exists and the organization is not sponsored,
`isProMember()` also syncs `Organization.plan` and `Organization.status` from
the subscription if they differ.

Current server-side Pro gates include:

- menu creation and publishing limits in `src/server/actions/menu/mutations.ts`;
- menu item creation/import limits in `src/server/actions/item/mutations.ts`;
- AI menu imports in `src/server/actions/menu-import/mutations.ts`;
- translations in `src/server/actions/item/translations.ts`;
- media upload limits in `src/app/api/file/route.ts`.

Public menus are discoverable/renderable for organizations whose status is one
of `ACTIVE`, `TRIALING`, or `SPONSORED`.

## Webhooks and Local Testing

Stripe webhooks are handled by the Better Auth route. The local forwarding
script in `package.json` is:

```sh
bun run stripe:listen
```

which runs:

```sh
stripe listen --forward-to localhost:3000/api/auth/stripe/webhook
```

Use the forwarded signing secret as `STRIPE_WEBHOOK_SECRET` for that local
session.

## Cache Tags

Subscription and organization reads use Next.js cache tags. Mutations update
these tags after successful changes:

- `organization-${organizationId}`
- `organization-${organizationId}-members`
- `organization-${organizationId}-subscription`
- `organization-current`
- `organizations-list`
- `membership-current`
- `membership-current-role`
- `permissions-all`
- `subscription-current`

If subscription or organization state is changed outside the app, expect cached
views to remain stale until their configured stale window expires or a matching
application action revalidates the affected tags.

## Common Failure Modes

### Signup redirects to access denied

Cause:

- The user's email is absent from `Waitlist`, or the row exists with
  `enabled = false`.

Fix:

- Set `Waitlist.enabled = true` for the normalized email and retry signup.

### Billing page shows the owner-only warning

Cause:

- The current member role is not `OWNER`, or `FLAGS_ENABLE_SUBSCRIPTIONS` is not
  set to `1`.

Fix:

- Confirm the active organization membership role and feature flag value.

### Checkout cannot start

Cause:

- The active organization is missing on the client, the Stripe price env vars are
  missing, or Better Auth rejects the subscription reference authorization.

Fix:

- Confirm the user has an active organization.
- Confirm `NEXT_PUBLIC_STRIPE_PRICE_PRO_MONTHLY` and
  `NEXT_PUBLIC_STRIPE_PRICE_PRO_YEARLY` are set.
- Confirm the user is an organization `owner` or `admin`.

### Customer portal returns null

Cause:

- `createStripePortal()` caught an error from Better Auth or Stripe.

Fix:

- Check Sentry events tagged `section: billing-portal`.
- Confirm `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, and `BETTER_AUTH_URL`.
- Confirm the referenced organization has a Stripe customer/subscription record.

### Pro access does not update after a Stripe change

Cause:

- Webhook delivery failed, the webhook signing secret is wrong, or cached
  subscription state is stale.

Fix:

- Verify Stripe webhook delivery to `/api/auth/stripe/webhook`.
- Confirm the webhook endpoint uses the same `STRIPE_WEBHOOK_SECRET` value as
  the deployment.
- Revisit the billing page or trigger an app action that updates
  `organization-${organizationId}-subscription`.

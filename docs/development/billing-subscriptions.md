# Billing and subscription workflow

Biztro uses Better Auth for organization membership and Stripe subscriptions,
with the Stripe integration mounted through the Better Auth auth route. This
guide covers the developer-facing flow for sign-in access, plan state, Pro
entitlements, and local troubleshooting.

## Source map

- `src/lib/auth.ts` configures Better Auth, the organization plugin, Stripe
  plans, invite email delivery, and session creation hooks.
- `src/lib/auth-client.ts` exposes the Better Auth React client with
  organization and Stripe client plugins.
- `src/app/api/auth/[...all]/route.ts` exposes Better Auth handlers, including
  Stripe webhook handling.
- `src/server/actions/user/queries.ts` resolves the current organization,
  membership, waitlist access, and `isProMember()`.
- `src/server/actions/subscriptions/*` reads active subscriptions and creates
  Stripe customer portal sessions.
- `src/app/dashboard/settings/billing/*` renders checkout, portal, and plan
  status UI.
- `src/app/config.ts` defines Basic-plan limits.
- `src/flags.ts` controls whether the billing settings UI is visible.

## Sign-in and account creation

Email/password auth is disabled. Users sign in through Google OAuth:

```ts
emailAndPassword: { enabled: false }
socialProviders: {
  google: {
    prompt: "select_account",
    clientId: process.env.AUTH_GOOGLE_ID!,
    clientSecret: process.env.AUTH_GOOGLE_SECRET!
  }
}
```

Account creation is invite-only through the `Waitlist` table. The
`databaseHooks.user.create.before` hook normalizes an email from the Better Auth
payload or context and calls `isWaitlistEnabled(email)`. Creation is blocked
when there is no matching row with `enabled: true`.

```ts
const email =
  typeof emailFromData === "string"
    ? emailFromData.trim().toLowerCase()
    : extractEmailFromContext(ctx)

if (email && !(await isWaitlistEnabled(email))) {
  throw new Error("AccessDenied")
}
```

The Better Auth error path maps `unable_to_create_user` to:

```txt
/auth-error?type=access_denied
```

### Enabling a local test user

Create or update a `Waitlist` row for the Google account email before testing a
first sign-in:

```ts
await prisma.waitlist.upsert({
  where: { email: "dev@example.com" },
  update: { enabled: true },
  create: { email: "dev@example.com", enabled: true }
})
```

Keep emails lowercase. `isWaitlistEnabled()` normalizes the incoming email and
matches it exactly against `Waitlist.email`.

## Organization session state

When Better Auth creates a session, `databaseHooks.session.create.before` calls
`getActiveOrganization(userId)` and writes the first membership's organization
ID into `Session.activeOrganizationId`.

Most dashboard queries then read the active organization through Better Auth
server APIs using the current request headers:

```ts
const currentOrg = await auth.api.getFullOrganization({
  headers: await headers()
})
```

Operational constraints:

- A user without an organization usually lands in not-found or empty dashboard
  states until an organization exists.
- Switching organizations must revalidate membership, permission, organization,
  and subscription cache tags.
- `getActiveOrganization()` currently picks the first membership returned by
  Prisma. Do not assume it is ordered by recency or role.

## Stripe plans and billing UI

The server-side Stripe plugin is configured in `src/lib/auth.ts`.

```ts
stripe({
  stripeClient,
  stripeWebhookSecret: process.env.STRIPE_WEBHOOK_SECRET!,
  createCustomerOnSignUp: true,
  subscription: {
    enabled: true,
    plans: [
      {
        name: "BASIC",
        priceId: process.env.NEXT_PUBLIC_STRIPE_PRICE_BASIC!,
        limits: {
          menus: appConfig.menuLimit,
          products: appConfig.itemLimit
        }
      },
      {
        name: "PRO",
        priceId: process.env.NEXT_PUBLIC_STRIPE_PRICE_PRO_MONTHLY!,
        annualDiscountPriceId: process.env.NEXT_PUBLIC_STRIPE_PRICE_PRO_YEARLY!,
        limits: { menus: 100, products: 1000 },
        freeTrial: { days: 30 }
      }
    ],
    authorizeReference: async ({ user, referenceId }) => {
      // Only organization owners/admins may manage billing.
    }
  }
})
```

The billing settings page is visible only when:

- `FLAGS_ENABLE_SUBSCRIPTIONS=1`
- the current member role is `OWNER`
- `subscriptionsEnabled()` resolves true

Checkout starts from `BasicPlanView` by calling:

```ts
authClient.subscription.upgrade({
  plan: Plan.PRO,
  annual: billingInterval === "yearly",
  referenceId: activeOrganization?.id,
  successUrl: "/dashboard/settings/billing",
  cancelUrl: "/dashboard/settings/billing"
})
```

Pro organizations use `createStripePortal(referenceId)` to create a Stripe
customer portal session and return users to `/dashboard/settings/billing`.

### Required environment variables

These variables are required by auth, billing, or the billing UI:

```txt
AUTH_GOOGLE_ID
AUTH_GOOGLE_SECRET
BETTER_AUTH_URL
STRIPE_SECRET_KEY
STRIPE_WEBHOOK_SECRET
NEXT_PUBLIC_STRIPE_PRICE_PRO_MONTHLY
NEXT_PUBLIC_STRIPE_PRICE_PRO_YEARLY
FLAGS_ENABLE_SUBSCRIPTIONS=1
```

`src/lib/auth.ts` also reads `NEXT_PUBLIC_STRIPE_PRICE_BASIC` for the Basic plan
configuration. Keep it available anywhere the Better Auth Stripe plugin is
initialized, even though the current env schema only validates the Pro price
IDs.

## Local webhook workflow

Use the repo script to forward Stripe webhook events to Better Auth:

```sh
bun run stripe:listen
```

The script forwards to:

```txt
localhost:3000/api/auth/stripe/webhook
```

Copy the webhook signing secret from the Stripe CLI output into
`STRIPE_WEBHOOK_SECRET` for the dev server process, then restart the dev server.

## Pro entitlement rules

Use `isProMember()` as the server-side source of truth for Pro features.

`isProMember()`:

1. resolves the current organization,
2. lists active subscriptions for `referenceId = organization.id`,
3. treats `active` and `trialing` subscriptions as active,
4. synchronizes `Organization.plan` and `Organization.status` from Stripe when
   the org is not sponsored,
5. returns true when the active subscription plan is `PRO` or the organization
   status is `SPONSORED`.

Creation actions gate Basic limits on the server:

- Basic menus: `appConfig.menuLimit` (`1`)
- Basic products: `appConfig.itemLimit` (`10`)
- Pro and sponsored organizations bypass these Basic limits

Do not rely on client plan state for enforcement. Client state is only UX; the
server action must check `isProMember()` before creating server-backed data.

## Sponsored organizations

An organization with `status = "SPONSORED"` is treated as Pro without requiring
an active Stripe subscription. `ProPlanView` renders a sponsored notice instead
of Stripe subscription details, and `isProMember()` returns true for sponsored
organizations.

Use this state for comped or manually managed organizations. Do not overwrite a
sponsored org's plan/status from Stripe sync logic.

## Cache tags to revalidate

Subscription and membership mutations should revalidate the relevant tags:

- `subscription-current`
- `organization-${organizationId}-subscription`
- `organization-${organizationId}`
- `organization-${organizationId}-members`
- `membership-current`
- `membership-current-role`
- `permissions-all`

The current code uses both dash-separated and colon-separated tag names in a few
older user actions. When adding new code, prefer the existing dash-separated
subscription tags used by `getCurrentSubscription()` and `isProMember()`.

## Common failure modes

### OAuth succeeds but account creation redirects to access denied

Cause:

- The user's email is missing from `Waitlist`, or the row exists with
  `enabled = false`.

Fix:

- Enable the email in `Waitlist`.
- Confirm the email is lowercase and exactly matches the Google account email.

### Billing page shows the owner warning

Cause:

- `FLAGS_ENABLE_SUBSCRIPTIONS` is not `1`, or the active member is not an
  organization owner.

Fix:

- Set `FLAGS_ENABLE_SUBSCRIPTIONS=1`.
- Confirm the active organization's `Member.role` is `owner`.

### Checkout fails to start

Cause:

- Missing Stripe price environment variables.
- No active organization is available from `authClient.useActiveOrganization()`.
- The user is not authorized by `authorizeReference()` for the organization.

Fix:

- Verify Pro monthly/yearly price IDs and the Basic price ID.
- Confirm the current session has `activeOrganizationId`.
- Confirm the member role is owner or admin for the organization reference.

### Plan stays Basic after a successful payment

Cause:

- Stripe webhooks are not reaching Better Auth, or the signing secret does not
  match the forwarded webhook endpoint.

Fix:

- Run `bun run stripe:listen`.
- Set `STRIPE_WEBHOOK_SECRET` from the Stripe CLI output.
- Restart the dev server after changing the secret.
- Reopen `/dashboard/settings/billing` to trigger fresh server reads and client
  query invalidation.

### Organization deletion is blocked

Cause:

- `deleteOrganization` prevents deleting organizations with `active` or
  `trialing` subscriptions.

Fix:

- Cancel or end the active Stripe subscription first, then retry deletion after
  subscription state is no longer active/trialing.

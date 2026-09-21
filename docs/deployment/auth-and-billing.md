# Authentication and Billing Deployment

This guide documents the current production-facing auth, organization, invite,
waitlist, and billing flow. It is based on the implementation in
`src/lib/auth.ts`, `src/lib/auth-client.ts`, `src/server/actions/user/*`,
`src/server/actions/subscriptions/*`, and the billing settings pages.

## Architecture

- Better Auth is exposed through `src/app/api/auth/[...all]/route.ts`.
- The server configuration lives in `src/lib/auth.ts`.
- The client uses `authClient` from `src/lib/auth-client.ts` with the
  organization and Stripe client plugins.
- Prisma stores Better Auth data in `prisma/models/auth.prisma`; app-owned
  waitlist data lives in the `Waitlist` model in `prisma/schema.prisma`.
- Stripe subscriptions are organization-scoped by passing the organization ID as
  the subscription `referenceId`.

## Sign-in and account creation

The app currently enables Google sign-in and disables email/password sign-in:

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

Account creation is invite/waitlist gated in the Better Auth user create
database hook:

1. `src/lib/auth.ts` extracts an email from the Better Auth payload/context.
2. `isWaitlistEnabled(email)` checks for a matching `Waitlist` row with
   `enabled: true`.
3. If the email is present and not enabled, the hook throws `AccessDenied`.
4. Better Auth creation errors with `unable_to_create_user` are redirected to
   `/auth-error?type=access_denied`.

The public waitlist form only inserts a disabled row through `joinWaitlist()`.
Granting access is an operational step: set the matching `Waitlist.enabled`
value to `true` before the user creates an account.

## Organization lifecycle

Organizations are managed through Better Auth's organization plugin with custom
fields mapped to the Prisma `Organization` model:

- `description`
- `banner`
- `status`
- `plan`
- `updatedAt`

`bootstrapOrg()` and `createOrg()` create organizations with:

- `status: "ACTIVE"`
- `plan: "BASIC"`
- the provided `slug`
- the requesting user as the owner/current active organization

`getActiveOrganization(userId)` returns the first membership-backed
organization for a user and is used when Better Auth creates a session to set
`activeOrganizationId`.

Deleting an organization is blocked while `getCurrentSubscription()` finds an
active or trialing subscription. If deletion is allowed, R2 organization assets
are deleted first, then Better Auth deletes the organization, and the current
session is signed out best-effort.

## Invitations

Member invitations use Better Auth organization APIs:

- `inviteMember()` calls `auth.api.createInvitation()`.
- `sendInvitationEmail()` builds `/invite/{id}` links from `getBaseUrl()` and
  sends the email with Resend.
- `/invite/[id]` fetches the invitation with `auth.api.getInvitation()`.
- `acceptInvite()` calls `auth.api.acceptInvitation()`.

`getBaseUrl()` prefers `BETTER_AUTH_URL`, then production `https://biztro.co`,
then the Vercel preview URL, and finally `http://localhost:3000`.

## Billing and plan entitlement

Stripe is configured through the Better Auth Stripe plugin in `src/lib/auth.ts`.
The server creates a Stripe client with:

```ts
const stripeClient = new Stripe(process.env.STRIPE_SECRET_KEY!, {
  apiVersion: Stripe.API_VERSION,
});
```

Subscription plans are defined in the Better Auth Stripe plugin:

- `BASIC` uses `appConfig.menuLimit` and `appConfig.itemLimit`.
- `PRO` has monthly and yearly Stripe price IDs and a 30-day free trial.

Checkout starts from `BasicPlanView`:

```ts
authClient.subscription.upgrade({
  plan: Plan.PRO,
  annual: billingInterval === "yearly",
  referenceId: activeOrganization?.id,
  successUrl: "/dashboard/settings/billing",
  cancelUrl: "/dashboard/settings/billing",
});
```

Only organization owners can view the billing settings page, and the page is
hidden unless `FLAGS_ENABLE_SUBSCRIPTIONS=1`.

The customer portal flow is server-backed:

1. `CustomerPortalButton` calls `createStripePortal(referenceId)`.
2. `createStripePortal()` calls `auth.api.createBillingPortal()` with the
   organization ID and a return URL of `/dashboard/settings/billing`.
3. Subscription cache tags are updated before returning the Stripe portal URL.

### Pro source of truth

Use `isProMember()` as the server-side entitlement check. It returns `true`
when either:

- Better Auth lists an active or trialing `PRO` subscription for the active
  organization ID, or
- the organization `status` is `SPONSORED`.

When a non-sponsored organization has an active subscription whose plan/status
differs from the organization row, `isProMember()` updates the organization
metadata through `auth.api.updateOrganization()`.

Current server-side Pro gates include:

- menu creation and duplication (`src/server/actions/menu/mutations.ts`)
- menu item creation and bulk import (`src/server/actions/item/mutations.ts`)
- translations and AI import flows
- file uploads that require Pro features or exceed the free media limit

Client upgrade dialogs are UX only. Keep new Pro-only mutations gated in their
server action or route handler.

## Required configuration

Validate required environment variables in `src/env.mjs` before deploying.
Auth and billing rely on:

- `BETTER_AUTH_SECRET`
- `BETTER_AUTH_URL`
- `AUTH_GOOGLE_ID`
- `AUTH_GOOGLE_SECRET`
- `RESEND_API_KEY`
- `STRIPE_SECRET_KEY`
- `STRIPE_WEBHOOK_SECRET`
- `NEXT_PUBLIC_STRIPE_PRICE_PRO_MONTHLY`
- `NEXT_PUBLIC_STRIPE_PRICE_PRO_YEARLY`

`src/lib/auth.ts` also references `NEXT_PUBLIC_STRIPE_PRICE_BASIC` for the
Better Auth `BASIC` plan configuration. If the target Stripe setup requires a
real Basic price, configure that value consistently with the code.

Set `FLAGS_ENABLE_SUBSCRIPTIONS=1` in environments where owners should see the
billing page.

If adding a new production or preview auth domain, update `trustedOrigins` in
`src/lib/auth.ts`; the current list includes:

- `BETTER_AUTH_URL`
- `https://biztro.co`
- `https://preview.biztro.co`
- a valid `NEXT_PUBLIC_VERCEL_URL` ending in `.vercel.app`
- `http://localhost:3000`

## Local webhook testing

Use the repository script to forward Stripe webhooks to Better Auth:

```sh
bun run stripe:listen
```

The script forwards to:

```txt
localhost:3000/api/auth/stripe/webhook
```

Make sure the printed Stripe webhook signing secret matches
`STRIPE_WEBHOOK_SECRET` in the local environment before testing checkout,
subscription updates, cancellations, or trial transitions.

## Troubleshooting

### New users are redirected to the access denied page

Check the `Waitlist` row for the normalized email. The user creation hook
requires `enabled: true` when it can extract an email from the auth payload.

### Billing page shows the owner-only warning

Confirm both conditions:

- the current member role is `owner`
- `FLAGS_ENABLE_SUBSCRIPTIONS=1`

### Checkout starts for the wrong organization

Stripe checkout uses `authClient.useActiveOrganization()` on the client and
passes that ID as `referenceId`. Switch the active organization before starting
checkout, then retry.

### Pro features stay locked after a Stripe update

`isProMember()` only treats active or trialing `PRO` subscriptions as Pro,
unless the organization status is `SPONSORED`. Check the Stripe subscription
status, the Better Auth `Subscription.referenceId`, and webhook delivery to
`/api/auth/stripe/webhook`.

### Sponsored organizations do not show subscription details

That is expected. `ProPlanView` returns a sponsored organization alert when
`Organization.status` is `SPONSORED` and does not require a Stripe subscription.

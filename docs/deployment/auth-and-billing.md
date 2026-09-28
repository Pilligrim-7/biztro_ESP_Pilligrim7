# Auth, Organizations, and Billing Runbook

Biztro uses Better Auth for sign-in, organization membership, invitations, and
Stripe subscription management. This document describes the deployed flow and the
source files to check when auth or billing behavior changes.

## Architecture

- `src/app/api/auth/[...all]/route.ts` exposes Better Auth's Next.js handlers.
- `src/lib/auth.ts` configures Better Auth with:
  - the Prisma adapter,
  - Google OAuth,
  - the organization plugin,
  - the Stripe subscription plugin,
  - user/session database hooks,
  - trusted origins for production, preview, and local development.
- `src/lib/auth-client.ts` creates the client with the organization and Stripe
  client plugins.
- Auth-related Prisma models live in `prisma/models/auth.prisma`; domain models
  live in `prisma/schema.prisma`.

The app stores the active organization on the session through
`session.activeOrganizationId`. When a session is created, `src/lib/auth.ts`
looks up the user's first membership with `getActiveOrganization()` and writes
that organization id into the session.

## Beta Access and Sign-in

Email/password sign-up is disabled. Google OAuth is the active social provider.

New account creation is waitlist gated. Invitations add users to organizations
after they are authenticated, but a brand-new invited user still goes through
the same account creation hook:

1. The Better Auth user create database hook extracts an email from the creation
   payload or auth context.
2. `isWaitlistEnabled()` checks `Waitlist.enabled` for that normalized email.
3. If there is no enabled waitlist row, the hook throws `AccessDenied`.
4. Better Auth surfaces that as `unable_to_create_user`; the auth `before` hook
   redirects to `/auth-error?type=access_denied`.
5. The auth error page shows the beta waitlist flow.

Relevant paths:

- `src/lib/auth.ts`
- `src/server/actions/user/queries.ts`
- `src/app/(auth)/auth-error/page.tsx`
- `prisma/schema.prisma` (`Waitlist`)

## Organization Invitations

Organization invitations are handled by the Better Auth organization plugin.

- `inviteMember()` creates an invitation with role `member`.
- `sendInvitationEmail()` sends the Resend email with `/invite/{id}`.
- `/invite/[id]` asks anonymous users to sign in first, then shows the accept
  card to authenticated users.
- `acceptInvite()` accepts the invitation through Better Auth and invalidates the
  organization, membership, permissions, and settings cache tags.

Relevant paths:

- `src/server/actions/user/mutations.ts`
- `src/app/(auth)/invite/[id]/page.tsx`
- `src/app/(auth)/invite/[id]/accept-invite-card.tsx`
- `src/lib/utils.ts`

## Billing Flow

Stripe billing is organization-scoped. The Stripe plugin stores subscriptions
with the organization id as `referenceId`.

### Plans

The Better Auth Stripe plugin registers:

- `BASIC`
  - limits: `appConfig.menuLimit` menus and `appConfig.itemLimit` products.
- `PRO`
  - monthly and annual Stripe price ids.
  - 30-day free trial.
  - plugin limits of 100 menus and 1000 products.

`src/lib/types/billing.ts` mirrors the UI tiers and display prices. Keep the
plugin plan configuration in `src/lib/auth.ts`, UI tiers, and Stripe dashboard
prices in sync.

### Checkout and Portal

- The billing page renders plan management only when `subscriptionsEnabled()`
  returns true and the current member role is `OWNER`; otherwise it renders an
  owner-only notice.
- `FLAGS_ENABLE_SUBSCRIPTIONS=1` enables the billing UI.
- Basic-plan users start checkout from the client with
  `authClient.subscription.upgrade()`.
- The checkout `referenceId` is the active organization id.
- Pro users open the Stripe customer portal through the server action
  `createStripePortal()`, which calls `auth.api.createBillingPortal()`.
- The local webhook helper forwards events to
  `/api/auth/stripe/webhook`.

Relevant paths:

- `src/app/dashboard/settings/billing/page.tsx`
- `src/app/dashboard/settings/billing/basic-plan-view.tsx`
- `src/app/dashboard/settings/billing/pro-plan-view.tsx`
- `src/app/dashboard/settings/billing/customer-portal-button.tsx`
- `src/server/actions/subscriptions/mutations.ts`
- `src/server/actions/subscriptions/queries.ts`
- `src/flags.ts`

### Subscription Entitlements

`isProMember()` is the server-side source of truth for Pro access:

- It reads the current organization.
- It lists active subscriptions for the organization `referenceId`.
- It treats `active` and `trialing` subscriptions as active.
- If the organization's stored plan/status differs from Stripe and the
  organization is not `SPONSORED`, it updates the organization through Better
  Auth.
- It returns true for a PRO subscription or a `SPONSORED` organization status.

Server actions must call `isProMember()` before allowing Pro-only creation or
generation. Do not rely on client-side plan state for authorization.

Current server-side gates:

- `createMenu()` blocks Basic users at `appConfig.menuLimit`.
- `createItem()` and `bulkCreateItems()` block Basic users at
  `appConfig.itemLimit`.
- Menu translation actions require Pro access and `AI_GATEWAY_API_KEY`.

Relevant paths:

- `src/server/actions/user/queries.ts`
- `src/server/actions/menu/mutations.ts`
- `src/server/actions/item/mutations.ts`
- `src/server/actions/item/translations.ts`
- `src/app/config.ts`

## Public Menu Availability

Public organization slugs are considered active when the organization status is:

- `ACTIVE`
- `TRIALING`
- `SPONSORED`

Those statuses are used when generating static subdomain params and metadata for
published menus. `getActiveMenuByOrganizationSlug()` also checks those statuses
before serving the organization's active published menu, falling back to the
most recently published menu for that organization. See
`docs/deployment/subdomain-routing.md` for DNS and Worker deployment details.

Relevant paths:

- `src/app/[subdomain]/page.tsx`
- `src/server/actions/organization/queries.ts`
- `src/proxy.ts`

## Required Environment

Auth and billing depend on these environment variables:

- `BETTER_AUTH_URL`
- `BETTER_AUTH_SECRET`
- `AUTH_GOOGLE_ID`
- `AUTH_GOOGLE_SECRET`
- `RESEND_API_KEY`
- `STRIPE_SECRET_KEY`
- `STRIPE_WEBHOOK_SECRET`
- `NEXT_PUBLIC_STRIPE_PRICE_BASIC`
- `NEXT_PUBLIC_STRIPE_PRICE_PRO_MONTHLY`
- `NEXT_PUBLIC_STRIPE_PRICE_PRO_YEARLY`
- `FLAGS_ENABLE_SUBSCRIPTIONS` for the dashboard billing UI

`src/lib/auth.ts` also reads `NEXT_PUBLIC_STRIPE_PRICE_BASIC` for the BASIC plan,
while `src/env.mjs` currently validates only the public Pro price ids. Keep the
auth config and env validation aligned when changing Stripe plans.

## Local Billing Checklist

1. Ensure the auth, Google OAuth, Resend, and Stripe variables are present.
2. Start the app with `bun run dev`.
3. In another terminal, run:

   ```bash
   bun run stripe:listen
   ```

4. Sign in with a waitlist-enabled email or an invited account.
5. Create or select an organization.
6. Set `FLAGS_ENABLE_SUBSCRIPTIONS=1` before starting the app if you need to see
   the billing page.
7. Open `/dashboard/settings/billing`.
8. Start checkout from the Basic plan or open the customer portal from an active
   Pro plan.
9. Confirm webhook delivery in the Stripe CLI output and verify the
   organization subscription cache tags refresh after checkout or portal changes.

## Common Failure Modes

### New users are sent to the beta waitlist page

Cause:

- The email does not have an enabled `Waitlist` row.

Fix:

- Add or enable the waitlist record for the normalized email before sign-in.

### Billing page shows an owner-only notice

Cause:

- The current member is not `OWNER`, or `FLAGS_ENABLE_SUBSCRIPTIONS` is not set
  to `1`.

Fix:

- Switch to an owner account and enable the flag before starting the app.

### Checkout or customer portal does not open

Cause:

- Missing Stripe environment, missing active organization, or an organization id
  that fails the Stripe plugin reference authorization.

Fix:

- Verify Stripe keys and price ids.
- Confirm the active organization id is being passed as `referenceId`.
- Confirm the current member is an owner or admin for the organization.

### Plan status does not update after Stripe events

Cause:

- Webhooks are not reaching `/api/auth/stripe/webhook`, or cached subscription
  tags have not been refreshed yet.

Fix:

- Run `bun run stripe:listen` locally and repeat the Stripe event.
- Check `STRIPE_WEBHOOK_SECRET`.
- Revisit `isProMember()` and subscription cache tags if the event succeeds but
  the UI remains stale.

# Billing and Subscriptions

Biztro uses Better Auth's Stripe plugin to manage organization-scoped
subscriptions. Billing state is used for dashboard access, Pro feature
entitlements, and public menu availability.

## Architecture

- `src/lib/auth.ts` configures Better Auth, the organization plugin, and the
  Stripe plugin.
- `src/app/api/auth/[...all]/route.ts` exposes the Better Auth API routes,
  including the Stripe webhook endpoint.
- `src/lib/auth-client.ts` registers the client-side Stripe subscription
  plugin.
- `src/app/dashboard/settings/billing/*` renders checkout, portal, and plan
  status UI.
- `src/server/actions/subscriptions/*` wraps billing portal creation and active
  subscription reads.
- `src/server/actions/user/queries.ts` exposes `isProMember()`, the server-side
  entitlement source of truth.
- `prisma/models/auth.prisma` stores Better Auth data, including
  `User.stripeCustomerId` and `Subscription`.

## Runtime Configuration

Required Stripe-related variables:

| Variable                               | Used by                        | Notes                                                                                                        |
| -------------------------------------- | ------------------------------ | ------------------------------------------------------------------------------------------------------------ |
| `STRIPE_SECRET_KEY`                    | Server Stripe client           | Secret key for API calls.                                                                                    |
| `STRIPE_WEBHOOK_SECRET`                | Better Auth Stripe plugin      | Must match the webhook endpoint receiving events.                                                            |
| `NEXT_PUBLIC_STRIPE_PRICE_BASIC`       | Better Auth Stripe plan config | Read directly in `src/lib/auth.ts`; keep present even though it is not currently validated by `src/env.mjs`. |
| `NEXT_PUBLIC_STRIPE_PRICE_PRO_MONTHLY` | Pro monthly plan               | Validated in `src/env.mjs` and used by checkout and display code.                                            |
| `NEXT_PUBLIC_STRIPE_PRICE_PRO_YEARLY`  | Pro yearly plan                | Used as the annual discount price ID.                                                                        |
| `FLAGS_ENABLE_SUBSCRIPTIONS`           | Billing page flag              | Set to `1` to show billing UI to organization owners.                                                        |

The Stripe client uses `Stripe.API_VERSION` from the installed Stripe SDK. Keep
the `stripe` package and `@better-auth/stripe` versions compatible rather than
hardcoding an older API version.

## Local Webhook Workflow

1. Start the app:

   ```bash
   bun run dev
   ```

2. In another terminal, forward Stripe webhooks to Better Auth:

   ```bash
   bun run stripe:listen
   ```

   This runs:

   ```bash
   stripe listen --forward-to localhost:3000/api/auth/stripe/webhook
   ```

3. Copy the signing secret printed by the Stripe CLI into
   `STRIPE_WEBHOOK_SECRET`.
4. Trigger checkout from `/dashboard/settings/billing` while signed in as the
   active organization's owner.
5. After Stripe redirects back, refresh the billing page if needed. Subscription
   reads are cached briefly with the `organization-{id}-subscription` tag.

## User Flows

### Upgrade to Pro

1. `BasicPlanView` calls `authClient.subscription.upgrade()`.
2. The request includes:
   - `plan: Plan.PRO`
   - `annual: true` for yearly checkout
   - `referenceId: activeOrganization.id`
   - success and cancel URLs pointing back to the billing page
3. Better Auth creates the Stripe checkout session and records subscription
   changes from webhook events.

### Manage an existing subscription

1. `ProPlanView` loads the active subscription with
   `getCurrentSubscription(organizationId)`.
2. `CustomerPortalButton` calls the `createStripePortal()` server action.
3. The server action calls `auth.api.createBillingPortal()` with the
   organization ID as `referenceId` and redirects the user to Stripe.

Only organization owners see the billing controls. The Stripe plugin also calls
`authorizeReference` in `src/lib/auth.ts`, which allows only owners and admins
to manage a subscription reference.

### Sponsored organizations

Organizations with `status === "SPONSORED"` are treated as Pro without requiring
an active Stripe subscription. The billing page shows a sponsored-plan notice
instead of the Stripe portal card.

## Entitlements and Limits

Use `isProMember()` on the server for feature gates. It returns true when the
active organization has:

- an active or trialing Better Auth subscription whose plan is `PRO`, or
- organization status `SPONSORED`.

When an active subscription plan differs from `Organization.plan`,
`isProMember()` attempts to update the organization record unless the
organization is sponsored.

Basic plan limits come from `src/app/config.ts`:

- `itemLimit: 10`
- `menuLimit: 1`

Server actions enforce these limits for item and menu creation. Pro-only flows
also include full-menu import from uploaded files and menu item translations.
Client components may show `UpgradeDialog` for UX, but new server-backed
creation or generation flows should still call `isProMember()` before mutating
data.

Public menu lookup allows organizations whose status is `ACTIVE`, `TRIALING`,
or `SPONSORED`. If webhook handling changes organization status to a terminal
state, public menu availability can change even if a published menu still
exists.

## Cache Tags

Billing and organization changes should revalidate the relevant Next cache tags:

- `organization-{organizationId}-subscription`
- `subscription-current`
- `organization-{organizationId}`
- `membership-current`
- `membership-current-role`

Use the existing mutation patterns before adding new cache tags; some older
actions still use colon-delimited tag names, so avoid copying those into new
code unless the surrounding call path already depends on them.

## Common Pitfalls

- Billing UI stays hidden unless `FLAGS_ENABLE_SUBSCRIPTIONS=1` and the active
  member is the organization owner.
- Checkout may fail if `referenceId` is missing because the client did not load
  an active organization.
- Webhook updates will not persist locally unless the Stripe CLI is forwarding
  to `/api/auth/stripe/webhook` and `STRIPE_WEBHOOK_SECRET` matches the CLI
  secret.
- A Pro subscription without `plan: "PRO"` will not grant entitlements because
  `isProMember()` compares the plan name, not only the Stripe status.
- Keep Stripe price IDs in sync with `src/lib/types/billing.ts`; those values
  are also used to display monthly vs yearly pricing on the Pro plan card.

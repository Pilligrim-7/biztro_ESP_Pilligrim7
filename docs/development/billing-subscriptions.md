# Billing and Subscriptions

Biztro uses Better Auth's Stripe plugin to manage organization subscriptions.
The billing surface is intentionally organization-scoped: Stripe subscriptions use
the organization id as `referenceId`, and server actions check the current
organization before enforcing limits or creating Stripe sessions.

## Source Map

- `src/lib/auth.ts` configures Better Auth, the organization plugin, and the
  Stripe plugin.
- `src/lib/auth-client.ts` enables the Better Auth organization and Stripe
  clients on the frontend.
- `src/app/dashboard/settings/billing/*` renders the billing page, checkout
  entry point, customer portal button, and subscription status UI.
- `src/server/actions/subscriptions/*` wraps subscription reads and the Stripe
  customer portal action.
- `src/server/actions/user/queries.ts` contains `isProMember()`, the server-side
  entitlement source of truth.
- `src/server/actions/menu/mutations.ts` and
  `src/server/actions/item/mutations.ts` enforce Basic plan creation limits.

## Required Configuration

Set the shared application environment from `src/env.mjs`, plus the subscription
values consumed directly by the auth configuration.

| Variable                               | Purpose                                                         |
| -------------------------------------- | --------------------------------------------------------------- |
| `STRIPE_SECRET_KEY`                    | Initializes the server-side Stripe client.                      |
| `STRIPE_WEBHOOK_SECRET`                | Verifies Better Auth Stripe webhook requests.                   |
| `NEXT_PUBLIC_STRIPE_PRICE_PRO_MONTHLY` | Monthly Pro price id.                                           |
| `NEXT_PUBLIC_STRIPE_PRICE_PRO_YEARLY`  | Yearly Pro price id.                                            |
| `NEXT_PUBLIC_STRIPE_PRICE_BASIC`       | Basic plan price id referenced by `src/lib/auth.ts`.            |
| `FLAGS_ENABLE_SUBSCRIPTIONS`           | Set to `1` to expose the billing page.                          |
| `BETTER_AUTH_URL`                      | Base URL used for auth, checkout, portal, and invitation links. |

> `NEXT_PUBLIC_STRIPE_PRICE_BASIC` is read in `src/lib/auth.ts` but is not
> currently validated by `src/env.mjs`, so a missing value may fail later than
> startup.

## Local Workflow

1. Start the app and local database.

   ```sh
   bun run db:dev
   bun run dev
   ```

2. Forward Stripe webhooks to the Better Auth catch-all route.

   ```sh
   bun run stripe:listen
   ```

   The script forwards to:

   ```txt
   localhost:3000/api/auth/stripe/webhook
   ```

3. Copy the webhook signing secret from the Stripe CLI output into
   `STRIPE_WEBHOOK_SECRET`.
4. Set `FLAGS_ENABLE_SUBSCRIPTIONS=1` for local billing-page access.
5. Sign in as an organization owner. Non-owners see a warning instead of billing
   controls.

## Runtime Flow

### Checkout

`BasicPlanView` calls `authClient.subscription.upgrade()` with:

- `plan: "PRO"`
- `annual: true` for yearly billing, `false` for monthly billing
- `referenceId: activeOrganization.id`
- success and cancel URLs pointing back to `/dashboard/settings/billing`

Better Auth creates Stripe customers on sign-up (`createCustomerOnSignUp: true`)
and writes subscription records through the Stripe webhook flow.

### Customer Portal

`CustomerPortalButton` calls the server action `createStripePortal(referenceId)`.
That action:

- calls `auth.api.createBillingPortal()`
- sets the return URL to `/dashboard/settings/billing`
- refreshes the cache tags `organization-{id}-subscription` and
  `subscription-current`
- returns the portal URL for client-side navigation

## Entitlements and Limits

Use `isProMember()` for server-side entitlement checks. It returns true when:

- the current organization has an active or trialing Stripe subscription whose
  plan is `PRO`, or
- the organization status is `SPONSORED`

It also updates the organization plan/status when the active subscription differs
from the organization record, except for sponsored organizations.

Basic plan limits are defined in `src/app/config.ts`:

- `menuLimit: 1`
- `itemLimit: 10`
- `mediaLimit: 20`

The server enforces creation limits in these paths:

- `createMenu()` and `duplicateMenu()` block non-Pro organizations at
  `menuLimit`.
- `createItem()` blocks non-Pro organizations at `itemLimit`.
- `bulkCreateItems()` blocks imports that would exceed `itemLimit`.

Client components may show upgrade dialogs for UX, but the server actions above
are the enforcement boundary.

## Sponsored Organizations

Organizations with `status: "SPONSORED"` receive Pro access without a Stripe
subscription. The billing page renders a sponsored-plan notice and skips the
Stripe portal. Public menu routing also treats sponsored organizations as active.

Use this status intentionally; it bypasses normal Stripe subscription checks.

## Troubleshooting

### Billing page shows an access warning

Check both conditions:

- `FLAGS_ENABLE_SUBSCRIPTIONS=1`
- the current member role is `owner`

### Checkout starts but subscription state does not update

- Confirm `bun run stripe:listen` is running.
- Confirm the forwarded endpoint is `/api/auth/stripe/webhook`.
- Confirm `STRIPE_WEBHOOK_SECRET` matches the current Stripe CLI forwarding
  secret.
- Revisit `/dashboard/settings/billing`; the page and portal action refresh
  subscription-related cache tags.

### Pro features remain locked after checkout

Verify `isProMember()` can find an active or trialing subscription for the
organization id used as `referenceId`. If the subscription belongs to a different
reference id, the org remains Basic.

### Public menu returns 404 for a paid organization

Public subdomain rendering only serves organizations with status `ACTIVE`,
`TRIALING`, or `SPONSORED`, and only when a published menu exists.

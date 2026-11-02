# Organization Billing and Subscriptions

Biztro uses the Better Auth Stripe plugin to bill organizations, not individual menus or users. The current implementation has two customer-facing plans:

- `BASIC` - free, limited by `src/app/config.ts`.
- `PRO` - paid monthly or yearly through Stripe Checkout.

Organizations can also be marked `SPONSORED`, which grants Pro access without requiring an active Stripe subscription.

## Architecture

The billing flow is split across these codepaths:

- `src/lib/auth.ts`
  - Initializes the Better Auth Stripe plugin.
  - Defines the `BASIC` and `PRO` plans passed to Better Auth.
  - Creates Stripe customers on sign-up.
  - Authorizes subscription management only for organization owners or admins.
- `src/lib/auth-client.ts`
  - Registers the Better Auth organization and Stripe client plugins.
- `src/app/api/auth/[...all]/route.ts`
  - Exposes Better Auth API routes, including the Stripe webhook endpoint.
- `src/app/dashboard/settings/billing/*`
  - Renders the billing settings page, Checkout button, current plan state, and Customer Portal button.
- `src/server/actions/subscriptions/*`
  - Reads active subscriptions and creates Stripe Customer Portal sessions.
- `src/server/actions/user/queries.ts`
  - Implements `isProMember()`, the server-side source of truth for Pro entitlement checks.

## Required configuration

Set these variables in local development and deployment environments:

| Variable                               | Purpose                                                           |
| -------------------------------------- | ----------------------------------------------------------------- |
| `STRIPE_SECRET_KEY`                    | Server-side Stripe API key used by the Better Auth Stripe plugin. |
| `STRIPE_WEBHOOK_SECRET`                | Signing secret for Better Auth's Stripe webhook handler.          |
| `NEXT_PUBLIC_STRIPE_PRICE_PRO_MONTHLY` | Stripe Price ID for the monthly Pro plan.                         |
| `NEXT_PUBLIC_STRIPE_PRICE_PRO_YEARLY`  | Stripe Price ID for the yearly Pro plan.                          |
| `FLAGS_ENABLE_SUBSCRIPTIONS`           | Set to `1` to enable the billing settings UI.                     |
| `BETTER_AUTH_URL`                      | Base URL used for auth callbacks and Stripe return URLs.          |

`src/lib/auth.ts` currently also references `NEXT_PUBLIC_STRIPE_PRICE_BASIC` for the Basic plan passed to the Better Auth plugin. `src/env.mjs` validates only the Pro price IDs, so keep these files aligned if the Basic plan configuration changes.

## Local development workflow

1. Start the app and the local database:

   ```bash
   bun run db:dev
   bun run dev
   ```

2. In a second terminal, forward Stripe webhooks to Better Auth:

   ```bash
   bun run stripe:listen
   ```

   The script forwards events to:

   ```text
   http://localhost:3000/api/auth/stripe/webhook
   ```

3. Copy the webhook signing secret printed by the Stripe CLI into `STRIPE_WEBHOOK_SECRET`.

4. Set `FLAGS_ENABLE_SUBSCRIPTIONS=1` so `/dashboard/settings/billing` renders the Checkout and portal UI.

5. Create or select an organization, then open:

   ```text
   /dashboard/settings/billing
   ```

Only organization owners can use the billing page UI. Non-owners see a warning, even when subscriptions are enabled.

## Checkout and portal flows

### Upgrade to Pro

The Basic plan screen calls:

```ts
authClient.subscription.upgrade({
  plan: Plan.PRO,
  annual: billingInterval === "yearly",
  referenceId: activeOrganization?.id,
  successUrl: "/dashboard/settings/billing",
  cancelUrl: "/dashboard/settings/billing",
});
```

The `referenceId` is the active organization ID. Better Auth uses it to associate the Stripe subscription with the organization.

### Manage an existing subscription

`CustomerPortalButton` calls `createStripePortal(referenceId)`, which uses:

```ts
auth.api.createBillingPortal({
  body: {
    referenceId,
    returnUrl: `${getBaseUrl()}/dashboard/settings/billing`,
  },
  headers: await headers(),
});
```

The server action refreshes these cache tags after creating the portal session:

- `organization-${referenceId}-subscription`
- `subscription-current`

## Pro entitlement checks

Use `isProMember()` for server-side gating. Client plan state is only UX.

`isProMember()`:

1. Loads the current organization with Better Auth.
2. Lists active subscriptions for that organization.
3. Treats `active` and `trialing` subscriptions with plan `PRO` as Pro.
4. Treats organizations with status `SPONSORED` as Pro.
5. Updates the organization `plan` and `status` from the active subscription when they differ, unless the organization is sponsored.

Current server-side gates include:

- Creating more than `appConfig.menuLimit` menus on Basic.
- Duplicating menus beyond the Basic menu limit.
- Creating more than `appConfig.itemLimit` products on Basic.
- Bulk item creation that would exceed the Basic item limit.
- AI menu import.
- AI menu translation.

Keep new Pro-only creation or generation features gated in the server action, even if the client already opens `UpgradeDialog`.

## Sponsored organizations

Set `Organization.status` to `SPONSORED` to grant Pro features without a Stripe subscription. Sponsored organizations render a sponsored Pro notice in the billing settings page and do not need a Customer Portal session.

Use this state intentionally for manual comped accounts. Do not overwrite it from Stripe sync code.

## Operational notes and pitfalls

- Billing UI is feature-flagged with `FLAGS_ENABLE_SUBSCRIPTIONS=1`; the server-side Pro gates still run regardless of whether the billing settings UI is visible.
- The subscription cache uses short private cache lifetimes and tags. When changing organization membership, active organization, or billing state, revalidate the relevant `organization-*-subscription` and `subscription-current` tags.
- `getBaseUrl()` prefers `BETTER_AUTH_URL`, then production `https://biztro.co`, then Vercel preview URLs, then localhost. Wrong values can produce incorrect Stripe return URLs and invitation links.
- Better Auth stores subscriptions in the Prisma `Subscription` model. The organization relationship is by `referenceId`, not by a Prisma relation.
- Checkout uses the active organization from the Better Auth client. If Checkout opens without a `referenceId`, verify that the active organization cookie/session is set.
- Stripe webhook events must reach `/api/auth/stripe/webhook`; otherwise subscription rows and entitlement checks will not update after Checkout.

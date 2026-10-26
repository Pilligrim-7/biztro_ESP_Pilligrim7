# Billing and Subscription Workflow

Biztro uses the Better Auth Stripe plugin for organization-scoped billing.
The active organization is the subscription reference, and server actions use
that subscription state to gate Pro-only creation and AI workflows.

## Source Map

- `src/lib/auth.ts` configures Better Auth, the organization plugin, and the
  Stripe subscription plugin.
- `src/lib/auth-client.ts` enables the Better Auth organization and Stripe
  client plugins.
- `src/app/api/auth/[...all]/route.ts` exposes the Better Auth route handlers,
  including the Stripe webhook endpoint used by the CLI script.
- `src/app/dashboard/settings/billing/*` renders the billing settings page,
  checkout CTA, Pro status view, and Stripe customer portal button.
- `src/server/actions/subscriptions/*` wraps subscription reads and portal
  creation.
- `src/server/actions/user/queries.ts` exposes `isProMember()`, the server-side
  source of truth for Pro entitlements.
- `src/lib/types/billing.ts` contains plan names, statuses, display prices, and
  Basic-plan limit error codes.

## Configuration

Set the Stripe and subscription variables before enabling billing:

| Variable                               | Used by                                                      | Notes                                       |
| -------------------------------------- | ------------------------------------------------------------ | ------------------------------------------- |
| `STRIPE_SECRET_KEY`                    | `src/lib/auth.ts`, `src/env.mjs`                             | Server-side Stripe client key.              |
| `STRIPE_WEBHOOK_SECRET`                | `src/lib/auth.ts`, `src/env.mjs`                             | Webhook signing secret.                     |
| `NEXT_PUBLIC_STRIPE_PRICE_PRO_MONTHLY` | `src/lib/auth.ts`, `src/lib/types/billing.ts`, `src/env.mjs` | Monthly Pro price id.                       |
| `NEXT_PUBLIC_STRIPE_PRICE_PRO_YEARLY`  | `src/lib/auth.ts`, `src/lib/types/billing.ts`, `src/env.mjs` | Yearly Pro price id.                        |
| `FLAGS_ENABLE_SUBSCRIPTIONS`           | `src/flags.ts`                                               | Set to `1` to show the billing settings UI. |

`src/lib/auth.ts` also reads `NEXT_PUBLIC_STRIPE_PRICE_BASIC` for the Basic
plan entry. That variable is not currently validated by `src/env.mjs`, so check
the plan catalog manually when changing Stripe price ids.

The Stripe client intentionally uses `apiVersion: Stripe.API_VERSION`, which
keeps the configured API version aligned with the installed `stripe` package.

## Local Webhook Workflow

1. Start the app and database in separate terminals:

   ```sh
   bun run db:dev
   bun run dev
   ```

2. Forward Stripe webhooks to Better Auth:

   ```sh
   bun run stripe:listen
   ```

   The script forwards to `localhost:3000/api/auth/stripe/webhook`.

3. Copy the webhook signing secret printed by the Stripe CLI into
   `STRIPE_WEBHOOK_SECRET`.

4. Keep `FLAGS_ENABLE_SUBSCRIPTIONS=1` set when testing the dashboard billing
   page at `/dashboard/settings/billing`.

## Checkout and Portal Flow

- Only organization owners see the billing UI when subscriptions are enabled.
  The page checks the feature flag, current membership role, current
  organization, item count, and `isProMember()` in parallel.
- Basic organizations can start checkout from
  `src/app/dashboard/settings/billing/basic-plan-view.tsx`.
  The client calls `authClient.subscription.upgrade()` with:
  - `plan: Plan.PRO`
  - `annual: true` for yearly billing
  - `referenceId: activeOrganization.id`
  - success and cancel URLs set back to `/dashboard/settings/billing`
- Pro organizations use
  `src/server/actions/subscriptions/mutations.ts` to create a Stripe customer
  portal session and redirect there.
- Checkout failures and portal failures are captured in Sentry with
  `billing-checkout` and `billing-portal` tags.

## Entitlement Rules

Use `isProMember()` for server-side entitlement checks. It returns `true` when
either:

- Better Auth reports an active or trialing subscription whose plan is `PRO`.
- The organization status is `SPONSORED`.

When the subscription plan/status differs from the organization record,
`isProMember()` updates the organization through Better Auth unless the
organization is sponsored.

Current Pro-gated behavior:

- Basic organizations are limited by `appConfig.menuLimit` and
  `appConfig.itemLimit`.
- `createMenu()` and `duplicateMenu()` block Basic organizations at the menu
  limit.
- `createItem()` and `bulkCreateItems()` block Basic organizations at the item
  limit.
- Full-menu import from a file requires Pro.
- AI menu translation requires Pro and `AI_GATEWAY_API_KEY`.
- Team member invitations are shown as a Pro upgrade flow for non-Pro
  organizations.

Editing existing menus, items, categories, and themes is not generally blocked
by the Pro check. Keep that behavior unless product requirements explicitly say
downgraded organizations should lose edit access to data they already created.

## Organization and Public Menu Constraints

- Public subdomain menus are served only when the organization status is
  `ACTIVE`, `TRIALING`, or `SPONSORED`
  (`src/server/actions/menu/queries.ts`).
- Organization deletion is blocked while the current subscription is `active`
  or `trialing`.
- Sponsored organizations skip Stripe subscription display and show a sponsored
  Pro notice on the billing page.

## Cache and Revalidation Notes

Subscription reads use private cache tags:

- `organization-${organizationId}-subscription`
- `subscription-current`

Portal creation and organization deletion update those tags. If a local webhook
test appears stale, confirm that Stripe delivered the webhook, then re-trigger a
read or invalidate the organization subscription tag during debugging.

## Common Pitfalls

- The feature flag key is spelled `enable-suscriptions` in `src/flags.ts`, but
  the environment variable is `FLAGS_ENABLE_SUBSCRIPTIONS`.
- Do not trust client plan state for authorization. Gate Pro-only mutations in
  server actions with `isProMember()`.
- Keep `Tiers` display pricing in `src/lib/types/billing.ts` aligned with the
  Stripe price ids configured in the environment.
- If a Pro checkout succeeds but limits still apply, inspect
  `Subscription.referenceId`; it must match the organization id passed as
  `referenceId` during checkout.

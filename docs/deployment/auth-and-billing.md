# Auth and Billing Deployment Runbook

Biztro uses Better Auth for authentication, organization membership, and Stripe-backed subscriptions. This runbook documents the production assumptions and developer workflows that are wired through the current source code.

## Source Map

- `src/lib/auth.ts` configures Better Auth, Google OAuth, organization support, waitlist enforcement, invitations, and the Stripe plugin.
- `src/app/api/auth/[...all]/route.ts` exposes the Better Auth Next.js route handlers.
- `src/lib/auth-client.ts` configures the client-side Better Auth organization and Stripe plugins.
- `src/lib/utils.ts` owns `getBaseUrl()` and invitation email URL generation.
- `src/server/actions/user/queries.ts` resolves the current organization, membership, permissions, waitlist status, and Pro entitlement.
- `src/server/actions/subscriptions/*` reads active subscriptions and opens the Stripe billing portal.
- `src/server/actions/organization/mutations.ts` creates organizations, joins the waitlist, and blocks organization deletion while a subscription is active.
- `src/env.mjs` validates the required runtime variables.

## Runtime Architecture

1. Requests to `/api/auth/*` are handled by `toNextJsHandler(auth)`.
2. Better Auth uses the Prisma adapter with the SQLite/LibSQL schema in `prisma/schema.prisma` and `prisma/models/auth.prisma`.
3. The organization plugin stores Biztro-specific organization fields through Better Auth additional fields: `description`, `banner`, `status`, `plan`, and `updatedAt`.
4. The Stripe plugin creates customers on signup and manages organization-scoped subscriptions by `referenceId`.
5. Server components and actions call `auth.api.*` with `headers()` so Better Auth can read the request session and active organization.

## URL and Origin Rules

`getBaseUrl()` is the source of truth for generated application links:

1. `BETTER_AUTH_URL`, when set.
2. `https://biztro.co` when `NEXT_PUBLIC_VERCEL_ENV=production`.
3. `https://${NEXT_PUBLIC_VERCEL_URL}` when `NEXT_PUBLIC_VERCEL_ENV=preview` and `NEXT_PUBLIC_VERCEL_URL` exists.
4. `http://localhost:3000` otherwise.

Keep these aligned:

- Better Auth `baseURL` reads `BETTER_AUTH_URL`.
- Trusted origins include `BETTER_AUTH_URL`, `https://biztro.co`, `https://preview.biztro.co`, valid Vercel preview URLs, and localhost.
- Organization invitation emails call `getBaseUrl()` before building `/invite/{id}` links.
- The Stripe billing portal action returns users to `${getBaseUrl()}/dashboard/settings/billing`.

Do not point `BETTER_AUTH_URL` at a tenant wildcard host. Tenant pages such as `https://slug.biztro.co` are public menu routes, while auth flows are expected to use the app host.

## Required Environment Variables

Validated in `src/env.mjs`:

- Database: `TURSO_DATABASE_URL`, `TURSO_AUTH_TOKEN`
- Auth: `AUTH_SECRET` in production, `BETTER_AUTH_SECRET` in production, `BETTER_AUTH_URL`, `AUTH_GOOGLE_ID`, `AUTH_GOOGLE_SECRET`
- Email: `RESEND_API_KEY`
- Stripe: `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, `NEXT_PUBLIC_STRIPE_PRICE_PRO_MONTHLY`, `NEXT_PUBLIC_STRIPE_PRICE_PRO_YEARLY`
- Assets: `R2_ACCOUNT_ID`, `R2_ACCESS_KEY_ID`, `R2_SECRET_KEY_ID`, `R2_BUCKET_NAME`, `R2_CUSTOM_DOMAIN`
- Analytics/AI: `NEXT_PUBLIC_POSTHOG_KEY`, optional `POSTHOG_HOST`, optional `POSTHOG_API_KEY`, optional `AI_GATEWAY_API_KEY`

Also verify `NEXT_PUBLIC_STRIPE_PRICE_BASIC` if the Basic plan is expected to be registered with Stripe. The Stripe plugin currently reads it directly from `process.env`.

## Invite-Only Signup Flow

The marketing waitlist form calls `joinWaitlist`, which creates a `Waitlist` row with `enabled=false` by default.

Account creation is blocked unless the normalized email has an enabled waitlist row:

1. `databaseHooks.user.create.before` extracts an email from the Better Auth payload or context.
2. `isWaitlistEnabled(email)` checks for a matching `Waitlist` row with `enabled=true`.
3. If the row is missing or disabled, the hook throws `AccessDenied`.
4. The Better Auth `before` middleware maps `unable_to_create_user` errors to `/auth-error?type=access_denied`.

Operational constraints:

- Adding someone to the waitlist is not enough to grant access; set `Waitlist.enabled` to `true`.
- Email comparison is normalized to lowercase before the lookup.
- If Better Auth cannot expose an email in the create hook context, the hook does not enforce the waitlist check for that attempt.

## Organization Session Flow

When a session is created, `databaseHooks.session.create.before` looks up the user's first membership via Prisma and stores that organization id as `activeOrganizationId`.

Organization creation goes through Better Auth server APIs:

- `bootstrapOrg` creates an organization and immediately sets it as active.
- `createOrg` creates an organization after checking slug availability.
- Both default new organizations to `status=ACTIVE`, `plan=BASIC`, and an empty `banner`.

Cache tags updated after organization mutations include the organizations list, current organization, membership, permissions, page settings, and subscription tags. If new organization-facing server reads are added, give them a cache tag that can be invalidated by these flows.

## Stripe Subscription Flow

The Stripe client in `src/lib/auth.ts` uses `Stripe.API_VERSION` so the SDK controls the API version instead of pinning an outdated literal.

Configured plans:

- `BASIC`
  - Price id: `NEXT_PUBLIC_STRIPE_PRICE_BASIC`
  - Limits: `appConfig.menuLimit` and `appConfig.itemLimit`
- `PRO`
  - Monthly price id: `NEXT_PUBLIC_STRIPE_PRICE_PRO_MONTHLY`
  - Yearly price id: `NEXT_PUBLIC_STRIPE_PRICE_PRO_YEARLY`
  - Limits: 100 menus and 1000 products
  - Trial: 30 days

Subscription management is organization-scoped:

- `authorizeReference` permits only organization `owner` or `admin` members to manage billing for a `referenceId`.
- `getCurrentSubscription(organizationId)` returns the first active or trialing subscription for the organization.
- `createStripePortal(referenceId)` opens the billing portal and refreshes subscription cache tags.
- `deleteOrganization` refuses to delete an organization with an active or trialing subscription.

## Pro Entitlements

Use `isProMember()` as the server-side source of truth for Pro-only actions.

Current checks include:

- Menu creation and restoration limits.
- Menu item creation and bulk import limits.
- Product translation and auto-translation actions.
- Menu import flows.

`isProMember()` returns true when:

- Better Auth lists an active or trialing subscription whose plan is `PRO`, or
- the organization status is `SPONSORED`.

If the subscription plan/status differs from the organization record, `isProMember()` attempts to update the organization through Better Auth. Errors are logged and captured with Sentry, but the entitlement check continues from the subscription response.

## Local Developer Workflow

Install and run with Bun:

```sh
bun install
bunx prisma generate
bun run db:dev
bun run dev
```

Forward Stripe webhooks to Better Auth while testing subscriptions:

```sh
bun run stripe:listen
```

The script forwards to:

```text
localhost:3000/api/auth/stripe/webhook
```

## Troubleshooting

### OAuth or invite links point at the wrong host

- Check `BETTER_AUTH_URL` first; it overrides every other `getBaseUrl()` branch.
- On Vercel previews, verify `NEXT_PUBLIC_VERCEL_ENV=preview` and `NEXT_PUBLIC_VERCEL_URL` is present.
- Make sure `BETTER_AUTH_URL` is also included in trusted origins by leaving it set consistently across app and auth configuration.

### New users cannot sign in

- Confirm a `Waitlist` row exists for the email.
- Confirm `Waitlist.enabled=true`.
- Confirm the OAuth provider returns an email in the Better Auth user creation payload/context.

### Billing portal returns null

- Check Sentry events tagged `section=billing-portal`.
- Verify the current user is an `owner` or `admin` of the organization reference id.
- Verify `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, and price ids are set in the runtime environment.

### Pro UI and server behavior disagree

- Trust server action results over client state.
- Re-check `isProMember()` and `getCurrentSubscription()` for the organization.
- If the organization has `status=SPONSORED`, Pro entitlement is expected even without an active Stripe subscription.

### Organization deletion is blocked

- This is expected while `getCurrentSubscription(id)` returns an active or trialing subscription.
- Cancel or move the subscription in Stripe before deleting the organization.

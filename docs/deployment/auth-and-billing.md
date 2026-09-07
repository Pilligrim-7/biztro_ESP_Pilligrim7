# Auth and Billing Deployment Runbook

Biztro uses Better Auth for sessions, organizations, invitations, and Stripe
subscription billing. The auth API is exposed through the Next.js catch-all
route at `src/app/api/auth/[...all]/route.ts`, which delegates `GET` and `POST`
requests to `toNextJsHandler(auth)`.

## Runtime architecture

- `src/lib/auth.ts` defines the server-side Better Auth instance.
- `src/lib/auth-client.ts` creates the client auth API with the organization and
  Stripe subscription client plugins.
- `src/proxy.ts` protects `/dashboard` by checking the Better Auth session
  cookie before the route is served.
- `src/server/actions/user/*` wraps organization membership, invitations,
  permissions, and Pro entitlement checks.
- `src/server/actions/subscriptions/*` wraps Stripe customer portal access and
  subscription reads.

## Required environment variables

The validated source of truth for environment variables is `src/env.mjs`.
For auth and billing deployments, configure these validated variables:

- `BETTER_AUTH_SECRET` - required in production.
- `BETTER_AUTH_URL` - canonical app URL used by Better Auth and by
  `getBaseUrl()`.
- `AUTH_GOOGLE_ID` and `AUTH_GOOGLE_SECRET` - Google OAuth provider
  credentials.
- `RESEND_API_KEY` - sends organization invitation email.
- `STRIPE_SECRET_KEY` - initializes the server-side Stripe client.
- `STRIPE_WEBHOOK_SECRET` - verifies Stripe webhooks handled by Better Auth.
- `NEXT_PUBLIC_STRIPE_PRICE_PRO_MONTHLY` and
  `NEXT_PUBLIC_STRIPE_PRICE_PRO_YEARLY` - read by the auth server and billing
  UI for Pro upgrades.

`src/lib/auth.ts` also reads `NEXT_PUBLIC_STRIPE_PRICE_BASIC` directly for the
BASIC Better Auth plan, so keep that value configured anywhere the auth server
initializes.

`getBaseUrl()` in `src/lib/utils.ts` resolves URLs in this order:

1. `BETTER_AUTH_URL`, when set.
2. `https://biztro.co` for production Vercel deployments.
3. `https://$NEXT_PUBLIC_VERCEL_URL` for preview Vercel deployments.
4. `http://localhost:3000` locally.

Keep `BETTER_AUTH_URL` pointed at the public canonical origin in production so
invitation links and billing portal return URLs are stable.

## Trusted origins

`src/lib/auth.ts` allows these origins:

- `BETTER_AUTH_URL`, if present.
- `https://biztro.co`.
- `https://preview.biztro.co`.
- `https://$NEXT_PUBLIC_VERCEL_URL`, only when the value matches
  `*.vercel.app`.
- `http://localhost:3000`.

When adding another deployed host, add it to `trustedOrigins` before expecting
OAuth redirects, session cookies, or Better Auth client calls to work there.

## Invite-only signup gate

New users are gated through the Better Auth `databaseHooks.user.create.before`
hook in `src/lib/auth.ts`.

The hook extracts an email from the creation payload or Better Auth context and
checks `isWaitlistEnabled(email)` in `src/server/actions/user/queries.ts`.
If the email is not enabled in the `waitlist` table, the hook throws
`AccessDenied`. The auth error page at `src/app/(auth)/auth-error/page.tsx`
renders the waitlist signup prompt for `type=access_denied`.

Operational constraints:

- Add or enable waitlist rows before inviting beta users.
- Keep email normalization lowercase; the hook lowercases before checking.
- If signup redirects to the generic error page instead of the waitlist page,
  inspect the Better Auth error query and the redirect handling in
  `src/lib/auth.ts`.

## Organizations and invitations

The Better Auth organization plugin is configured in `src/lib/auth.ts`.
Additional Prisma organization fields exposed through the plugin are:

- `description`
- `banner`
- `status`
- `plan`
- `updatedAt`

Organization creation flows in
`src/server/actions/organization/mutations.ts` create ACTIVE BASIC
organizations, keep the new organization active for the creator, and refresh
organization-related cache tags.

Invitations flow through:

1. `inviteMember()` in `src/server/actions/user/mutations.ts`.
2. Better Auth `createInvitation`.
3. `sendOrganizationInvitation()` in `src/lib/utils.ts`, which sends the Resend
   email with an `/invite/:id` link.
4. `src/app/(auth)/invite/[id]/page.tsx`, which asks unauthenticated users to
   sign in and then renders `AcceptInviteCard`.
5. `acceptInvite()` in `src/server/actions/user/mutations.ts`.

If invitations are not arriving, verify `RESEND_API_KEY`, the `noreply@biztro.co`
sender, and Sentry events tagged with `section: "email"`.

## Stripe subscriptions

The Stripe integration is configured through `@better-auth/stripe` in
`src/lib/auth.ts`.

- The server Stripe client uses `apiVersion: Stripe.API_VERSION`, so the Stripe
  API version follows the installed `stripe` package. Review billing behavior
  and webhook compatibility when upgrading the Stripe SDK.
- `createCustomerOnSignUp` is enabled.
- The BASIC plan uses `appConfig.menuLimit` and `appConfig.itemLimit`.
- The PRO plan uses the monthly and yearly Pro price IDs and includes a 30-day
  free trial.
- `authorizeReference` only lets organization owners or admins manage billing
  for that organization.

The local webhook command is:

```sh
bun run stripe:listen
```

It forwards Stripe events to:

```text
http://localhost:3000/api/auth/stripe/webhook
```

## Pro entitlement source of truth

Use `isProMember()` in `src/server/actions/user/queries.ts` for server-side Pro
checks. It:

1. Loads the current Better Auth organization.
2. Lists active subscriptions for the organization reference ID.
3. Treats `active` and `trialing` subscriptions as active billing states.
4. Syncs the organization `plan` and `status` from Stripe when they differ,
   except for `SPONSORED` organizations.
5. Returns true for a PRO active/trialing subscription or a SPONSORED
   organization.

Do not rely on client-side plan labels for enforcement. Creation limits, upload
limits, imports, translations, and billing pages should use the server-side
helpers so downgraded or stale clients cannot bypass Pro gates.

## Troubleshooting checklist

- Dashboard redirects to `/login`: verify Better Auth session cookie handling in
  `src/proxy.ts`.
- OAuth or auth client calls fail only on a new host: add that host to
  `trustedOrigins`.
- Invite links point at the wrong domain: check `BETTER_AUTH_URL` and
  `getBaseUrl()`.
- Billing portal fails: inspect Sentry events tagged
  `section: "billing-portal"` and confirm the user is an owner or admin for the
  `referenceId` organization.
- Checkout fails: inspect Sentry events tagged `section: "billing-checkout"` and
  confirm the Pro price IDs are set.
- Pro gates look stale after a billing change: refresh the organization
  subscription cache tag or revisit the page after the private cache stale
  window.

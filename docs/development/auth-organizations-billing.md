# Auth, Organizations, and Billing

This runbook documents how Biztro wires Better Auth, organization state, and
Stripe subscriptions. Use it when changing login, onboarding, member
management, plan gating, or billing flows.

## Source map

- `src/app/api/auth/[...all]/route.ts` exposes the Better Auth Next.js handler.
- `src/lib/auth.ts` configures Better Auth, Prisma persistence, organization
  support, invitation email delivery, waitlist gating, and Stripe subscriptions.
- `src/lib/auth-client.ts` configures the client plugins for organizations and
  Stripe subscriptions.
- `src/lib/session.ts` provides `getCurrentUser()` for server components and
  server actions.
- `src/lib/safe-actions.ts` defines `authActionClient` and
  `authMemberActionClient`.
- `src/server/actions/user/queries.ts` reads active organization, membership,
  permission, waitlist, and Pro entitlement state.
- `src/server/actions/user/mutations.ts` switches organizations and manages
  invitations/members.
- `src/server/actions/organization/mutations.ts` bootstraps, creates, updates,
  and deletes organizations.
- `src/server/actions/subscriptions/*` reads active subscriptions and creates
  Stripe customer portal sessions.
- `prisma/models/auth.prisma` contains the Better Auth, organization, member,
  invitation, and subscription tables.

## Auth entry points

All auth HTTP traffic flows through:

```ts
// src/app/api/auth/[...all]/route.ts
export const { GET, POST } = toNextJsHandler(auth)
```

The `auth` object is built in `src/lib/auth.ts` with:

- `better-auth/minimal`
- Prisma adapter with SQLite provider
- `nextCookies()`
- `organization()` plugin
- `stripe()` plugin
- Google OAuth
- email/password disabled

The server config keeps the existing NextAuth-shaped schema by mapping Better
Auth field names onto Prisma columns:

- sessions: `expiresAt -> expires`, `token -> sessionToken`
- accounts: `providerId -> provider`, `accountId -> providerAccountId`,
  token fields -> existing snake_case columns

When changing auth models, update both Prisma schema and the field mappings in
`src/lib/auth.ts`.

## Trusted origins and base URL

`trustedOrigins` currently includes:

- `process.env.BETTER_AUTH_URL`
- `https://biztro.co`
- `https://preview.biztro.co`
- the Vercel preview host when `NEXT_PUBLIC_VERCEL_URL` matches
  `*.vercel.app`
- `http://localhost:3000`

`getBaseUrl()` in `src/lib/utils.ts` also prefers `BETTER_AUTH_URL`, then
production `https://biztro.co`, then Vercel preview URLs, then localhost.
Invitation links and Stripe portal return URLs depend on this behavior.

## Invite-only account creation

User creation is gated by a Better Auth database hook:

1. `user.create.before` normalizes an email from the Better Auth payload or
   request context.
2. `isWaitlistEnabled(email)` checks `Waitlist.enabled` in Prisma.
3. If no enabled waitlist row exists, the hook throws `AccessDenied`.
4. The auth `before` middleware maps Better Auth's
   `unable_to_create_user` error to `/auth-error?type=access_denied`.

Important constraints:

- The waitlist check runs at account creation time, not as a global route
  middleware.
- Empty or missing emails are denied because `isWaitlistEnabled()` returns
  `false` without an email.
- The waitlist table is intentionally simple: `email` is unique and `enabled`
  controls access.

## Session active organization

The session create hook sets `activeOrganizationId` when a user already belongs
to an organization:

```ts
const organization = await getActiveOrganization(String(session.userId))
return {
  data: {
    ...session,
    activeOrganizationId: organization?.id
  }
}
```

`getActiveOrganization(userId)` currently returns the first Prisma `Member`
record for the user and includes its organization. This gives existing users an
active organization during session creation, but it is not a user preference
resolver.

When adding organization selection behavior:

- Use `auth.api.setActiveOrganization()` for explicit switches.
- Invalidate affected cache tags after switching.
- Do not assume `getActiveOrganization()` applies ordering or user-specific
  preference beyond the first membership row returned by Prisma.

## Organization bootstrap and onboarding

The first organization flow uses `bootstrapOrg` in
`src/server/actions/organization/mutations.ts`.

Server-side steps:

1. Require an authenticated user via `authActionClient`.
2. Validate input with `orgSchema`.
3. Check slug availability with `auth.api.checkOrganizationSlug()`.
4. Create the organization with Better Auth:
   - `keepCurrentActiveOrganization: true`
   - current `userId`
   - `status: "ACTIVE"`
   - `plan: "BASIC"`
   - empty `banner`
   - `updatedAt` serialized as ISO text for the plugin additional field
5. Set the new organization active with `auth.api.setActiveOrganization()`.
6. Revalidate organization, membership, permission, subscription, and settings
   cache tags.

Client-side `NewOrgForm` then calls:

```ts
await authClient.organization.setActive({
  organizationId: organization.id
})
```

This keeps the client Better Auth state aligned with the server-created
organization before refreshing or redirecting.

## Organization data exposed through Better Auth

The Better Auth organization plugin maps these additional Prisma fields:

- `description`
- `banner`
- `status`
- `plan`
- `updatedAt`

`getCurrentOrganization()` reads `auth.api.getFullOrganization()` and appends
cache-busting query parameters to `logo` and `banner` when those storage keys
are present.

If a new organization scalar should be available through Better Auth APIs,
update:

1. `prisma/models/auth.prisma`
2. the plugin `additionalFields` mapping in `src/lib/auth.ts`
3. any returned TypeScript shape used by onboarding or settings components

## Membership and permission actions

Server actions use `headers()` with Better Auth server APIs. Current actions
include:

- `switchOrganization()` -> `auth.api.setActiveOrganization()`
- `inviteMember()` -> `auth.api.createInvitation()`
- `acceptInvite()` -> `auth.api.acceptInvitation()`
- `removeMember()` -> `auth.api.removeMember()`
- `safeHasPermission()` -> `auth.api.hasPermission()`

Invitation email delivery is configured in the organization plugin through
`sendOrganizationInvitation()`, which uses Resend and the React email template
in `src/emails/invite`.

## Billing and Pro entitlement

Stripe billing is configured through the Better Auth Stripe plugin in
`src/lib/auth.ts`.

Current subscription plans:

- `BASIC`
  - price id: `NEXT_PUBLIC_STRIPE_PRICE_BASIC`
  - limits from `appConfig.menuLimit` and `appConfig.itemLimit`
- `PRO`
  - monthly price id: `NEXT_PUBLIC_STRIPE_PRICE_PRO_MONTHLY`
  - annual price id: `NEXT_PUBLIC_STRIPE_PRICE_PRO_YEARLY`
  - 30-day free trial
  - limits: 100 menus and 1000 products

Only organization `owner` or `admin` members can manage a Stripe subscription
for that organization. The plugin enforces this in `authorizeReference()` by
checking the Prisma `Member` row for the requested organization.

`isProMember()` is the server-side source of truth for Pro gating:

1. Read the current organization.
2. List active subscriptions for `referenceId = organization.id`.
3. Treat `active` and `trialing` subscriptions as active.
4. Sync `organization.plan` and `organization.status` from Stripe when needed,
   unless the organization is `SPONSORED`.
5. Return true for a `PRO` active subscription or a sponsored organization.

Use `isProMember()` in server actions that create or generate Pro-only data.
Client-side plan state is only UX state.

## Billing UI access

`src/app/dashboard/settings/billing/page.tsx` loads the subscription UI only
when:

- `FLAGS_ENABLE_SUBSCRIPTIONS=1` enables the feature flag, and
- the current membership role is `owner`.

The customer portal button calls `createStripePortal(referenceId)`, which
creates a Better Auth billing portal session and returns to
`/dashboard/settings/billing`.

## Required configuration checklist

Auth and billing initialization reads these environment variables:

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
- `FLAGS_ENABLE_SUBSCRIPTIONS=1` to show the billing UI

Database and file storage variables are validated in `src/env.mjs`; keep that
file as the source of truth when adding new required environment variables.

## Local troubleshooting

### Sign-up redirects to access denied

Check the `Waitlist` table. The user's normalized email must exist with
`enabled = true`.

### New organization exists but UI still shows no active organization

Check both sides of the active-org flow:

- `bootstrapOrg` must call `auth.api.setActiveOrganization()`.
- `NewOrgForm` must call `authClient.organization.setActive()`.
- Relevant cache tags must be updated before route refresh.

### Billing page says only owners can access it

Confirm the current Better Auth membership role is `owner`. The page does not
show billing controls to admins or members, even though the Stripe plugin
authorization allows both owner and admin roles to manage subscriptions.

### Stripe portal cannot be created

Verify:

- `STRIPE_SECRET_KEY` and `STRIPE_WEBHOOK_SECRET`
- the organization id passed as `referenceId`
- the current user's membership in that organization
- the return URL generated by `getBaseUrl()`

### Plan does not update after a subscription change

`isProMember()` synchronizes organization plan/status during server reads. Make
sure the billing page or Pro-gated server action calls it after Stripe webhooks
or portal changes, and check the `organization-${id}-subscription` cache tag if
the UI still shows stale subscription data.

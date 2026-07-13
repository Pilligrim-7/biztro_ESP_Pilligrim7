# Public Menu Workflow

This document explains how Biztro menus move from editable draft data to the
public menu served at `https://slug.biztro.co`.

Use it when changing menu publishing, catalog sync, public rendering, or cache
invalidation behavior.

## Source map

| Area                         | Files                                               |
| ---------------------------- | --------------------------------------------------- |
| Menu data model              | `prisma/schema.prisma`, `prisma/models/auth.prisma` |
| Menu queries                 | `src/server/actions/menu/queries.ts`                |
| Menu mutations               | `src/server/actions/menu/mutations.ts`              |
| Catalog-to-menu sync         | `src/server/actions/menu/sync.ts`                   |
| Public menu route            | `src/app/[subdomain]/page.tsx`                      |
| Read-only Craft render       | `src/app/[subdomain]/resolve-editor.tsx`            |
| Host rewrite                 | `src/proxy.ts`                                      |
| Publish UI and QR/copy links | `src/components/menu-editor/menu-publish.tsx`       |
| Public URL helper            | `src/lib/utils.ts`                                  |

## Data model

`Menu` stores two snapshots:

- `serialData`: the draft/editor Craft.js node tree. The editor and
  authenticated preview read this value.
- `publishedData`: the immutable public snapshot captured at publish time.
  The public route prefers this value.

Both snapshots are JSON strings compressed with `lzutf8` and encoded as base64.

Additional published fields mirror the active public snapshot:

- `publishedAt`
- `publishedFontTheme`
- `publishedColorTheme`

`Organization.activeMenuId` points at the menu that should be public for the
organization. A menu can be `DRAFT` or `PUBLISHED`, but only published menus can
be active for public routing.

## Publish and active-menu rules

Publishing is handled by `updateMenuStatus` in
`src/server/actions/menu/mutations.ts`.

When status becomes `PUBLISHED`:

1. The current draft `serialData` is copied to `publishedData`.
2. The current draft theme fields are copied to published theme fields.
3. `publishedAt` is set.
4. `Organization.activeMenuId` is set to the published menu id.
5. Cache tags for the public subdomain, menu detail, and menu list are updated.

When status becomes `DRAFT`:

1. `publishedData` and published theme fields are cleared.
2. `publishedAt` is cleared.
3. `Organization.activeMenuId` is cleared only if it points at that menu.

Other lifecycle actions:

- `setActiveMenu` only accepts menus whose status is `PUBLISHED`.
- `deleteMenu` refuses to delete the current active menu.
- `revertMenuToPublished` copies `publishedData` and published theme fields
  back into the draft fields. It does not publish a new snapshot.
- `createMenu` and `duplicateMenu` enforce the Basic plan menu limit from
  `src/app/config.ts` unless `isProMember()` allows more.

## Public menu resolution

Public menus are resolved in `getActiveMenuByOrganizationSlug(slug)`.

Resolution order:

1. Find the organization by `slug`.
2. Require organization status to be `ACTIVE`, `TRIALING`, or `SPONSORED`.
3. If `activeMenuId` is set, return that menu only when it belongs to the
   organization and has status `PUBLISHED`.
4. Otherwise, fall back to the latest published menu for the organization,
   ordered by `publishedAt desc`, then `updatedAt desc`.
5. Return `null` when no eligible organization or published menu exists.

This fallback keeps a public menu available if the active pointer is missing,
but it should not be treated as the preferred state. Publishing or explicitly
setting an active menu should keep `activeMenuId` aligned.

## Public render pipeline

Production host-based routing is documented in
`docs/deployment/subdomain-routing.md`. At the app layer:

1. `src/proxy.ts` rewrites `slug.biztro.co` and `slug.localhost` requests to
   `/{slug}` while reserving infrastructure hosts such as `www` and `preview`.
2. `src/app/[subdomain]/page.tsx` calls
   `getActiveMenuByOrganizationSlug(params.subdomain)`.
3. The route renders `publishedData ?? serialData`.
   - Normally a published menu has `publishedData`.
   - The `serialData` fallback supports older or incomplete published records.
4. The snapshot is decoded from lzutf8/base64 and parsed as Craft.js nodes.
5. `sanitizeCraftNodes` ensures every node has a props object before rendering.
6. `extractMenuDataFromNodes` and `normalizePublicMenuItems` build the searchable
   public item list.
7. `ResolveEditor` renders the Craft.js `Frame` with `enabled={false}`.
8. `PublicMenuTracker` sends the `public_menu_viewed` PostHog event.

The authenticated editor preview at
`src/app/menu-editor/[id]/preview/page.tsx` reads `serialData`, not
`publishedData`. Use it to inspect the draft before publishing.

## Catalog sync

Catalog changes can rehydrate stored Craft.js nodes through
`src/server/actions/menu/sync.ts`.

The sync layer fetches current organization, default location, active
categories/items, featured items, and uncategorized items. It updates these
block props when their source data changes:

- `HeaderBlock`: organization and location
- `CategoryBlock`: category and active category items
- `FeaturedBlock`: featured active items
- `ItemBlock`: uncategorized active item

If a referenced category or item no longer exists, the corresponding block is
removed from the node tree.

Sync targets:

- Draft menus: `serialData` is updated when sync runs.
- Published menus: `publishedData` is updated only when the caller chooses to
  update published menus.

The published-menu preference is stored in
`Organization.metadata.menuSync.updatePublishedOnCatalogChange`.
`executeMenuSyncWithPreference` returns `needsPublishedDecision: true` when no
preference exists and the caller did not pass an explicit choice. The dashboard
uses this to prompt before changing already published menus.

## Cache tags and invalidation

Public menu reads use Next cache tags:

- `subdomain-{slug}`: public route, organization metadata, active menu lookup.
- `menu-{id}`: menu detail and decoded render data.
- `menus-{organizationId}`: dashboard menu list.
- `translations-{organizationId}`: public language availability.

Common invalidation points:

- `updateMenuStatus`: updates `subdomain-{slug}`, `menu-{id}`, and
  `menus-{organizationId}`.
- `setActiveMenu`: updates `subdomain-{slug}`, `menu-{id}`, and
  `menus-{organizationId}`.
- `updateMenuSerialData` and `revertMenuToPublished`: update `menu-{id}`.
- `deleteMenu` and `duplicateMenu`: update `menus-{organizationId}`.
- `rehydrateMenusForOrganization`: updates changed `menu-{id}` tags,
  `subdomain-{slug}`, and `menus-{organizationId}`, then revalidates `/{slug}`.

When adding a new mutation that affects public output, update the relevant
cache tags in the same mutation or helper.

## URLs and local development

`getPublishedMenuUrl(subdomain)` returns:

- Production Biztro hosts: `https://{subdomain}.biztro.co`
- Non-Biztro hosts, including local dev and preview hosts:
  `{baseUrl}/{subdomain}`

The proxy also supports `slug.localhost` by rewriting it to `/{slug}`. This is
useful when testing host-based behavior locally, while the generated publish
link remains path-based unless the base host is `biztro.co`.

## Analytics

The public route mounts `PublicMenuTracker`, which records `public_menu_viewed`
with:

- `organization_id`
- `menu_id`
- `slug`
- `path`
- `hostname`
- `referrer`

Dashboard analytics read this event in `src/server/actions/analytics/queries.ts`.

## Troubleshooting

### Published URL returns 404

Check:

1. Organization has a `slug`.
2. Organization status is `ACTIVE`, `TRIALING`, or `SPONSORED`.
3. At least one menu for the organization has status `PUBLISHED`.
4. `activeMenuId`, if set, points at a published menu for the same organization.
5. The public route has been invalidated with `subdomain-{slug}` after changes.

### Public menu shows older catalog data

Check whether the last catalog mutation updated only drafts. Published menus are
not rehydrated unless the user or stored preference allows updating published
snapshots.

### Editor preview differs from public menu

This is expected after draft edits. Preview reads `serialData`; public reads the
published snapshot. Publish the menu to copy the draft into `publishedData`.

### Static or infrastructure subdomains route as menus

Keep the reserved host list in `src/proxy.ts` aligned with the Cloudflare Worker
reserved list in `docs/deployment/subdomain-routing.md`.

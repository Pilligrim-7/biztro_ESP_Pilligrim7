# Public Menu Publishing and Rendering

This guide documents how a dashboard menu becomes the public menu served at an
organization slug such as `https://example.biztro.co`.

Use this when changing menu publishing, public rendering, cache invalidation, or
the data model fields that decide which menu is visible.

## Codepaths

- `src/components/menu-editor/menu-publish.tsx` sends publish, unpublish, save,
  and revert actions from the editor UI.
- `src/server/actions/menu/mutations.ts` persists menu snapshots and updates the
  organization's active menu.
- `src/server/actions/menu/queries.ts` selects the menu that a public slug should
  render.
- `src/app/[subdomain]/page.tsx` decodes, sanitizes, and renders the public
  menu.
- `src/proxy.ts` rewrites host-based subdomain requests to the same
  `[subdomain]` route.
- `src/server/actions/menu/sync.ts` rehydrates saved menu snapshots after catalog
  changes.

## Data model

`Menu` stores editable and published state separately:

- `serialData` is the current editor snapshot.
- `publishedData` is the snapshot served publicly.
- `publishedAt` marks the last publish time shown on the public page.
- `fontTheme` and `colorTheme` are the current editor themes.
- `publishedFontTheme` and `publishedColorTheme` preserve the themes captured at
  publish time.
- `status` is either `DRAFT` or `PUBLISHED`.

`Organization.activeMenuId` points at the published menu that should be served
for the organization's slug.

## Publishing workflow

Publishing runs through `updateMenuStatus`:

1. The editor serializes Craft.js state and calls the server action.
2. When publishing, the action:
   - sets `status` to `PUBLISHED`,
   - copies `serialData` into `publishedData`,
   - sets `publishedAt`,
   - copies current theme ids into the published theme fields,
   - sets `Organization.activeMenuId` to the published menu id.
3. When unpublishing, the action:
   - sets `status` to `DRAFT`,
   - clears `publishedData`, `publishedAt`, and published theme fields,
   - clears `Organization.activeMenuId` only if it points at that menu.
4. The action invalidates cache tags:
   - `subdomain-${organization.slug}`
   - `menu-${menu.id}`
   - `menus-${organization.id}`

The editor can also save draft-only changes through `updateMenuSerialData`.
Saving draft data updates `serialData` and invalidates `menu-${menu.id}`, but it
does not update `publishedData` or the public page.

## Active menu selection

The public page calls `getActiveMenuByOrganizationSlug(slug)`.

Selection rules:

1. Find the organization by `slug`.
2. Serve nothing unless the organization status is one of:
   - `ACTIVE`
   - `TRIALING`
   - `SPONSORED`
3. If `activeMenuId` is set, return that menu only when it belongs to the
   organization and has `status = PUBLISHED`.
4. If the active menu is missing or no longer published, fall back to the most
   recently published menu for the organization, ordered by `publishedAt` and
   then `updatedAt`.

This fallback keeps older organizations with published menus available even when
`activeMenuId` has not been populated yet.

## Public rendering

`src/app/[subdomain]/page.tsx` renders the menu from the selected menu snapshot:

1. It prefers `publishedData` and only falls back to `serialData` when
   `publishedData` is absent.
2. It decodes the snapshot with `lzutf8` base64 decompression.
3. It parses the decoded JSON as Craft.js nodes.
4. It ensures every node has a `props` object before rendering.
5. It extracts searchable menu items from the sanitized nodes for
   `PublicMenuProvider`.
6. It reads the public page background color from the `Sitio` node and footer
   text color from the `Cabecera` node.
7. It loads available item translations for the organization and passes them to
   `TranslationProvider`.

Invalid, missing, or undecodable snapshots return `notFound()`.

`generateStaticParams()` returns all active organization slugs. When there are no
active organizations, it returns `_placeholder` so static generation still has at
least one path.

## URL generation and routing

`getPublishedMenuUrl(subdomain)` builds links for published menus:

- Production Biztro hosts become `https://<subdomain>.biztro.co`.
- Non-Biztro hosts, including local or preview bases, fall back to
  `<baseUrl>/<subdomain>`.

`src/proxy.ts` supports both:

- host-based public URLs, such as `restaurant.biztro.co`, and
- path-based public URLs, such as `/restaurant`.

Reserved hosts such as `www`, `preview`, `images`, `static`, `mail`,
`pm-bounces`, and `send` are not treated as tenant slugs.

Production Cloudflare and Vercel setup is documented separately in
`docs/deployment/subdomain-routing.md`.

## Snapshot rehydration after catalog changes

`rehydrateMenusForOrganization` in `src/server/actions/menu/sync.ts` updates
saved snapshots after catalog data changes.

- Draft snapshots are updated when `updateDraft` is true.
- Published snapshots are updated when `updatePublished` is true and the menu is
  still `PUBLISHED`.
- Updated menu snapshots invalidate `menu-${menu.id}`.
- Organization-level changes invalidate `subdomain-${organization.slug}` and
  revalidate `/${organization.slug}`.

Use this path when menu items, categories, location data, or organization assets
need to be reflected inside existing Craft.js snapshots.

## Common pitfalls

- A draft save is not a publish. Public pages read `publishedData` when it
  exists.
- A published menu must remain `PUBLISHED` to be selected by `activeMenuId`.
- Organizations with `CANCELED`, `PAST_DUE`, `UNPAID`, `PAUSED`,
  `INCOMPLETE`, or `INCOMPLETE_EXPIRED` status do not serve public menus.
- Cache invalidation must include both the menu tag and the subdomain tag when a
  change affects public output.
- Do not treat reserved infrastructure hosts as tenant slugs; keep the reserved
  list aligned with the Cloudflare Worker documented in the deployment guide.

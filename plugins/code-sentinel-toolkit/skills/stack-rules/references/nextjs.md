# Next.js

Covers Next.js apps (App Router and Pages Router). Load together with `react.md`; this pack
wins on conflict.

## Naming
- Route segments follow the file conventions (`page.tsx`, `layout.tsx`, `route.ts`,
  `loading.tsx`, `error.tsx`, `not-found.tsx`); flag misspelled special files, which Next.js
  silently ignores.
- Server actions verb-led (`createOrder`), in files marked `"use server"`.

## Typing
- Route handler and page props typed (`params`, `searchParams`); request bodies validated
  at runtime before use (schema library), not only typed.
- Environment variables read through one typed, validated module rather than raw
  `process.env.X` scattered across files.
- Universal and `react.md` typing rules apply.

## Error handling
- Route handlers and server actions return an explicit error status or result on failure;
  flag a `catch` that returns 200 or `{ ok: true }`.
- `error.tsx` boundaries for segments that fetch data; `notFound()` / `redirect()` not
  swallowed by a surrounding `try/catch` (they work by throwing).
- Server-side errors logged with request context; client receives a safe message, not the
  stack or internal details.

## Boundaries
- Server/client split: `"use client"` only where interactivity needs it. Flag server-only
  code (secrets, database or SDK clients, filesystem) imported into a client component, and
  secrets exposed through `NEXT_PUBLIC_*` variables.
- Use the `server-only` package (or the repo's equivalent) on modules that must never reach
  the client bundle.
- Data access in server components, route handlers or server actions, not in client
  components calling a data store directly.
- Server actions and route handlers are public endpoints: each one checks auth and input
  itself, regardless of which page calls it.
- Middleware kept light; no heavy data access on every request.
- N+1: sequential `await fetch` in a server component loop (use `Promise.all` or one
  batched call); request waterfalls between nested layouts and pages.

## State and lifecycle
- After a mutation in a server action or route handler, call `revalidatePath` /
  `revalidateTag` (or `router.refresh()` on the client) for every view the mutation makes
  stale.
- `fetch` caching: the cache mode (`cache`, `next.revalidate`, route segment config) matches
  whether the data is per-user. Flag per-user data fetched with a shared cache.
- Hydration mismatches: values that differ between server and client render (`Date.now()`,
  `Math.random()`, `window` checks during render).
- `react.md` state and lifecycle rules apply to client components.

## Testing
- Unit and component tests as in `react.md`. Server actions and route handlers tested as
  functions or through a request, including the unauthenticated and invalid-input paths.
- End-to-end (Playwright or Cypress) for critical journeys such as sign-in and checkout.
- Coverage minimum read as in `react.md`; in a monorepo, the app's own config first.

## PR size exclusions
`.next/**`, `next-env.d.ts`, `out/**`, generated route type files, plus `react.md`
exclusions.

## Sensitive paths
`middleware.ts`, auth routes and callbacks, server actions that write data, route handlers
under `api/`, anything reading secrets or cookies, `next.config.*` headers, rewrites and
redirects.

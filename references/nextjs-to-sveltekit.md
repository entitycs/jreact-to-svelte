# Next.js → SvelteKit

Use this when the user comes from Next.js (App Router or Pages Router) or asks about SSR/SSG/streaming/routing/data-loading/server-actions/API-routes. SvelteKit's model is **load-before-render on the server**, which maps cleanly to Next's App Router data model and replaces most of what React devs do with client `useEffect` fetching.

The biggest mental shift: **SvelteKit co-locates routing, data loading, server logic, and UI in one folder** (`src/routes/whatever/+file.ts`). Next separates these into `page.tsx`, `layout.tsx`, `route.ts`, server actions. SvelteKit unifies them.

---

## Routing — file conventions

| Next.js (App Router) | SvelteKit | Notes |
|---|---|---|
| `app/page.tsx` | `src/routes/+page.svelte` | root page |
| `app/blog/page.tsx` | `src/routes/blog/+page.svelte` | `/blog` |
| `app/blog/[slug]/page.tsx` | `src/routes/blog/[slug]/+page.svelte` | dynamic segment |
| `app/blog/[...rest]/page.tsx` | `src/routes/blog/[[...rest]]/+page.svelte` or `[...rest]` | catch-all; `[[...rest]]` = optional catch-all |
| `app/(marketing)/about/page.tsx` | `src/routes/(marketing)/about/+page.svelte` | route group — **parentheses don't add a URL segment**, same as Next |
| `app/@parallel/page.tsx` (parallel route) | no direct equivalent | use layout slots/snippets or components instead |
| `app/dashboard/layout.tsx` | `src/routes/dashboard/+layout.svelte` | nested layout, wraps all children |
| root layout | `src/routes/+layout.svelte` | app shell; children render at `{@render children()}` (Svelte 5) / `<slot />` (Svelte 4) |

A route folder can contain any subset of these `+`-prefixed files:

| File | Purpose |
|---|---|
| `+page.svelte` | the page UI (component) |
| `+page.ts` | page **load** function, runs in browser + server (universal) — for data derived from params/query that's safe to run client-side |
| `+page.server.ts` | page load that must run **on the server only** (DB, secrets) — **also where form actions live** |
| `+layout.svelte` / `+layout.ts` / `+layout.server.ts` | same idea for the layout |
| `+error.svelte` | error UI for this route subtree |
| `+loading.svelte` | shown during navigation while load runs (SvelteKit 2.x) |
| `+server.ts` | **API endpoint** — exports `GET`, `POST`, `PATCH`, `DELETE`, etc. |

Read params/query inside `load` from its `params` / `url` arguments; in a component via `page.params` / `page.url`.

---

## Data loading — `load` replaces fetching-in-component

This is the key translation. **Stop fetching in components with `useEffect`.** Put data loading in a `load` function.

### Next Pages Router `getServerSideProps` / `getStaticProps`

```ts
// Next, pages/blog/[slug].tsx
export async function getServerSideProps({ params }) {
  const post = await getPost(params.slug);
  return { props: { post } };
}
export default function Page({ post }) { return <Article post={post} />; }
```

```ts
// SvelteKit, src/routes/blog/[slug]/+page.server.ts
import type { PageServerLoad } from './$types';
export const load: PageServerLoad = async ({ params }) => {
  const post = await getPost(params.slug);
  return { post };
};
```

```svelte
<!-- src/routes/blog/[slug]/+page.svelte -->
<script lang="ts">
  let { data } = $props(); // { post }
</script>
<Article post={data.post} />
```

`+page.server.ts` = always server (like `getServerSideProps`).
`+page.ts` = isomorphic (runs on server during SSR, on client during client-side nav) — like the data layer Next's App Router evolved toward.

### Next App Router `async` server component

```tsx
// Next
export default async function Page({ params }) {
  const post = await getPost(params.slug);
  return <Article post={post} />;
}
```

Same SvelteKit `+page.server.ts` + `+page.svelte` as above. The `load` function is the async server component; the `.svelte` file is the client UI that receives its output as `data` props.

### Client-side fetching (when you really need it)

If data must load after mount (user interaction, browser-only API), `onMount(() => fetch(...))` still works — but prefer `load` + `invalidate`/`invalidateAll` for refreshable server data.

---

## Mutations — form actions replace server actions

| Next | SvelteKit |
|---|---|
| server action (`'use server'` function called from a client component) | **form action** in `+page.server.ts` |
| `<form action={fn}>` | `<form method="POST">` + `?/actionName` (named) or default |

```ts
// src/routes/login/+page.server.ts
import type { Actions } from './$types';
export const actions: Actions = {
  default: async ({ request }) => {
    const data = await request.formData();
    const email = String(data.get('email'));
    // ...validate, create session, etc.
    return { success: true }; // or throw redirect(303, '/dashboard')
  }
};
```

```svelte
<!-- src/routes/login/+page.svelte -->
<script lang="ts">
  import { enhance } from '$app/forms';
  let { form } = $props(); // the action's return value, available after submit
  let loading = $state(false);
</script>

<form method="POST" use:enhance={() => {
  loading = true;
  return async ({ update }) => { await update(); loading = false; };
}}>
  <input name="email" type="email" />
  <button disabled={loading}>Log in</button>
</form>
```

Key wins vs Next server actions:
- **Progressive enhancement built in.** The form works with JS disabled — it's a real POST to the route. `use:enhance` (from `$app/forms`) progressively upgrades it to an AJAX submit. No equivalent default in Next.
- Named actions via `?/login`, `?/register` in the `action` attribute.
- The `form` prop in the page gives you the action's return value reactively for showing success/errors.

---

## API endpoints — `+server.ts` replaces `route.ts`

```ts
// Next: app/api/users/route.ts
export async function GET() {
  return Response.json(await getUsers());
}
```

```ts
// SvelteKit: src/routes/api/users/+server.ts
import { json } from '@sveltejs/kit';
import type { RequestHandler } from './$types';

export const GET: RequestHandler = async ({ url }) => {
  return json(await getUsers());
};
```

Export `GET`, `POST`, `PATCH`, `PUT`, `DELETE`, `OPTIONS`, `HEAD`. Same fetch-based `Request`/`Response` ergonomics as Next route handlers.

---

## Navigation & client routing

| Next | SvelteKit |
|---|---|
| `<Link href="/x">` | `<a href="/x">` (plain anchor — SvelteKit intercepts same-app links) |
| `useRouter().push('/x')` / `router.push` | `goto('/x')` from `$app/navigation` |
| `useRouter().replace` | `goto('/x', { replaceState: true })` |
| `useRouter().refresh()` | `invalidateAll()` (re-run all `load`s) or `invalidate(url/filter)` |
| `usePathname()` | `page.url.pathname` (from `$app/state`) or `$page.url.pathname` (legacy `$app/stores`) |
| `useSearchParams()` | `page.url.searchParams` |
| `useParams()` | `page.params` |
| route segment as prop | `let { data } = $props()` — `data` carries the merged load output |

### The `$app/stores` → `$app/state` migration

Older SvelteKit code uses `$page` as a store (auto-subscribed with `$`). SvelteKit 2.12+ ships `$app/state` exposing `page` as a plain reactive object (no `$` prefix needed). Use `$app/state` in new code; recognize `$app/stores` in old code.

---

## SSR, SSG, prerendering

| Next | SvelteKit |
|---|---|
| `'use client'` / `'use server'` directive boundary | **no directive boundary** — file location decides: `.svelte` UI runs both sides; `+server.ts` is server-only; `.server.ts` filename suffix forces server-only |
| server component | the `load` function (server) + the `.svelte` (shared) |
| client component (uses state/effects) | a `.svelte` using `$state`/`$effect` — fine, it still SSRs and hydrates |
| `'use client'`-only lib | guard with `import { browser } from '$app/environment'` or `onMount` |
| `generateStaticParams` (static dynamic routes) | `export const prerender = true;` in `+page.server.ts`/`+page.ts`, or `entries` in `+page.server.ts` for dynamic prerendered routes |
| `export const dynamic = 'force-static'` | `export const prerender = true` |
| `export const revalidate = 60` (ISR) | `export const prerender = true` + adapters; ISR-equivalent via `invalidate` on demand. (SvelteKit's prerendering is build-time; for scheduled revalidation use a cron + adapter or external cache.) |
| `output: 'export'` (static export) | `adapter-static` + `prerender = true` |
| full dynamic SSR | default (with `adapter-node` or your hosting adapter) |

Per-route and per-layout config goes in `+page.server.ts`/`+page.ts`/`+layout.server.ts` as exports: `prerender`, `ssr`, `csr`, `trailingSlash`, `entries`.

---

## Streaming & loading states

| Next | SvelteKit |
|---|---|
| `loading.tsx` | `+loading.svelte` (shown during navigation while `load` runs) |
| Suspense streaming a promise | return a `Promise` from `load` and consume with `{#await promise}...{:then data}...{:catch e}...{/await}` — it streams |
| React `Suspense` fallback | `{#await}` block, or a top-level `+loading.svelte` for route transitions |

Streaming example:

```ts
// +page.server.ts
export const load = async () => {
  return {
    slow: getSlowData() // NOT awaited — streams to the client
  };
};
```

```svelte
{#await data.slow}
  <Spinner />
{:then value}
  <Result {value} />
{/await}
```

The slow data arrives after the initial HTML; the `{#await}` block swaps in the result. No Suspense boundary, no client `useEffect` to track loading.

---

## Error handling

| Next | SvelteKit |
|---|---|
| `error.tsx` (route error UI) | `+error.svelte` in the route folder |
| `not-found.tsx` | throw `error(404, '...')` or `new Error(...)`; render via `+error.svelte`, or a root `src/routes/+error.svelte` |
| `global-error.tsx` | root `src/routes/+error.svelte` + `src/error.html` for errors outside the app shell |
| throwing in a server component | throw `error(status, message)` (from `@sveltejs/kit`) in a `load` or action — surfaces as `page.status`/`page.error` in `+error.svelte` |
| typed errors | `error()` is always serialized; for rich error types, use `createErrorBag` or return error data from actions |

`+error.svelte` receives `let { page } = $props()` and reads `page.status` / `page.error.message`.

---

## Middleware & cross-request hooks

| Next | SvelteKit |
|---|---|
| `middleware.ts` (`export function middleware`) | `src/hooks.server.ts` exporting `handle` (server, every request) and/or `src/hooks.client.ts` |
| `next()` / `NextResponse.next()` | `return await resolve(event)` (the next handler) |
| `NextResponse.redirect()` | `redirect(303, '/x')` |
| `NextResponse.rewrites` config | dynamic logic inside `handle` |
| setting request-scoped data for downstream | `event.locals.x = ...` then read in `load` via `locals` |

```ts
// src/hooks.server.ts
import type { Handle } from '@sveltejs/kit';
export const handle: Handle = async ({ event, resolve }) => {
  event.locals.user = await getUser(event);
  const response = await resolve(event);
  response.headers.set('x-custom', 'value');
  return response;
};
```

---

## Project structure & config quick map

| Next | SvelteKit |
|---|---|
| `next.config.js` | `svelte.config.js` + `vite.config.ts` |
| `app/` directory | `src/routes/` (configurable) |
| `public/` | `static/` |
| `package.json` script `next dev` | `vite dev` (SvelteKit CLI) |
| `next build` / `next start` | `vite build` then run via an adapter (e.g. `adapter-node`, `adapter-auto`) |
| middleware edge runtime | adapters / per-route `export const runtime` is not a SvelteKit concept; deployment target is chosen by adapter |
| env vars `NEXT_PUBLIC_*` | `PUBLIC_` prefix for client-exposed vars (`import.meta.env.PUBLIC_*`); Vite env handling |
| `app/` route groups, parallel, intercepting | route groups only; parallel/intercepting routes have no equivalent — model with layouts + components |

---

## RSC / "server component" mental model

React Server Components distinguish server-only components from client components by file directive. SvelteKit doesn't have a server/client component *split* — instead:

- **Server-only logic lives in `load`/actions/`+server.ts`** (plain `.ts` files, never shipped to client if `.server.ts`).
- **`.svelte` files are universal** — they SSR on the server and hydrate on the client. There's no "mark this as a client component." You can use `$state`/`$effect` freely; the reactive bits hydrate, the rest is static HTML.
- **`+page.server.ts` / `+*.server.ts` filename suffix** guarantees server-only — the bundler excludes it from the client build.

So a Next dev's mental model "this component is a server component, that one is a client component" becomes "this *data fetch* is in the server load, and the *UI* is shared and hydrates." Cleaner separation: data on the server, markup everywhere.

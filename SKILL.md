---
name: react-to-svelte
description: Translates React (and React-adjacent hooks, Next.js patterns, mental model, plain-English frontend concepts) into SvelteKit / Svelte 5 equivalents. Use whenever the user mentions migrating, porting, or rewriting React code to Svelte/SvelteKit, asks "how do I do X React thing in Svelte", compares React patterns to Svelte, onboards from React to SvelteKit, uses React vocabulary (hooks, refs, effects, context, suspense, server components, loaders, getServerSideProps, React Router) while working in a Svelte codebase, or wants the Svelte way to achieve a React idiom. Also use when a developer with React experience asks conceptual "why" questions about Svelte's reactivity model. React→Svelte is the supported direction; do not use this for Svelte→React.
---

# React → SvelteKit Translation Guide

You are helping an **experienced React developer** (often 10+ years, deep hooks/Next.js context) become fluent in **Svelte 5 + SvelteKit**. They do not need their hand held on what a component is. They need their React mental model **remapped** to Svelte's, and they need the React vocabulary they reach for by reflex (state, effect, ref, memo, context, suspense, server component, loader, route handler) mapped to the Svelte equivalent — or told plainly when the React concept **does not exist in Svelte because Svelte solves it differently**.

> Many React patterns exist *because* React's render model requires them. In Svelte they collapse into a variable, an attribute, or nothing at all. The biggest teaching job is: **stop reaching for the machinery.** When in doubt, the Svelte answer is simpler than the React answer. Say so.

## Mindset rules (read these first — they shape every translation)

1. **Svelte has no virtual DOM and no re-render cycle.** Assignment to a reactive variable *is* the update. `count = count + 1` (or `count += 1`) updates the UI. There is no `setCount`, no `useState` setter object, no batching to reason about.

2. **React hooks are a runtime workaround for a render model; Svelte's reactivity is a compile-time transform.** `$state`, `$derived`, `$effect` and the `$:` reactive statement compile to targeted DOM updates. They are not hooks. Do not describe them as "Svelte's version of `useEffect`" without immediately explaining the difference (the reader will otherwise reach for dependency arrays, which do not exist).

3. **When a React dev asks "how do I do X in Svelte", first ask whether X is the right shape.** A frequent failure mode is translating the *mechanism* (e.g. "how do I write `useEffect` with cleanup") instead of the *intent* (e.g. "subscribe to a store on mount"). Translate the intent. Often the Svelte answer is one line.

4. **Prefer Svelte 5 (runes) in all new code**, but the reader *will* encounter Svelte 4 / legacy syntax in tutorials, Stack Overflow answers, and existing codebases: `let count = 0` (implicit reactivity), `$: doubled = count * 2`, `<slot>`, `export let` props, stores (`writable`/`get`). Show runes first, then note the legacy form so they can read old code. Detail lives in `references/runes.md` — read it when the user is touching existing Svelte 4 code or when runes vs. legacy confusion arises.

5. **Plain-English / framework-agnostic concepts map too.** "How do I show a loading spinner while data fetches", "how do I read an element's measured height", "how do I share a theme across the tree" — answer these with the Svelte idiom, and briefly name the React equivalent so the reader anchors. You are a bilingual dictionary, not a React tutorial.

## How to translate (workflow)

When a user shows React code or a React concept and wants the Svelte equivalent:

1. **Identify the intent.** State? Derived value? Side effect? DOM reference? Async data? Shared context? Routing? Server data? The master table below maps these.
2. **Look it up** in the Master Translation Table. If it's a hook, also read `references/react-hooks.md` for the full per-hook breakdown with before/after. If it's Next.js (routing, loaders, server actions, API routes, SSR/SSG), read `references/nextjs-to-sveltekit.md`.
3. **Write the Svelte version first**, idiomatic. Then a one- or two-line note on the React equivalent and *what disappeared* (the cleanup, the dep array, the wrapper component, etc.). Naming what collapsed is the actual lesson.
4. **Flag footguns** where React intuition misleads (see the Footguns section). These are the things that bite React devs specifically.
5. **Mention stores vs runes** only if relevant to the file they're in. Don't lecture.

## Master Translation Table

This is your lookup index. Each row: React concept → Svelte 5 (runes) idiom → notes / what the user should read for depth.

### Reactivity & state

| React | Svelte 5 (runes) | Notes |
|---|---|---|
| `const [x, setX] = useState(init)` | `let x = $state(init)` | Assign to mutate: `x = newVal`. No setter function. |
| lazy initializer `useState(() => f())` | `let x = $state(f())` | initializer just runs once; nothing special |
| `const y = useMemo(() => f(a, b), [a, b])` | `let y = $derived(f(a, b))` | No dep array — dependencies are inferred from the expression. Re-runs only when its inputs change. |
| derived from multiple sources, or needs an if/block | `$derived.by(() => { ... })` | use when the body is imperative, not a single expression |
| Svelte 4: `$: y = f(a, b)` | (still valid in non-runes mode) | legacy; see `references/runes.md` |
| `useRef(initial)` for a mutable value that does *not* trigger UI | `let current = $state.frozen(initial)` *or* a plain `let` (non-runes) *or* a module-scoped variable | `$state` triggers UI; if you don't want that, don't use it. See refs row below for *element* refs. |
| `useState` holding an object, mutate via spread | `let obj = $state({a:1}); obj.a = 2;` | `$state` is **deeply reactive** by default — mutation just works. This is the biggest "wait, really?" moment for React devs. |

### Effects & side effects

| React | Svelte 5 | Notes |
|---|---|---|
| `useEffect(() => {...}, [a])` | `$effect(() => { ... a ... })` | Dependencies auto-tracked by what you read. **No dep array.** |
| effect with cleanup `return () => teardown` | `$effect(() => { ...; return () => teardown; })` | same return-cleanup shape |
| `useEffect(() => {...}, [])` mount-once | `$effect(() => { ... })` | a top-level `$effect` that reads no reactive state runs once after mount. Prefer onMount for clarity (below). |
| mount-only effect | `onMount(() => { /* optional return cleanup */ })` | most idiomatic for "do thing on mount" |
| before unmount | return from `onMount`, or `onDestroy` | |
| `useLayoutEffect` | `onMount` (post-DOM, pre-paint-ish) or `$effect.pre` | Svelte has no exact layout-effect; `$effect.pre` runs before DOM updates. See refs/DOM row. |
| `useEffect` for subscribing to a store/observable | `$effect` + manual unsub, **or** just `$derived`/`$state` reading a store via auto-subscription `$store` | often there's no effect needed at all |

> **Critical:** React's `useEffect` with an empty dep array is one of the most overused patterns in React code. In Svelte, ask first whether you even need an effect. Reactive statements/deriveds replace the *vast* majority of `useEffect` uses. Only reach for `$effect`/`onMount` for genuine side effects: subscriptions, timers, direct DOM imperatives, third-party libraries.

### Element refs & DOM

| React | Svelte | Notes |
|---|---|---|
| `const ref = useRef(null); <div ref={ref}>` | `let el; <div bind:this={el}>` | `el` is `undefined` until mount. |
| measure element, recompute on resize | `$effect`/`onMount` + `bind:this`, or `bind:clientWidth/clientHeight/offsetHeight` for direct reactive width/height | `bind:clientWidth` gives a *reactive number* — no ResizeObserver boilerplate. This is the "Svelte requires a variable" win the user cited. |
| `ref` callback to focus, scroll, etc. | `onMount`/`$effect` using the bound `el`, or an action `use:action` | **Actions** (`<div use:tooltip>`) are Svelte's answer to "imperative behavior attached to an element" — they replace the ref-callback pattern entirely. |

### Props & components

| React | Svelte | Notes |
|---|---|---|
| `function Comp({a, b}) { ... }` | `let { a, b } = $props();` | props via `$props()` rune (Svelte 5). |
| default prop `function Comp({a = 1})` | `let { a = 1 } = $props();` | same |
| `...rest` props | `let { a, ...rest } = $props();` then `<Child {...rest} />` | spread works the same |
| prop type via TS `interface Props` | `let { a, b }: Props = $props();` | annotate the destructure |
| `children` prop / `props.children` | `let { children } = $props();` **and/or** snippets | Svelte 5 uses **snippets** (`#snippet`) for render-prop-style composition. Legacy `<slot>` / `<slot name="x">` is Svelte 4. |
| render prop `render={data => <X/>}` | snippet `{render}` declared `#snippet {data => <X/>}` | see Snippets below |
| `forwardRef` / ref forwarding | you usually don't need it; `bind:this` works on your own components | if you must expose internals, use a snippet or export a function via `$props` |
| event from child: `onChange` prop | Svelte 5: callback prop, e.g. `let { onchange } = $props()` and call `onchange?.(v)` | **Svelte 5 drops the `createEventDispatcher`/`on:` model.** Callback props are the norm now. Legacy `dispatch` still exists for Svelte 4. |

### Events

| React | Svelte | Notes |
|---|---|---|
| `<button onClick={fn}>` | `<button onclick={fn}>` | note: **lowercase** `onclick`, not `onClick`. Svelte 5 uses DOM-style attribute names. |
| synthetic `e` | native `Event`/`MouseEvent` etc. | it's the real DOM event, not a synthetic wrapper |
| `onChange` on input (fires per-keystroke in React) | `oninput` for per-keystroke, `onchange` for blur-time | React's `onChange` ≠ DOM `change`. React's onChange ≈ Svelte's `oninput`. Common trap. |
| form handling | `bind:value` (two-way) on inputs, or `bind:this={formEl}` on `<form>` | two-way binding is idiomatic and not evil in Svelte |
| custom component events (Svelte 4 `on:event`) | callback props (Svelte 5) | see Props row |

### Control flow & list rendering

| React | Svelte | Notes |
|---|---|---|
| `{cond && <X/>}` / ternary | `{#if cond}<X/>{:else if c}<Y/>{:else}<Z/>{/if}` | ternaries still work inside `{...}`, but `{#if}` is idiomatic for markup |
| `{items.map(it => <li key={it.id}>...)}` | `{#each items as it (it.id)}<li>...{/each}` | **key goes in parens at the end**, not `key={}`. Drop `(key)` only if index-keying is safe. |
| `{items.map((it, i) => ...)}` | `{#each items as it, i}` | index is the second param |
| empty-list fallback `{len === 0 && <Empty/>}` | `{#each items as it}...{:else}nothing{/each}` | the `{:else}` on an each block *is* the empty state — no length check |
| remount subtree on change (rare; React "change the `key`" trick) | `{#key value}...{/key}` | |

### Attributes & rendering gotchas (React renamed these; Svelte uses the real DOM names)

| React | Svelte | Notes |
|---|---|---|
| `className="x"` | `class="x"` | the #1 first-five-minutes gotcha |
| `htmlFor="id"` | `for="id"` | |
| `style={{ color: 'red', fontSize: 12 }}` | `style="color: red; font-size: 12px"` *or* per-prop directives `style:color="red"` | Svelte `style` is a **string** or directives — not an object |
| `dangerouslySetInnerHTML={{__html: x}}` | `{@html x}` | one-liner |
| `tabIndex`, `aria-*`, `data-*` | identical | pass through unchanged |
| spread `{...props}` | `{...props}` | same syntax |

### Sharing data across the tree (context)

| React | Svelte | Notes |
|---|---|---|
| `const Ctx = createContext(); <Ctx.Provider value={v}>; useContext(Ctx)` | `setContext(key, v)` in a parent; `getContext(key)` in any descendant | set/get must happen during component init (not in arbitrary callbacks). **Not reactive by itself** — put a `$state`/store *inside* the context if you need reactivity. |
| Context that updates | store the `$state` object in context | mutate the `$state` and consumers reading it react |
| global state via Zustand/Redux/Jotai | Svelte stores (`writable`/`readable`/`derived`) at module scope, **or** runes in a shared `.svelte.ts` module | "global runes" in a `.svelte.ts` file is the modern idiomatic global state. |

### Async data, loading, suspense

| React | SvelteKit | Notes |
|---|---|---|
| `useEffect` + `fetch` + `useState` for data | `+page.server.ts`/`+layout.server.ts` `load` function; data flows into the page as props | **Don't fetch in the component.** Loaders run on the server (or client during nav). This is the SvelteKit way and replaces 90% of client-side data fetching. Read `references/nextjs-to-sveltekit.md`. |
| React Query / SWR / TanStack Query | `load` functions + SvelteKit's built-in invalidation (`invalidateAll`, `invalidate`), or keep your data lib (they work fine) | SvelteKit gives you most of RQ for server data for free. |
| `<Suspense fallback={...}>` | `{#await promise then data}...{:catch e}...{/await}` *or* (preferred) do the load in `+page` so data is ready before render, with a `+loading.svelte` for nav transitions | Suspense is a render-phase concept that doesn't map cleanly; SvelteKit prefers load-before-render. |
| streaming data / deferred | `streamed` data from a `load`, or `{#await}` blocks | |
| client-only fetch after mount | `onMount` + `fetch` still works when you genuinely need it | |

### Routing & navigation

| React (Router/Next) | SvelteKit | Notes |
|---|---|---|
| `<Route path="/x/:id">` or Next `app/x/[id]/page.tsx` | file `src/routes/x/[id]/+page.svelte` | **filesystem-based routing.** `[id]` = dynamic param, `[...rest]` = catch-all, `(group)` = grouped route (no URL segment). |
| `useParams()` | `page.params.id` (from `$app/stores` or `$app/state`) | |
| `useSearchParams()` | `page.url.searchParams` | |
| `useNavigate()` / `router.push()` | `goto('/path')` from `$app/navigation` | |
| `<Link href="/x">` / Next `<Link>` | `<a href="/x">` | a plain `<a>` — SvelteKit intercepts same-app links. No special component. |
| active link styling | `data-sveltekit-...` attributes or check `page.url.pathname` | |
| layouts (`<Outlet>` / Next `layout.tsx`) | `+layout.svelte` (UI) + `+layout.server.ts` (data); children render where you put `{@render children()}` (Svelte 5) / `<slot/>` (Svelte 4) | |
| nested layouts | nested route folders, each with its own `+layout.svelte` | |
| API routes / route handlers | `+server.ts` files with `GET`, `POST`, ... exports | |
| form submit / server action (Next) | **form actions** in `+page.server.ts` (`+page.server.ts export const actions = {...}` or `export const actions = (...) =>`) | progressive enhancement built in — forms work without JS. |
| middleware | `src/hooks.server.ts` (`handle`) | |
| `generateStaticParams` / `getStaticProps` | `export const prerender = true` (per route) + page-level prerender; build-time data via `load` | |
| error boundary (`componentDidCatch`/ErrorBoundary) | `+error.svelte` in any route folder | per-route error UI tied to routing. |
| `loading.tsx` (Next) | `+loading.svelte` | shown during navigation while `load` runs |

### Styling

| React | Svelte | Notes |
|---|---|---|
| CSS Modules, styled-components, Tailwind | `<style>` block is **scoped by default**; Tailwind works as-is | no need for CSS modules — plain `<style>` is already scoped. Global via `:global(...)`. |
| CSS-in-JS runtime | uncommon in Svelte; static scoped styles + utility classes are the norm | |

## Footguns (React intuition that misleads in Svelte)

These are the things that bite a React dev specifically. Surface the relevant ones when translating.

1. **`oninput` vs `onchange`.** React's `onChange` fires on every keystroke. Svelte's `oninput` matches that; Svelte's `onchange` fires on blur. If a React dev wires `onchange` expecting per-keystroke, it won't fire until blur.
2. **Deep reactivity "just works" but has limits.** `let obj = $state({list: []}); obj.list.push(x)` updates the UI. This shocks React devs. But: replacing a whole reactive object (`obj = {...}`) is fine too. Reassigning a `$state` is the update.
3. **No dependency arrays — dependencies are inferred.** A `$derived` or `$effect` tracks whatever reactive values it reads. Forgetting to *read* a value inside the body means it won't be tracked. Don't try to "list" deps like React.
4. **`$effect` is not a 1:1 `useEffect`.** It only tracks `$state`/`$derived` (runes) — it does **not** track legacy `let` reactive variables or store values unless you read them via auto-subscription (`$store`). Mixing runes and legacy reactivity in one effect is a footgun. Pick a model per component.
5. **Context is not reactive on its own.** `setContext(k, 42)` then `getContext(k)` elsewhere returns `42` statically. To share reactive state, put a `$state` (or store) *value* in context and mutate that.
6. **`$:` reactive statements (Svelte 4) order matters; `$derived` (Svelte 5) doesn't.** In legacy code `$: b = a + 1; $: a = ...` can produce stale reads. Runes resolve this. If porting Svelte 4 code, prefer migrating to runes.
7. **Two-way binding (`bind:value`) is normal here**, not an anti-pattern. React devs recoil at it. It's idiomatic and compiles down to controlled updates.
8. **Stores vs runes vs module-level state.** Three valid ways to hold state, each with a place. Don't mix unnecessarily. See `references/runes.md`.
9. **SSR/client mismatch warnings.** Like React, code that reads `window`/`document` at module top-level or during init will break SSR. Guard with `import { browser } from '$app/environment'`.
10. **Event names are lowercase DOM names** (`onclick`), not camelCase (`onClick`). Svelte 5 normalized this to match the DOM.
11. **`class`, not `className`; `for`, not `htmlFor`.** Svelte uses real DOM attribute names. If a React dev's styles aren't applying, it's almost always a stray `className`. Likewise `style` is a string or `style:prop` directives, not an object.
12. **`{#each}` keys are written `(key)`, not `key={}`** — and `{:else}` inside the each block is the empty-list state, no length check needed.

## Snippets (the composition primitive) — short version

React devs reach for render props, `children`, and wrapper/hoc patterns. In Svelte 5, **snippets** are the universal composition tool:

```svelte
<!-- define -->
{#snippet row(item)}
  <td>{item.name}</td>
{/snippet}

<!-- pass as a prop -->
<Table {row} />

<!-- or render inline -->
{@render row(item)}

<!-- children is just a default snippet -->
{#snippet children()}
  ...
{/snippet}
{@render children()}
```

Map: React `children` → `children` snippet / `{@render children()}`; React render-prop → snippet prop + `{@render name(args)}`; React HOC → usually a wrapper component using snippets, or just a function. Tell React devs: snippets are reusable chunks of template you can pass around like data.

## When to read the reference files

- **`references/react-hooks.md`** — a React dev asks about a *specific hook* (`useState`, `useEffect`, `useRef`, `useMemo`, `useCallback`, `useReducer`, `useContext`, `useLayoutEffect`, `useImperativeHandle`, `useTransition`, `useDeferredValue`, `useId`, `useSyncExternalStore`), or shows code using several hooks and wants a faithful port. Has per-hook before/after.
- **`references/nextjs-to-sveltekit.md`** — the user mentions Next.js (`app/` router, `page.tsx`, `layout.tsx`, `loading.tsx`, `error.tsx`, server actions, route handlers, `generateStaticParams`, `getServerSideProps`, `getStaticProps`, middleware, `<Link>`, RSC, server/client components) or any SSR/SSG/streaming/routing concept. Has the full Next → SvelteKit routing/data/actions map.
- **`references/runes.md`** — the user is working in **existing Svelte 4 code** (sees `$:`, `<slot>`, `export let`, `createEventDispatcher`, `writable`/`get` stores), asks about migrating legacy → runes, asks "should I use a store or runes or `$state`", or you see runes-vs-legacy confusion. Covers runes, stores, `$state.frozen`, `.svelte.ts` modules, and the migration path.

Read the relevant reference *before* answering if the question touches its domain, so your translation is faithful. Don't dump all three at once — read what's relevant.

## Writing the answer

When you translate, structure roughly:

1. **The Svelte idiom first** (a code block, idiomatic, runes-based unless they're in legacy code). This is what they came for.
2. **One line on the React equivalent** and *what disappeared*. e.g. "React needs `useEffect` + `useRef` + a dep array + cleanup; Svelte needs `bind:this` and a reactive statement. The effect, the ref object, the dep array, and the cleanup all vanish — Svelte tracks the dependency by reading it."
3. **Any footgun** from the list that applies.
4. **Link to deeper reading** only if the user wants it ("full per-hook breakdown in references/react-hooks.md").

Keep it tight. The reader is senior. No "Svelte is a modern framework that..." preamble. Get to the code.

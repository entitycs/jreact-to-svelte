# Runes, Stores, and Legacy Svelte 4

Use this when the user is working in **existing Svelte 4 code** (sees `$:`, `<slot>`, `export let`, `createEventDispatcher`, `writable`/`get`), is migrating legacy → runes, or is confused about which state primitive to pick ("store or runes or `$state`?").

Svelte 5 (released late 2024) introduced **runes** — explicit reactivity primitives that compile to fine-grained updates. Svelte 4 had *implicit* reactivity (`let` assignment) and `$:` reactive statements. Both still work (Svelte 5 supports legacy components in "non-runes mode"), so a React dev will see both flavors in the wild.

---

## The three models, when to use each

1. **Runes** (Svelte 5+) — `$state`, `$derived`, `$effect`, `$props`, `$bindable`. Use for **all new code**. Explicit, deep reactivity, works in `.svelte.ts`/`.svelte.js` modules too (enabling shared/global reactive state without stores).
2. **Stores** (Svelte 3/4/5) — `writable`, `readable`, `derived` from `svelte/store`. Still fully supported and idiomatic for **cross-component/cross-module reactive state** and for wrapping external sources (`useSyncExternalStore` equivalent). Auto-subscribe in markup with the `$` prefix.
3. **Legacy implicit reactivity** (Svelte ≤4) — `let x = 0;` (assignment is reactive), `$: y = f(x)`. Read it in old code; don't write it in new code.

Rule of thumb for new code: **runes by default; a store when you need a subscribable that works outside `.svelte` files or wraps an external source.** Within a single app it's fine to use both.

---

## The translation: Svelte 4 → Svelte 5 runes

| Svelte 4 (legacy) | Svelte 5 (runes) | Notes |
|---|---|---|
| `let count = 0` (reactive on assignment) | `let count = $state(0)` | assignment still updates UI |
| `$: doubled = count * 2` | `let doubled = $derived(count * 2)` | explicit, ordered correctly, no "$: ordering" bugs |
| `$: { side effects }` | `$effect(() => { side effects })` | |
| `export let foo = 1` (props) | `let { foo = 1 } = $props()` | |
| bindable prop `export let foo` + `<Child bind:foo>` | `let { foo = $bindable() } = $props()` | must opt in with `$bindable()` |
| `<slot />` | `{@render children()}` with `let { children } = $props()` | snippets replace slots |
| `<slot name="x" />` | named snippet `{@render x()}` | |
| slot props `<Comp let:item={item}>` | snippet parameters `{@render row(item)}` | |
| `createEventDispatcher` + `dispatch('x', v)` + `on:x` | callback prop `let { onx } = $props()`; call `onx?.(v)`; parent `<Comp onx={fn}>` | **events became callback props** |
| `on:click` / `on:click={fn}` (component events) | `onclick={fn}` for DOM, callback props for components | DOM events: lowercase `onclick` |
| `<input on:input>` | `<input oninput>` | lowercased |
| `{$count}` (store auto-subscribe) | same `$count` syntax still works for stores | unchanged; stores still exist |
| `use:action` | unchanged | actions work the same in Svelte 5 |

---

## Runes reference

### `$state` — reactive state

```svelte
<script>
  let count = $state(0);          // primitive
  let user = $state({ name: '' }); // object — DEEPLY reactive
  user.name = 'Sam';               // updates UI, no spread needed
</script>
```

Deep reactivity: `$state` wraps objects/arrays so nested mutations trigger updates. This is the big surprise for React devs (who must treat state immutably).

`$state.raw` — opt out of deep reactivity (only re-assigns trigger updates, like React's state discipline). Use when you hold large immutable data and don't want proxy overhead.

`$state.frozen` — a mutable, **non-reactive** box. Use for "useRef-like value that shouldn't rerender." Read it like `current.value`.

### `$derived` / `$derived.by` — computed values

```svelte
<script>
  let radius = $state(1);
  let area = $derived(Math.PI * radius ** 2);      // single expression
  let summary = $derived.by(() => {                 // imperative body
    const parts = [];
    if (radius > 0) parts.push(`r=${radius}`);
    return parts.join(', ');
  });
</script>
```

Dependencies are inferred from what's read. No dep array. Re-evaluates lazily and only when inputs change.

### `$effect` / `$effect.pre` — side effects

```svelte
<script>
  let theme = $state('dark');
  $effect(() => {
    document.body.dataset.theme = theme;   // reads theme → tracked
    return () => document.body.removeAttribute('data-theme'); // cleanup
  });
</script>
```

- `$effect` runs after DOM updates.
- `$effect.pre` runs before DOM updates (closer to layout-effect).
- Dependencies auto-tracked from runes reads. **Does not track legacy `$:`/`let` reactivity or raw stores** (only `$store` auto-subscriptions). Mixing models inside one effect is a footgun — pick one reactivity model per effect.
- Return a function for cleanup.
- Prefer `$derived` for computation, `onMount` for mount-once. `$effect` is for genuine side effects.

### `$props` / `$bindable` — component props

```svelte
<script lang="ts">
  interface Props { title: string; count?: number; onsave?: (v: string) => void; }
  let { title, count = 0, onsave }: Props = $props();
  // bindable (two-way from parent):
  let { open = $bindable(false) } = $props();
</script>
```

Default values inline. Callback props replace `createEventDispatcher`. `$bindable()` opts a prop into `bind:open` from the parent.

### Snippets — composition primitive

```svelte
<script>
  let { children, header } = $props();
</script>

{#snippet headerContent()}
  <h1>{@render header()}</h1>
{/snippet}

<header>{@render headerContent()}</header>
<main>{@render children()}</main>

<!-- passing snippets as props -->
<Table rows={data} row={(item) => (
  <!-- can't write JSX here; use #snippet inline -->
)}>
```

Snippets can be defined inline:

```svelte
<Table {data}>
  {#snippet row(item)}
    <tr><td>{item.name}</td></tr>
  {/snippet}
</Table>
```

…and `Table` receives `row` as a prop snippet, rendering it with `{@render row(item)}`. This is the Svelte 5 replacement for slot props / render props.

---

## Stores (still valid, still idiomatic for some cases)

```ts
// stores.svelte.ts  (note: .svelte.ts lets runes live here too)
import { writable, readable, derived } from 'svelte/store';

export const count = writable(0);
count.set(5);
count.update(n => n + 1);
const unsubscribe = count.subscribe(v => console.log(v));
```

Auto-subscribe in any `.svelte` markup:

```svelte
<script>
  import { count } from './stores.svelte.ts';
</script>
<button onclick={() => count.update(n => n + 1)}>{$count}</button>
```

The `$count` syntax subscribes on mount, unsubscribes on destroy, and reads the current value. This is the Svelte-native `useSyncExternalStore`.

### When to pick a store vs runes

- **Single component or component + direct children** → runes (`$state` in the component, pass via props/context).
- **Shared across many components / modules, no clear owner** → either:
  - **Global runes** in a `.svelte.ts` module (modern, idiomatic): `export let count = $state(0);` exported and imported anywhere. The reactivity travels with the import.
  - **A store** (`writable`) — still great, especially if you need subscribe/unsubscribe semantics or are wrapping an external source.
- **Wrapping an external async source** (browser API, third-party observable) → `readable` store with a subscribe function.
- **Derived from multiple stores** → `derived` store; or, if all sources are runes, a `$derived`.

There's no wrong answer — but **don't mix stores and runes in the same component's reactive graph** unless you understand the boundary (`$effect` tracks `$store` auto-subscriptions but not raw store objects).

---

## Global / cross-component state — the modern way

React devs coming from Zustand/Redux/Jotai: the idiomatic Svelte 5 answer is a **runes module**:

```ts
// src/lib/state/auth.svelte.ts
let user = $state<User | null>(null);

export function getUser() { return user; }
export function setUser(u: User | null) { user = u; }
```

Import `getUser`/`setUser` anywhere. Because `$state` lives in a `.svelte.ts` module, all importers share one reactive instance. For convenience you can also `return { get user() { return user; }, setUser }` from a factory. This replaces 95% of "which state library?" questions.

For context-scoped state (per-root-of-tree, not global), use `setContext('auth', authModule)` so the state is isolated to a subtree (useful in tests, storybook, multi-tenant).

---

## Migration: legacy component → runes

When porting a Svelte 4 component to runes:

1. **Opt in to runes** by using *any* rune (e.g. `$props()` or `$state`). Once a component contains a rune, it's in runes mode and legacy implicit reactivity stops working in that file.
2. `let x = 0` → `let x = $state(0)`.
3. `$: y = f(x)` → `let y = $derived(f(x))`.
4. `$: { ... }` side-effect block → `$effect(() => { ... })`.
5. `export let foo` → `let { foo } = $props()`.
6. `createEventDispatcher` + `on:x` → callback prop.
7. `<slot>` / `let:` slot props → snippets.
8. `on:click` (component event) → `onclick` for DOM; callback prop for component events.

Do this per-component; you can have a runes component and a legacy component in the same app. Don't half-migrate a file — a component is either fully runes or fully legacy.

---

## Quick "which should I use?" decision tree

```
Need reactive state?
  ├─ In one component .............. $state
  ├─ Shared app-wide ............... runes in a .svelte.ts module  (modern)
  │                               ... or a writable store  (classic)
  ├─ Provided per-subtree .......... setContext with a $state/store inside
  └─ Wrapping an external source ... readable store

Need a computed value? ............ $derived / $derived.by

Need a side effect? ............... $effect (or onMount for mount-only)

Need a DOM measurement? ........... bind:this + bind:clientWidth/Height

Need a mutable non-reactive box? .. $state.frozen (or plain let)

Reading existing Svelte 4 code? ... recognize $:, <slot>, export let,
                                    createEventDispatcher, on:click,
                                    and $store — they still work, but
                                    write new code in runes.
```

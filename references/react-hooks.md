# Per-Hook Translation: React Hooks → Svelte 5

Use this when a user names a specific hook or shows code built around hooks. Each section: what the hook is *for* (intent), the Svelte idiom, a before/after, and what disappears.

The recurring lesson: **most React hooks exist to work around React's render model. Svelte has no render model to work around, so the hook usually collapses to a variable.**

---

## `useState` → `let x = $state(init)`

Intent: hold reactive component state.

```jsx
// React
function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}
```

```svelte
<!-- Svelte 5 -->
<script>
  let count = $state(0);
</script>

<button onclick={() => count++}>{count}</button>
```

What disappears: the setter function, the tuple destructuring, the functional-updater form. You just assign.

- Object state: `let obj = $state({ a: 1 }); obj.a = 2;` — **mutate directly**, deep reactivity is built in. No spread needed, no immutability discipline.
- Lazy init `useState(() => expensive())`: just `let x = $state(expensive())` — runs once at init, nothing special.
- Lazy init reading props: read props in the initializer normally; `$props()` is available at top of `<script>`.

Legacy note: Svelte 4 is just `let count = 0;` (implicit reactivity on assignment). Runes opt in to deeper, explicit reactivity.

---

## `useEffect` → `$effect`, `onMount`, or usually *nothing*

**Read this carefully — this is the #1 over-translation.** Most `useEffect` uses in React code do not need an effect in Svelte at all. Ask *why* the effect exists:

- **To recompute a derived value** → use `$derived`. Not an effect.
- **To sync state to another piece of state** → use `$derived`. Not an effect. (This is the most common misuse in React code.)
- **To run on mount** → `onMount`.
- **To subscribe/unsubscribe, set up a timer, integrate an imperative lib** → `$effect` (or `onMount` for mount-only).

### Genuine side effect with cleanup

```jsx
// React
useEffect(() => {
  const id = setInterval(tick, 1000);
  return () => clearInterval(id);
}, []);
```

```svelte
<!-- Svelte 5 -->
<script>
  import { onMount } from 'svelte';
  onMount(() => {
    const id = setInterval(tick, 1000);
    return () => clearInterval(id); // cleanup, same shape
  });
</script>
```

### Effect that reacts to changing deps

```jsx
// React
useEffect(() => {
  log(userId);
}, [userId]);
```

```svelte
<!-- Svelte 5 -->
<script>
  let userId = $state(...);
  $effect(() => {
    log(userId); // deps auto-tracked from what you READ
  });
</script>
```

**No dependency array.** Dependencies are inferred from whatever reactive values you read inside the body. Forgetting to read a value = it won't be tracked. There is no equivalent of "I forgot to add it to the array" — there's no array.

Footgun: `$effect` tracks **runes** (`$state`, `$derived`). It does **not** track legacy `let` reactive variables or raw store objects — only auto-subscribed `$store` reads. Don't mix runes and legacy reactivity inside one effect.

### What disappears

The dependency array, the exhaustive-deps lint rule, the stale-closure dance, the cleanup-only-on-unmount mental model, and most importantly — the *need* for the effect in the first place for derived state.

---

## `useLayoutEffect` → `$effect.pre` or `onMount`

Intent: run after DOM mutations but before the browser paints (measure then sync-mutate).

Svelte has no exact layout-effect. Options:

- **`$effect.pre`** — runs before the DOM is updated (closest to "before paint"). Use when you must read a value and sync-adjust before the user sees it.
- **`onMount`** — runs after the component is first in the DOM. Fine for "measure on mount."
- **`bind:clientWidth` / `bind:offsetHeight`** — if all you want is a measured dimension as a reactive number, **this replaces the entire measure-in-layout-effect pattern.**

```jsx
// React: measure an element's height reactively
function Box() {
  const ref = useRef(null);
  const [h, setH] = useState(0);
  useLayoutEffect(() => {
    if (ref.current) setH(ref.current.offsetHeight);
  });
  // + you'd add a ResizeObserver for real responsiveness
  return <div ref={ref}>...{h}px...</div>;
}
```

```svelte
<!-- Svelte -->
<script>
  let h = $state(0);
</script>

<div bind:clientHeight={h}>
  ...{h}px...
</div>
```

What disappears: the ref, the layout effect, the state, the dep-array reasoning, and (because `bind:clientHeight` is reactive to size changes) the ResizeObserver you'd otherwise hand-write. This is the canonical "Svelte requires a variable" example.

---

## `useRef` → `bind:this`, or `$state.frozen`, or a plain variable

`useRef` does **two different jobs** in React. Disambiguate first.

### Job 1: reference a DOM node

```jsx
const ref = useRef(null);
return <input ref={ref} />;
// later: ref.current.focus()
```

```svelte
<script>
  let input;
  onMount(() => input?.focus());
</script>
<input bind:this={input} />
```

`input` is `undefined` until the element mounts — same lifecycle caveat as `ref.current`.

### Job 2: hold a mutable value that does NOT trigger renders

```jsx
const lastTime = useRef(0);
// mutate freely, no rerender
```

```svelte
<script>
  let lastTime = $state.frozen(0); // mutable, NOT reactive
  // or, even simpler in many cases:
  let lastTime = 0; // plain variable — fine if it's not used in markup
</script>
```

Use `$state.frozen` when you want a box that survives renders but shouldn't trigger UI. Use a plain module-or-component-scoped `let` if the value never appears in markup. **Don't use `$state`** for this — it'd trigger unwanted updates.

What disappears: the `.current` property, the `null` initialization dance, the "is it a DOM ref or a value ref?" overload.

---

## `useMemo` → `$derived`

Intent: recompute only when inputs change.

```jsx
const sorted = useMemo(() => items.toSorted(by), [items, by]);
```

```svelte
let sorted = $derived(items.toSorted(by));
```

No dep array. Inputs inferred from the expression. Use `$derived.by(() => { ...imperative... })` when you need statements, not one expression.

What disappears: the dep array and the "did I list every dependency correctly" anxiety. `$derived` recomputes only when its read inputs change — that's the whole point of `useMemo`, achieved automatically.

When you genuinely *don't* need `useMemo` in React (cheap computation, premature optimization) you also don't need `$derived` — just compute inline.

---

## `useCallback` → usually nothing

Intent: stable function identity across renders (to satisfy child memo / effect deps).

**In Svelte this concern largely does not exist.** There's no render cycle recreating functions, no `React.memo` re-render storm, no effect-dep identity churn. Write a plain function.

```jsx
// React
const handleClick = useCallback(() => doThing(id), [id]);
<Child onClick={handleClick} />
```

```svelte
<!-- Svelte -->
<script>
  function handleClick() { doThing(id); }
</script>
<Child onclick={handleClick} />
```

What disappears: the entire hook. If a React dev reaches for `useCallback` in Svelte, stop them — it's not a thing.

(Edge case: if you store a function in a reactive `$state` and need its identity stable, that's a different, rare problem — solve it then.)

---

## `useReducer` → `$state` + a function, or a store

Intent: complex state with action-based transitions.

```jsx
const [state, dispatch] = useReducer(reducer, init);
```

```svelte
<script>
  let state = $state(init());
  function dispatch(action) { state = reducer(state, action); }
</script>
```

Because `$state` is deeply reactive, you can also mutate `state` in place inside the reducer instead of returning a new object — but keeping the reducer pure and reassigning is fine and more familiar.

For truly global/shared reducer state, put `state` + `dispatch` in a `.svelte.ts` module (runes work in `.svelte.ts`/`.svelte.js` files) and import everywhere.

What disappears: the dispatch stability concern. `dispatch` is just a function; no identity churn.

---

## `useContext` → `getContext` (+ `setContext` in a parent)

Intent: read a value provided by an ancestor without prop-drilling.

```jsx
const theme = useContext(ThemeContext);
```

```svelte
<!-- somewhere ancestor, during init -->
<script>
  import { setContext } from 'svelte';
  setContext('theme', theme); // theme can be a $state for reactivity
</script>

<!-- descendant -->
<script>
  import { getContext } from 'svelte';
  const theme = getContext('theme');
</script>
```

Key differences from React Context:
1. **Keyed by a string/symbol**, not a Context object. Use a symbol exported from a shared module to avoid typos.
2. **`setContext`/`getContext` only work during component initialization** (top-level of `<script>`), not inside callbacks/effects. React lets you `useContext` anywhere a hook can go.
3. **Not reactive by itself.** `getContext('theme')` returns the value as it was when set. For reactive context, **store a `$state` (or a store) in context and mutate that** — consumers reading the `$state` will react.

What disappears: the `.Provider` component wrapping the tree, the "context value identity causes global rerender" problem (Svelte's fine-grained reactivity means only consumers of the specific `$state` field update).

---

## `useImperativeHandle` → usually nothing, or a snippet / exported function

Intent: parent calls `ref.current.method()` on a child to invoke imperative logic.

Svelte generally discourages this — the idiomatic answers are props (state down), callback props (events up), snippets (composition). If you genuinely need it: expose a function on the component's instance via the `<script module>` exports or pass a callback up through props.

```svelte
<!-- Child.svelte -->
<script>
  let { register } = $props();
  function focusMe() { /* ... */ }
  $effect(() => register?.({ focusMe }));
</script>
```

What disappears: `forwardRef` + `useImperativeHandle` + `useRef` on the parent — collapsed into one callback prop. Strongly prefer not needing this at all.

---

## `useTransition` / `useDeferredValue` → defer via `{#await}` or streaming loaders

Intent: keep UI responsive during expensive render or async work.

Svelte's render model doesn't have the synchronous-render-blocking problem React's transitions solve, so there's less need. For async data, SvelteKit's `load` functions run during navigation (with a `+loading.svelte` state), and `{#await}` blocks handle in-component promises. Streaming from a server `load` gives incremental data.

There's no direct `useDeferredValue` equivalent — if you genuinely need to debounce an expensive reactive computation, throttle the input source (e.g. a debounced store) rather than deferring the output.

---

## `useId` → Svelte generates IDs; or `crypto.randomUUID`

Intent: stable unique IDs for SSR-safe `id` attributes (label/htmlFor pairing).

Svelte doesn't ship a dedicated hook for this. Options:
- `crypto.randomUUID()` (browser + Node ≥19 + Bun/Deno) for genuine uniqueness.
- A module-level counter for simple stable IDs within a session.
- For SSR-safety, generate the ID inside a `load` or ensure it's computed identally on server and client (avoid `Math.random()`).

Most of the time React devs reach for `useId`, the actual need is a `<label htmlFor>` pairing — which you can satisfy by putting the `<label>` and `<input>` in the same `<div>` and using implicit association, avoiding IDs entirely.

---

## `useSyncExternalStore` → Svelte stores

Intent: subscribe to an external mutable source (browser API, third-party store) with SSR safety.

This is exactly what **Svelte stores** (`readable`, `writable`) are for:

```js
// a store wrapping window.matchMedia
import { readable } from 'svelte/store';

export const prefersDark = readable(false, (set) => {
  const mq = window.matchMedia('(prefers-color-scheme: dark)');
  set(mq.matches);
  const handler = (e) => set(e.matches);
  mq.addEventListener('change', handler);
  return () => mq.removeEventListener('change', handler);
});
```

```svelte
<script>
  import { prefersDark } from './stores.js';
  let $prefersDark; // auto-subscription via $ prefix
</script>
{$prefersDark ? 'dark' : 'light'}
```

The `$store` auto-subscription in markup is the Svelte-native equivalent — it subscribes on mount, unsubscribes on destroy, and reads the current value. The store's `start`/`stop` function is the subscribe/unsubscribe from `useSyncExternalStore`.

What disappears: the getSnapshot/injecting-server-values boilerplate. Stores handle subscription lifecycles automatically.

---

## Quick "which rune?" cheat sheet

| You want to... | Use |
|---|---|
| hold state that updates the UI | `$state` |
| compute a value from other state | `$derived` / `$derived.by` |
| run a side effect when state changes | `$effect` |
| run once on mount | `onMount` |
| hold a mutable value that does NOT update UI | `$state.frozen` or plain `let` |
| reference a DOM node | `bind:this` |
| declare props | `$props()` |
| read a store reactively in markup | `$store` (auto-subscribe) |

# v-observe

> Read [`CONVENTIONS.md`](./CONVENTIONS.md) first — it owns the TypeScript-first contract, modern-syntax rules, "feels native" quality bar, and `package.json` template that all packages share.

**Status:** Greenfield (folder bootstrapped, no code yet). Replaces archived `v-intersect`, `v-mutate`, `v-resize`.

## TypeScript exports (must-haves)

The package's `dist/vObserve.d.ts` must export every symbol below by name so consumers can import types without recreating them:

```ts
import type {
  ObserveOptions,         // top-level discriminated config
  IntersectConfig,        // intersect: { … }
  ResizeConfig,           // resize: { … }
  MutateConfig,           // mutate: { … }
  IntersectCrossEvent,    // crossed handler payload
  IntersectDirection,     // 'enter-from-above' | 'enter-from-below' | 'leave-to-above' | 'leave-to-below'
  ResizeBracketEvent,     // crossed handler payload
  ResizeOrientation,      // 'portrait' | 'landscape' | 'square'
  MutateEventType,        // 'attr:class' | 'children:added' | 'removed' | …
  MutateEvent,            // diff payload union
  ObserveStateAttribute,  // template literal type for the data attribute
} from 'v-observe'
```

Generic on `MutateEvent<T = HTMLElement>` so consumers parsing dynamic content into a domain type get type-safe `added: T[]`. Default `T = HTMLElement` so most users get the right thing without touching generics.

## Why this exists

VueUse exposes three thin separate wrappers — `vIntersectionObserver`, `vResizeObserver`, `useMutationObserver` — each one a near-1:1 over the native API. A consumer needing all three signals on the same element imports three packages, wires three callbacks, and writes the diff/threshold/debounce logic by hand.

`v-observe` is ONE directive that does all three, but it's not a kitchen-sink. The win comes from the cross-observer integration that's only possible when one directive owns all three signals:

- **Cross-observer gating** — `mutate.gateOnIntersect: true` skips mutation callbacks while the host is off-screen (no wasted DOM-diff work). `resize.gateOnIntersect` ditto.
- **Unified observer pool** — single `IntersectionObserver`, `ResizeObserver`, `MutationObserver` pool keyed by config; 100 directive instances with identical config = 1 of each browser-side observer.
- **Single state attribute** — `data-observe-state` reflects all three signals' current state at once for CSS hooks (no per-observer attribute soup).
- **Diff payloads everywhere** — `from`/`to`/`delta` computed for resize, semantic event types + diffs for mutate, threshold-crossed events with direction for intersect.
- **One install, one import, one mental model.**

If we can't justify a feature against this bar (must FIX a real native-API restriction VueUse leaves to the consumer), we don't ship it.

## API

```vue
<!-- intersect-only -->
<img v-observe="{ intersect: { once: true, on: lazyLoad } }">

<!-- resize-only with breakpoint crossings -->
<div v-observe="{ resize: { breakpoints: { sm: 320, md: 640, lg: 960 }, on: 'crossed', handler: handleBracket } }">

<!-- mutate-only with self-removal detection -->
<div v-observe="{ mutate: { on: 'removed', handler: onSelfRemoved } }">

<!-- combined: visibility-gated mutation observation -->
<div v-observe="{
  intersect: { thresholds: [0.5], crossed: handleVisibility },
  mutate: { on: 'children:added', match: '.item', handler: onAdd, gateOnIntersect: true }
}">
```

## P0 — Bootstrap

- [ ] **Scaffold package** (acceptance: package.json with `name: 'v-observe'`, `version: '0.1.0'`, peerDeps `vue ^3.0.0`, scripts mirror `v-trap-focus` (`tsup` build, `vitest` test). Files: `vObserve.ts`, `vObserve.test.ts`, `tsconfig.json`, `vitest.config.ts`, `playground.html`, `README.md`, `LICENSE` (MIT). Repo layout matches sibling `v-*` directives so subagents working across packages have one mental model. Initial `vObserve.ts` exports a typed directive stub that accepts the discriminated config but does nothing — fills in the shape for the API tests below.)

## P0 — Intersect mode

- [ ] **`once: true` auto-disconnect** (acceptance: handler fires once, observer unobserves the element, WeakMap state cleared. Tests pin: 1 call across multiple scrolls; 0 calls after unmount.)

- [ ] **`thresholds: number[]` with per-threshold `crossed` event** (acceptance: `crossed({ threshold, direction: 'up' | 'down', ratio })` fires once per threshold per crossing. 0.25→0.6 fires for [0.25, 0.5] in order. 0.8→0.3 fires for [0.75, 0.5] reverse with `direction: 'down'`. Equal ratio = hit. Tests pin all transitions.)

- [ ] **Scroll-direction inference** (acceptance: callbacks receive `direction: 'enter-from-above' | 'enter-from-below' | 'leave-to-above' | 'leave-to-below'` from comparing `boundingClientRect.top` against previous tick. First tick uses root center as reference. Tests pin all four directions.)

- [ ] **`stableFor: number` debounced state** (acceptance: handler only fires when intersection state has been stable for X ms. 5 rapid flips within 100ms collapse to 0 calls; settled state at 250ms = 1 call with final ratio. Per-element timer reset on every IO callback. Unmount cancels pending timer.)

- [ ] **Shared observer pool** (acceptance: module-level `Map<configKey, IntersectionObserver>` ref-counted. 100 directives with same config = 1 observer (verified via `vi.spyOn(window, 'IntersectionObserver')`). Final unmount disconnects + removes pool entry. Mixed configs = one observer per unique key.)

## P0 — Resize mode

- [ ] **`breakpoints` mode** (acceptance: array form `[320, 640, 960]` with default labels `<320` / `320-640` / etc. Object form `{ sm: 320, md: 640, lg: 960 }` for custom labels. `on: 'crossed'` fires `{ axis, threshold, direction, from, to }` only on threshold crossings. `axis: 'width' | 'height' | 'both'`. Tests: 100 1px ticks within bracket = 0 calls; jump across two brackets = 2 ordered calls; bidirectional crossings.)

- [ ] **Orientation crossings** (acceptance: `on: 'orientation'` fires only on portrait↔landscape flip. `{ from, to, ratio }` payload. `squareTolerance: number` widens the square band. Tests: continuous resize keeping orientation = 0 calls; flip = 1 call.)

- [ ] **Diff payload everywhere** (acceptance: `{ from: { width, height }, to: { width, height }, delta: { width, height } }` on every resize callback. First tick has `from: null`. Tests pin across debounce/breakpoint windows.)

- [ ] **`box: 'border' | 'content' | 'device-pixel'` shortcut** (acceptance: maps to RO `borderBoxSize` / `contentBoxSize` / `devicePixelContentBoxSize`. Default `'border'`. Falls back to `contentBoxSize × DPR` when `devicePixelContentBoxSize` undefined.)

- [ ] **`debounce: number`** (acceptance: collapses tick storms into one call per window. Coexists with `breakpoints` (debounce wraps breakpoint logic). Unmount cancels.)

- [ ] **Pool ResizeObservers** (acceptance: module-level Map, ref-counted. RO supports multiple targets per observer so 100 directives = 1 observer. Mixed `box` modes = one per mode.)

## P0 — Mutate mode

- [ ] **Semantic event types** (acceptance: `on: 'attr:class' | 'attr:style' | 'attr:*' | 'attr:<name>' | 'children:added' | 'children:removed' | 'text' | 'removed'`. Single string or array. Maps to MO `init` config internally.)

- [ ] **Diff payload — not raw records** (acceptance: handler receives `{ type, name?, from?, to?, added?, removed?, target }`. Directive sets `attributeOldValue: true` automatically so `from` is reliable. Multiple records in one MO callback collapse to one diff per type.)

- [ ] **Self-removal detection (the killer feature)** (acceptance: when `'removed'` is in subscription, directive ALSO observes target's parent and detects target detachment — fires `'removed'` once on `parent.removeChild`, on `parent.innerHTML = ''`, on parent's `children` reset. Documents the one-level-up limit explicitly.)

- [ ] **Selector-filtered events** (acceptance: `match: '.item'` filters `children:added` / `children:removed` to matching nodes only. Multi-match array supported. Non-element nodes always rejected.)

- [ ] **Built-in debounce** (acceptance: `debounce: number` collapses chatty subtrees into one call per window. Unmount cancels.)

- [ ] **Pool MutationObservers** (acceptance: Map keyed by config + target. MO doesn't natively support multiple targets per observer instance — pool here means we don't double-attach when the same target is observed twice with the same config. Less impactful than IO/RO pooling but worth doing for consistency.)

## P0 — Cross-observer integration

- [ ] **`gateOnIntersect`** (acceptance: when set on `resize` or `mutate` config, that observer's callback only fires when `intersect` reports the host as visible (`intersectionRatio > 0`). Implementation: directive caches latest intersect state and gates the resize/mutate fan-outs. Skipped fires don't queue — they drop. Tests: mount off-screen, mutations don't fire; scroll into view, mutations resume; scroll out, mutations stop.)

- [ ] **Unified `data-observe-state` attribute** (acceptance: single attribute reflecting current state across all three observers, format `intersect:visible|hidden;resize:bracket;mutate:active|idle`. Updated atomically. Cleared on `unmounted`. Tests pin format + transitions.)

## P1 — Polish

- [ ] **README differentiation table vs VueUse** (acceptance: side-by-side feature table for all three observers + the cross-observer features. Concrete consumer code per row showing what raw VueUse setup would look like. No FUD.)

- [ ] **Bundle size budget** (acceptance: ESM ≤ 5 KB minified for the full directive (all three observer types + cross-observer integration). Validated in CI. Documents tree-shaking caveats — using only `intersect` config still ships all three observer types because the directive's binding shape is dynamic. Acceptable trade for the unified API.)

- [ ] **Playground covers all three modes + combined** (acceptance: `playground.html` has 4 demo cards: intersect-only (lazy-load), resize-only (responsive layout), mutate-only (DOM diff log), combined (gated mutation). Smoke test mounts each + verifies state attribute reflects expected signals.)

## P2 — Future

- [ ] **`v-observe.intersect` / `v-observe.resize` / `v-observe.mutate` modifiers** (acceptance: Vue directive modifier syntax (`v-observe.intersect`) lets consumers skip the wrapping object when only one mode is needed. `<img v-observe.intersect="{ once: true, on: load }">` is sugar for `<img v-observe="{ intersect: { once: true, on: load } }">`. Multiple modifiers stackable. Tests pin parity with object form.)

- [ ] **Composable `useObserve` for non-template usage** (acceptance: ONLY if there's a real non-template need — first establish the directive form covers 95%+ of cases. Punted until requested.)

## Gotchas

- All three observers fire async (microtask / RAF). The directive's `mounted` runs synchronously; first observer callback may arrive 1+ frames later. Tests must `await nextTick()` + RAF spin before asserting first state.
- `'removed'` self-detection observes target's PARENT — if parent is also removed, the parent observer fires once but our handler doesn't see the target removal there (different mutations). Document: works for one-level removals; if grandparent is removed in one operation, the directive's `unmounted` fires and we don't double-emit `'removed'`. Acceptable: prevents double-firing when Vue itself unmounts the tree.
- Cross-observer gating `gateOnIntersect` requires `intersect` to be configured. If only `resize: { gateOnIntersect: true }` is set, throw a typed error at mount time with a clear message.

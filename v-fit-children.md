# v-fit-children

> Read `CONVENTIONS.md` first — it owns the cross-cutting rules (TypeScript-first, simple stupid
> code, playground coverage). This brief owns only what is specific to this package.

**Status:** Published — 2.1.0 on npm (owner `ozjsey`, 11 versions); **3.0.0 built locally, pending
publish** (Run 20, 2026-08-16). 2.2.0 and 2.3.0 were built but never published; 3.0.0 supersedes
both. 3.0.0 is **breaking** — see the README changelog. Default posture remains **conservative**:
it is one of only three names owned on npm, and the next change needs explicit user sanction.
The package has its own nested git repo with real history and its own `ARCHITECTURE.md` (the
`src/` split is a deliberate anti-copy-paste measure requested by the owner — keep it). Default
posture: **conservative**. It is one of only three names actually owned on npm; changes need
explicit user sanction, and additive-only unless a major bump is deliberately planned here.

## TypeScript exports (the 2.2.0 contract)

```ts
import vFitChildren, {            // default export = the directive
  vFitChildren,                   // named form of the same
  FitChildrenPlugin,              // app.use() registration
  DIRECTIVE_NAME,                 // 'fit-children'
} from 'v-fit-children'
import type {
  FitChildrenOptions,             // generic <T = unknown>, T = the data array item
  FitChildrenEventDetail,         // generic <T = unknown>
  FitChildrenFitState,            // 'fits' | 'overflowing' — the host data-attribute value
} from 'v-fit-children'
```

## The 3.0 engine (Run 20) — read this before touching measurement

The ghost is **gone**. Everything below the next heading is retained as history: it explains what
2.x did and why it was replaced, not what the code does now.

Per recalculation, inside one rAF: `measure.ts` un-hides what the directive hid, releases the
`width`/`max-width`/`flex` it imposed, takes ONE layout read of the real children (width + the
*measured* spacing before each — CSS gap and margins together, whatever their source), and
restores. `fit.ts` (pure) computes the greedy run plus smart fit. `visibility.ts` hides the
overflow and sizes the host to exactly that run.

**The invariant** (also in `ARCHITECTURE.md`): `measure.ts` is the only DOM read, `visibility.ts`
the only DOM write, `fit.ts` is numbers in / numbers out. That is what prevents a second,
disagreeing measurement — the exact defect class that produced partially-visible children for all
of 2.x.

**Hard-won specifics, all found by measuring in a real browser, each after a wrong guess:**

- Read available width **before** unconstraining the host and **after** un-hiding children. Read it
  after unconstraining and a host that is its own container reports `max-content` (everything
  always fits); read it while children are still hidden and a shrink-to-fit host reports its own
  collapsed size and spirals to zero.
- Available width is `min(container content width, host content width)` — wrappers, padding and a
  sibling badge between the two mean the host rarely gets all the container's width.
- Sizing the host needs **all three** of `width`, `max-width` and `flex`. `flex: 1` resolves to
  `flex-basis: 0%`, and basis beats `width` outright for a flex item; `flex-grow` beats `width`;
  `flex-shrink` undoes it. A probe showed inline `141.359px` computing to `134.281px`.
- Add the host's border+padding to the imposed width under `box-sizing: border-box`, or the
  *content* box lands short by exactly that (a constant clip).
- Include the **last visible child's trailing margin** in the total — spacing between children is
  shared, but that one belongs to nobody, and omitting it sizes the host short by exactly it.
- `gap` is a **floor** over measured spacing, never a replacement: margins are already measured, so
  overriding downward under-counts and lets a chip through that does not fit.

**Verify visually, always.** `pnpm smoke` only proves a demo renders without throwing — Run 19
passed it while three demos showed clipped chips. The tool for this is
`scratchpad/cdp.mjs` + `measure.js` (Node 22 built-in WebSocket, no deps): it drives the real
playground, sweeps every width slider, and flags any visible child past the host's content edge or
any `host.scrollWidth > host.clientWidth`. Current state: **9/9 demos clean at 19 widths each,
against sources and against the built dist.**

## History — why the ghost design existed, and what it cost (2.x, REMOVED in 3.0)

The directive hides overflowing children with `display: none !important`; hidden elements measure
0 in place, so the next recalculation cannot measure them where they stand. The answer is a
**ghost**: an invisible fixed-position `div` on `document.body` into which every child is cloned
each recalc (kept children first, `flex-shrink: 0`), sized to `containerWidth − offsetNeededInPx`
(or full width when everything fits — "smart fit"), with an `IntersectionObserver`
(root = ghost, threshold 1.0) deciding which clones fully fit. Real children are then shown/hidden
to match. It measures real layout — gaps, borders, `scrollWidth` overflow — rather than guessing.

Known costs, accepted for now (document, don't silently "fix"):

- **Clone side effects.** `cloneNode(true)` clones `img`/`iframe` (they load network resources
  from the ghost), duplicates `id`s in the document, and upgrades custom elements in the ghost.
- **Lost ancestor CSS context.** Clones sit under `body`, so descendant selectors rooted above the
  host, inherited fonts, ancestor CSS variables and container queries can make a clone measure
  differently than its original. Vue scoped-style attributes DO survive cloning.
- **Latency.** rAF + async IO callback = 1–2 frames between trigger and applied visibility; the
  previous state is visible (clipped, host has `overflow: hidden`) in between.
- **Shrinkable children.** Observable clones keep default `flex-shrink: 1` inside the
  width-constrained ghost; chip-like children resist shrinking, which is why this has not bitten.
  The 2.2.0 total-width path measures at `max-content` and is immune.

## ~~P1 — proposed v3: synchronous in-place measurement~~ — SHIPPED, Run 20 (2026-08-16)

Done, and the predictions held: ~40% of the source deleted, the whole clone-side-effect class
gone, single-frame response, real elements measured in their real CSS context, and the entire
`MockIntersectionObserver` harness dead (the suite was rebuilt around a `getBoundingClientRect`
row simulator plus pure unit tests over `computeFit`). The gate was satisfied by a real-browser
geometry pin captured before the rewrite and replayed after — see the 3.0 section above.

One prediction was wrong in an important way: the rewrite was framed as a cleanup with equivalent
behavior. It was not — 2.x was **visibly broken**, clipping children into partial visibility in
three of nine demos, and the ghost's lost-CSS-context cost was the direct cause. It should have
been routed as a bug fix, not an optional refactor.

## P1 — conventions drift, batch when touched next

- [x] ~~Dual-format build + sourcemaps~~ — done Run 19 (mirrors `v-observe` incl. the
      sourceMappingURL rename fixup; `require()` verified).
- [ ] `mounted`/`unmounted` hook migration (siblings' shape; tests call `beforeMount`/
      `beforeUnmount` directly today). Still open after Run 20 — fold into the next breaking change.
- [ ] `bubbles: true` on `fit-children-updated` (sibling convention; behavior change for delegated
      listeners — v3 at earliest).
- [x] ~~`data-v-fit-decorative` untested and undemoed~~ — done Run 19: README attributes table +
      data-index rule, 3 tests, playground card `08-decorative.vue`.
- [x] ~~`playground.html` broken since 2.0.0~~ — repaired Run 19: `.min.js` import, `sortBySize`/
      `rowCount` demos rewritten to the 2.x surface (`data`/state attribute), verified mounting in
      headless Chrome.
- [ ] `v-fit-children-resources/` (the README's demo-video host) — convert or fold in; tracked as
      root `TASKS.md` P2.
- [x] ~~Margin-aware smart-fit total~~ — fixed in 3.0: spacing is measured from the real layout,
      so margins are counted directly and `gap` became a floor rather than a replacement.

## Gotchas — they're there for a reason (verify before "cleaning up")

- **Consumer-hidden children are skipped by ownership marker**: inline `display: none` WITHOUT
  `data-v-fit-hidden` means the consumer (v-show) hid it — not measured, not shown/hidden, not
  reported, but still consumes its data index. Class-based hiding is NOT detected (documented
  limitation); don't "improve" this with a computed-style read — see the minimal-mock rule below.
- **`MockResizeObserver` records targets and passes entries**: `trigger()` hands the observed
  elements through as entries, and the per-child observer filters out entries whose target carries
  `data-v-fit-hidden` (our own hide/show churn). Tests select observers by their recorded target
  sets — don't revert the mock to a no-op `observe()`, which is what let the 2.x suite stay green
  while the per-child observer did not exist at all.
- **The event dispatches only when the outcome changes** (3.0). Hiding a child is itself a resize,
  so an unconditional dispatch lets a listener that renders from the payload loop forever. The
  signature covers the fit AND the `data` reference, because an unchanged fit over new data still
  changes `hiddenData`.
- **`data-v-fit-w` is gone** (3.0) along with the ghost. Nothing stamps widths onto clones any
  more; `measure.ts` reads the real elements.
- **No width caching, ever.** Commit `984a2ef` ("removed caching in its entirety, it has done more
  damage than good") removed it deliberately; every pass re-measures fresh.
- **Tests call directive hooks directly** with hand-built bindings (no `@vue/test-utils`), and the
  gap tests mock `getComputedStyle` to return ONLY `{ gap, columnGap }` — code running during a
  recalc must not depend on other computed properties. This is why the 2.3.0 `overflowX` read is
  `getComputedStyle(child).overflowX || child.style.overflowX` — the fallback is load-bearing.
- **`beforeMount` (not `mounted`) is safe here** because all measurement is rAF-deferred and the
  `!state.targetElement && wrapperElement` dance compensates; don't "fix" it casually.
- The test file's double-quote + semicolon style is the local convention for that file; sources are
  single-quote, no semicolons. Match each file, don't restyle.

# v-fit-children

> Read `CONVENTIONS.md` first — it owns the cross-cutting rules (TypeScript-first, simple stupid
> code, playground coverage). This brief owns only what is specific to this package.

**FEATURE FREEZE (owner, 2026-09-16): nothing new ships on this package.** No `lines`, no
badge attribute, no priority ranking, no `expanded` — the ideation fleet's four proposals are
all declined. Maintenance only: correctness fixes (FIT-2's P0), and truth fixes to the shipped
CHANGELOG/README (DOC-1). Do not re-propose features here; this line is the answer.

**Status:** Published — **2.3.0 on npm** (owner `ozjsey`, scoped as `@ozjsey/v-fit-children`;
the registry holds 2.2.0 and 2.3.0, `dist-tags.latest = 2.3.0`, published 2026-09-14T10:01:48Z —
verified 2026-09-17). **2.3.1 is built locally and not published.** This line previously said
"2.2.0 on npm … 2.3.0 is built locally and not published", months after 2.3.0 went up; the shipped
2.3.0 tarball said the same thing about itself. Default posture remains **conservative**: it is one of only three names
owned on npm, and the next publish needs explicit user sanction. The package has its own nested git
repo with real history and its own `ARCHITECTURE.md` (the `src/` split is a deliberate
anti-copy-paste measure requested by the owner — keep it).

> **Verify the version against the registry, not against this line.** Four different version
> numbers were live in this repo at once in 2026-09 — `package.json`, the README changelog, this
> brief, and a stray `.tgz` in the package root that was *ahead* of what was published and built
> from a different config. `npm view @ozjsey/v-fit-children versions` costs nothing.

## TypeScript exports — the whole surface

```ts
import vFitChildren, {            // default export = the directive
  vFitChildren,                   // named form of the same
} from '@ozjsey/v-fit-children'
import type {
  FitChildrenOptions,             // generic <T = unknown>, T = the data array item
  FitChildrenEventDetail,         // generic <T = unknown>
  FitChildrenFitState,            // 'fits' | 'overflowing' — the host data-attribute value
} from '@ozjsey/v-fit-children'
```

**That is all of it.** There is **no `FitChildrenPlugin`, no `DIRECTIVE_NAME`, and no
`data-fit-children-state`** — those three were named in this brief and in `CLAUDE.md` for a year and
have never existed in `src/`, in `dist/`, in the README or in any published tarball. `directive.ts`
documents the absence deliberately: registering a directive is the application's decision, not this
package's. `<script setup>` picks up `vFitChildren` by naming convention with no registration at
all, and anyone who wants it global writes one `app.directive()` call under whatever name suits
them. **Do not "restore" them.** The host attribute is `data-v-fit-state`.

## The engine — read this before touching measurement

No ghost. No `IntersectionObserver`. No `requestAnimationFrame`. No cached pass.

Per recalculation, synchronously inside the trigger: `measure.ts` un-hides every child the directive
hid, takes ONE `getBoundingClientRect` pass over the real children (width + the *measured* spacing
before each — CSS gap and margins together, whatever their source — + trailing margin), and records
it. `fit.ts` (pure) computes the greedy run plus smart fit. `visibility.ts` hides the overflow,
stamps `data-v-fit-state`, and dispatches. **`visibility.ts` never touches the host's size**; the
host is measured, never written.

Every caller already runs between layout and paint — a `ResizeObserver` callback does, and the
`updated` hook is a post-render effect in the same task — so deferring to a frame is what used to
make the previous state visible, clipped, in between.

**The invariant** (also in `ARCHITECTURE.md`, and now enforced by `architecture.test.ts` rather than
by prose): `measure.ts` is the only module that reads layout, `visibility.ts` the only one that
writes visibility, `fit.ts` is numbers in / numbers out. That is what prevents a second, disagreeing
measurement — the exact defect class that produced partially-visible children for all of 2.x, and
that came back in 2.2.0 through `observers.ts` storing `ResizeObserver` content rects.

**Hard-won specifics, all found by measuring in a real browser, each after a wrong guess:**

- Available width is `min(container content width, host content width)`, both taken with every child
  shown. **Nothing is subtracted from either** — they are content widths, so border, padding and any
  classic scrollbar are already out. Naming an ancestor as `widthRestrictingContainer` therefore
  only changes the answer when the host can measure *wider* than the space it is given; a flex item
  or a block-level host is already bounded and the option is inert there.
- **A classic scrollbar is not available width.** It is inside the border box `getBoundingClientRect`
  reports and outside `clientWidth`. Leaving it in makes the budget ~15px too generous on Windows
  and Linux and admits one child too many. macOS overlay scrollbars make the correction zero, which
  is exactly how it shipped.
- **A shrinkable flex child must be measured by its `scrollWidth`, not its box.** A flex item whose
  `overflow` is not `visible` has an automatic minimum size of zero (Flexbox §4.5), so with the
  default `flex-shrink: 1` its box is whatever the host had left — our own answer echoed back, and
  `full <= available` becomes true by construction. The playground never revealed this because
  `.pg-chip` is `white-space: nowrap`.
- **The measuring cursor advances by the child's BOX** (`rect.right`), never by a larger
  `scrollWidth`: the next child is laid out against the box, so advancing by the spill clamps the
  gap in front of it to zero and under-bills the row.
- Include the **last visible child's trailing margin** in the total — spacing between children is
  shared, but that one belongs to nobody, and omitting it judges the row short by exactly it.
- `gap` is a **floor** over measured spacing, never a replacement: margins are already measured, so
  overriding downward under-counts and lets a chip through that does not fit. The floor applies from
  the *second* child on — the first child's `spacingBefore` is its distance from the host's content
  edge, not a gap between siblings.
- A **decorative** child left trailing the visible run introduces nothing (`Ada · Grace · Alan ·`),
  so it is trimmed. That can only give the row back width.

**Verify visually, always.** `pnpm smoke` only proves a demo renders without throwing.
`pnpm geometry` measures the rendered layout and now also catches an empty row, over-hiding,
non-monotonicity, a contradicting state attribute and a hidden-set change with no event. `pnpm
interactions` drives the tab in a real Chrome and reads the payload a listener received back out of
the card — and it can be pointed at the **published** artifact for a negative control:

```bash
cd playground
pnpm interactions
PLAYGROUND_UNALIAS=v-fit-children pnpm interactions   # the same checks against npm's latest (2.3.0)
```

## What 2.3.0 fixed, and why none of it was a layout bug

FIT-1 drove the published artifact through 2,193 measurements: **zero half-clipped children, worst
overshoot +0.5px inside its own `EPSILON`, monotonic everywhere.** The row this package computes is
good. Every 2.3.0 fix is about what it *reported* about that row, or what it measured beforehand.
`CHANGELOG.md` has the full list with the evidence; the shape of it:

- The event's dispatch gate watched the **visible** set plus the `data` array's **reference**, while
  every field of the payload is about the **hidden** set. A first pass that hid everything was
  silent (no "+N more" badge on a phone, which is where it matters); `items.push()` / `pop()` never
  re-reported; `isOverflowing` went stale on the resize path while the attribute stayed correct,
  because the attribute write sits above the early return.
- Pinning a child by `data-v-fit-keep` consumed a `data` index and pinning it by `keepVisibleEl` did
  not, so the two documented mechanisms produced different `hiddenData` for the same row.
- The three measurement items above (scrollbar, shrinkable flex child, cursor).

## Gotchas — they're there for a reason (verify before "cleaning up")

- **Consumer-hidden children are skipped by ownership marker**: inline `display: none` WITHOUT
  `data-v-fit-hidden` means the consumer (`v-show`) hid it — not measured, not shown/hidden, not
  reported, but still consumes its data index. Class-based hiding is NOT detected (documented
  limitation); don't "improve" this with a computed-style read.
- **Pinning and `data` membership are unrelated.** Membership is declared by the absence of
  `data-v-fit-decorative` and nothing else. A pinned *control* that is not one of your items needs
  both attributes, or every index after it skews. The directive warns once per distinct mismatch.
- **`MockResizeObserver` records targets and passes entries built from the target's own laid-out
  rect.** Tests select observers by their recorded target sets — don't revert the mock to a no-op
  `observe()`, which is what let the 2.x suite stay green while the per-child observer did not exist
  at all, and don't revert `trigger()` to reporting a module-level `hostWidth` that nothing assigns
  (it made the feedback-loop tests unable to fail).
- **The event dispatches only when the PAYLOAD changes.** Hiding a child is itself a resize, so an
  unconditional dispatch lets a listener that renders from the payload loop forever. The gate
  compares every field against the one last dispatched — not a proxy for them. A proxy is what
  2.2.0 had, and it is the whole of FIT-1 F1, F2 and the quality audit's worst finding.
- **`state.metrics` is a per-pass record, not a cache.** It is overwritten by every pass. There is
  no width caching and no `remeasure` flag any more; both went with the cached pass in 2.3.0.
- **`mounted` / `beforeUnmount` are the hooks** (not `beforeMount`), because measurement is
  synchronous and there is no rAF to compensate with. There is no wrapper element and no ghost.
- **`architecture.test.ts` is the enforcement**, and the two source comments that point at it are
  now telling the truth. Before 2.3.0 a comment in `measure.ts` justified a dead `||` fallback by
  citing gap tests that "mock `getComputedStyle` down to two keys" — no test did, and the branch was
  unreachable. Treat a comment that cites a test as a claim to check, not a reason to keep code.
- The test files' double-quote + semicolon style is the local convention for those files; sources are
  single-quote, no semicolons. Match each file, don't restyle.

## P1 — conventions drift, batch when touched next

- [x] ~~Dual-format build + sourcemaps~~ — done Run 19 (mirrors `v-observe` incl. the
      sourceMappingURL rename fixup; `require()` verified).
- [x] ~~`mounted`/`unmounted` hook migration~~ — the directive uses `mounted` / `updated` /
      `beforeUnmount`. The tests still invoke the hooks directly with hand-built bindings.
- [ ] `bubbles: true` on `fit-children-updated` (sibling convention; behavior change for delegated
      listeners — v3 at earliest).
- [x] ~~`data-v-fit-decorative` untested and undemoed~~ — done Run 19: README attributes table +
      data-index rule, tests, playground card `08-decorative.vue`.
- [ ] `v-fit-children-resources/` (the README's demo-video host) — convert or fold in; tracked as
      root `TASKS.md` P2.
- [x] ~~Margin-aware smart-fit total~~ — fixed in the 3.0 engine work: spacing is measured from the
      real layout, so margins are counted directly and `gap` became a floor.
- [x] ~~No `CHANGELOG.md`~~ — added with 2.3.0. It is a publish blocker under
      `tickets/_STANDARDS.md`, and this was the only package in the portfolio without one.
- [x] ~~No browser spec~~ — `playground/scripts/interactions/v-fit-children.mjs`, 13 checks over 7
      of 11 cards, negative-controlled against the published artifact.

## History — the ghost design, and what it cost (2.x, REMOVED)

Retained because it explains what 2.x did and why it was replaced, not what the code does now.

The directive hides overflowing children so they measure 0 in place, which means the next
recalculation cannot measure them where they stand. The 2.x answer was a **ghost**: an invisible
fixed-position `div` on `document.body` into which every child was cloned each recalc, sized to
`containerWidth − offsetNeededInPx`, with an `IntersectionObserver` deciding which clones fully fit.

Its costs, all real: `cloneNode(true)` loaded `img`/`iframe` resources from the ghost, duplicated
`id`s and upgraded custom elements; clones sat under `body`, so descendant selectors, inherited
fonts, ancestor CSS variables and container queries could make a clone measure differently from its
original; rAF + an async IO callback put 1–2 frames between trigger and applied visibility, with the
previous state visible and clipped in between.

The rewrite deleted ~40% of the source and the entire clone-side-effect class. One prediction was
wrong in an important way: it was framed as a cleanup with equivalent behaviour. It was not — 2.x
was **visibly broken**, clipping children into partial visibility in three of nine demos, and the
ghost's lost-CSS-context cost was the direct cause. It should have been routed as a bug fix, not an
optional refactor.

# v-teleport-to

> Read [`CONVENTIONS.md`](./CONVENTIONS.md) first.

**Status:** Published at v1.0.0 (514 tests across 39 documented runs). The `src/` is mature — modular, tested across two Vue minor versions + SSR. Has its own internal `TASKS.md` / `PROGRESS.md` / `ARCHITECTURE.md` for run-by-run iteration.

## Why a strategic update is needed (v2.0)

The current implementation positions in place via CSS `position: fixed`. That escapes `overflow: hidden` ancestors but **breaks under any transformed/filtered/contained ancestor** — `transform`, `filter`, `perspective`, `contain`, `will-change` all create a containing block for `position: fixed` per CSS spec. Modern apps hit this constantly: animated sidebars, GPU-layered cards, motion-design. The README's escape valve is "use the `useTeleportTo` composable inside `<Teleport to="body">`" — which violates the "feels native" rule (consumer wires the teleport themselves and binds a styles ref).

v2.0 owns the teleport itself. Bare `v-teleport-to="{ to: trigger }"` works regardless of ancestor styles, regardless of `v-for`, regardless of Nuxt SSR — no consumer wrapper, no companion composable, no `<ClientOnly>`.

## TypeScript exports (must-haves)

`dist/vTeleportTo.d.ts` already exports the public surface. v2.0 widens it. After v2.0:

```ts
import type {
  TeleportToOptions,        // includes new fields: teleport?, teleportTarget?
  TeleportToEventDetail,
  TeleportToStyles,
  VirtualReference,
  TeleportToPlacement,      // 'auto' | 'top' | 'bottom' | 'left' | 'right'
  TeleportToStrategy,       // 'fixed' | 'absolute'
  TeleportToOverflow,       // 'shift' | 'hide' | 'none'
  TeleportToCrossAxisAlign, // 'start' | 'center' | 'end'
} from 'v-teleport-to'
```

If any of those aren't currently exported, v2.0 picks them up.

## P0 — v2.0 NATIVE TELEPORT (BREAKING)

These tasks land as a single major-version bump (v1.0.0 → v2.0.0). They go into `v-teleport-to/TASKS.md` as a new top-priority section so the existing `PROGRESS.md` log captures the run history.

- [ ] **Programmatic DOM teleport in `mounted`** — record `{ originalParent, anchorNextSibling }` in the existing `WeakMap` state; call `target.appendChild(el)` (default `target: document.body`; configurable via `teleportTarget?: HTMLElement | string`). All positioning math reads `getBoundingClientRect` of the original ref unchanged — viewport coords are correct because the host is now a direct child of `body`. On `unmounted`, restore via `originalParent.insertBefore(el, anchorNextSibling)` if both still connected.

  Tests pin: bare directive moves element to body (verifiable via `document.body.children` count delta); transformed-ancestor scenario (`<div style="transform:translateZ(0)">`) still escapes correctly; `v-for` of 100 items moves all 100; unmount restores to original anchor; HMR reload sees original DOM tree (because unmount always restores); custom `teleportTarget` HTMLElement; custom `teleportTarget: '#modal-root'` string with `querySelector` lookup; target-not-found falls back to body with a console warn.

- [ ] **`teleport: false` opt-out for legacy/edge cases** — when `teleport: false` is passed, skip the DOM move and apply v1.x in-place `position: fixed` behavior. Useful when (a) host needs to inherit ancestor CSS context (font, scoped styles), (b) teleporting would break a focus trap, (c) consumer is already inside a Vue `<Teleport>`. Default `true` (the breaking change). Tests pin: `teleport: false` keeps el in original parent, math still works, mid-life flip moves/restores correctly.

- [ ] **`<Teleport>` collision detection** — auto-detect when host is already inside a Vue `<Teleport to="body">` (walk up parents until either body or a known teleport sentinel is found, OR check `el.parentElement === document.body` already). Skip own teleport; leave `position: fixed` math in place. Prevents double-teleport. Tests pin: directive inside Vue `<Teleport>` does NOT re-move the element.

- [ ] **SSR contract documented** — Vue 3's directive lifecycle hooks only fire client-side per framework contract; no `<ClientOnly>` wrap needed at consumer side. Extend `vTeleportTo.ssr.test.ts` with one assertion that confirms `mounted` is not invoked in Node env. README adds explicit "SSR / Nuxt" section saying "no `<ClientOnly>` needed".

- [ ] **Migration guide in README + CHANGELOG** — top-level "## v2.0 migration" section explains: (1) why the breaking change (transformed ancestors), (2) what changed (host reparented to body), (3) `teleport: false` escape valve, (4) old `useTeleportTo + <Teleport>` pattern is no longer needed. v1.x → v2.0 side-by-side code example. CHANGELOG.md gains a v2.0 entry.

- [ ] **Real-browser regression test for transformed ancestor** — Playwright/Cypress test mounts directive bound to a ref inside `<div style="transform:translateZ(0)">`. Asserts host's `getBoundingClientRect` tracks ref's viewport coords across scroll. This is the case jsdom can't reproduce — the entire reason for v2.0. CI-only or `npm run test:browser`. Replaces (or supersedes) the existing P7 "real-browser drift smoke" task.

## Constraints / gotchas for the v2.0 implementation

- **The directive's existing `WeakMap<HTMLElement, DirectiveState>` keys by element identity**. After teleport, the host element identity is unchanged — only its parent changes. Existing per-host state still works without modification.
- **Restoration on `unmounted` is best-effort**. If the original parent has been removed (Vue unmounted the parent component first, which is the normal case), don't try to restore — the host element is going to be garbage-collected anyway. Just `stateMap.delete(el)`.
- **HMR**: Vite's HMR can call `unmounted` then `mounted` again on the same physical element when a parent component reloads. The `unmounted` restoration step ensures HMR sees a consistent original DOM tree before the next `mounted` re-teleports. Pin this in a test that simulates HMR by manually calling unmounted + mounted in sequence.
- **Nested directives**: If two `v-teleport-to` instances are bound where one's host is a descendant of another's host, the inner one's teleport-target may have shifted (its host's chain to body changed). Acceptable — both end up at `body` directly, so the chain is irrelevant after teleport. Tests pin nested case.
- **Don't break composable form**. `useTeleportTo` continues to work as-is for consumers who manage their own teleportation (e.g. inside Vue's `<Teleport>` slots they wired manually). Document the directive vs composable choice in v2.0 README.

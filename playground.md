# playground

Cross-package live demo app. Read [`CONVENTIONS.md`](./CONVENTIONS.md) first.

## Identity

`playground/` is a private (never published) Vite + Vue 3 app with **one tab per v-\* library** and
**one card per feature**, where every card's source is the real `.vue` file and can be edited in the
browser while it runs. 7 libraries, 84 demos as of 2026-09-05. pnpm is the package manager.

## Why this exists

Three problems it fixes, none of which the per-package `playground.html` files solve:

1. **No cross-package surface.** Seven packages, no single place to see them working, so "does the
   documented API still work" was only ever answered by unit tests — which mock the browser.
2. **Docs drift silently.** A README recipe that no longer compiles is invisible until a user hits
   it. Here every documented feature has a card, and `pnpm smoke` fails when one stops compiling.
   That is how the stale `v-select-text` README (still documenting `condition`, unaware of
   `enabled` / `trigger` / contenteditable / `useSelectText`) was confirmed — and rewritten in the
   2026-08-23 static-text pass.
3. **No ad-hoc exploration.** Answering "what happens if I set `crossAxisAlign: 'end'` with
   `flip: true`" meant editing a file and rebuilding. Now it is a two-second edit in the page.

## Contract

- **Two alias targets.** Default: every library specifier resolves to its `.ts` source (HMR, no
  stale bundles). `PLAYGROUND_TARGET=dist` (`pnpm dev:dist` / `pnpm smoke:dist`) resolves to each
  package's built `module`/`main` entry instead — the consumer's view, and the fast answer to "is
  this dist stale". The alias key is the package's **npm name** and the folder is its **directory**;
  `vite.config.ts` keeps them as separate columns because conflating them is PG-15, which hid the
  fact that `@ozjsey/v-fit-children` — the one published package — had never been dist-tested.
- **Dist mode has no fallback.** A missing package.json, a missing `module`/`main`, or an entry that
  is not on disk **throws at config load**, naming the package; the dev server does not start. This
  is deliberate: a dist run that quietly tests source is worse than no dist run, because it reports
  green.
- **The compiler is the runtime's own Vue.** Demos are compiled in the page by `src/sfc/*` using
  `vue/compiler-sfc` (npm-pinned to the exact `vue` version) plus `sucrase` for the TypeScript strip
  that esbuild performs in a real Vite app. `src/main.ts` asserts the two versions match and refuses
  to boot otherwise, and `smoke` / `interactions` assert it from outside the page. PG-14: this used
  to be `vue3-sfc-loader`'s bundled 3.4.15 against a 3.5.41 runtime, which compiles a directive
  inside a `v-for` to a non-block vnode — never re-patched, so **`updated` never fired** and no
  option change ever reached a directive on any `v-for` card.
- **Each package's own install path, enforced.** `installLibraries()` calls the plugin the README
  tells users to call. A build that loads without a usable directive — or without the plugin it is
  supposed to export — is a **boot failure**, not a fallback. Degrading quietly here is what let a
  deliberately broken `@ozjsey/v-fit-children` dist render all 97 cards and report `smoke:dist`
  green.
- **Demos are copy-pasteable.** No playground-specific props, helpers or globals are injected into a
  demo. What you read is what you would paste into an app.
- **One Vue instance and one CodeMirror instance** across the playground, the library sources and
  the compiled demos — `resolve.dedupe` + exact aliases + tsconfig `paths` for Vue; direct deps +
  dedupe + a single `optimizeDeps.include` pass for the CodeMirror/Lezer family (a second
  `@codemirror/state` breaks extension `instanceof` checks). The smoke test's `?editors=open` pass
  guards the CodeMirror half permanently.
- **Green means green.** `pnpm smoke`, `pnpm smoke:dist` and `pnpm typecheck` all pass, or the
  change is not done.

## Definition of "covered"

A library's tab is complete when **every option, every binding form, every emitted event, every
`data-*` state and every documented caveat has a card**. Concretely, for each package: walk the
README's options table and exports table top to bottom; anything not reachable from a card is a gap.

Known deliberate exclusions, all because they do not render:
`bigdecimal-string` (pure arithmetic), `dependency-grouper` (CLI over the filesystem),
`vue-provide-seeker` (VS Code extension), `inhouse-agent` (training harness).
`v-trap-focus` was cancelled 2026-08-09 (crowded niche) and its tab removed with it.

## Cross-library cards

Some features only exist in combination — `v-copy`'s history picker is a copy history *plus* a
`v-teleport-to` dropdown, and it is the reason the package exists. The registry is one tab per
library, so the rule is:

> **A cross-library card lives in the tab of the library whose feature it proves; every other
> package it touches goes in `uses`.**

`DemoMeta.uses?: string[]` has three consumers: a distinct chip on the card, the tab's filter
haystack, and the editor hint — which would otherwise claim every import resolves to the tab's own
package, which is false the moment a card imports a second one. Promote to a `recipes/` folder only
if a third such card appears; until then a new tab would duplicate a library's identity.

## Maintenance rule

**A change to a v-\* public API is not finished until its tab is updated.** Adding an option means
adding or extending a card in the same run, in the same way tests and README are updated. The tab
count in `playground/README.md` and the demo count here are part of that update.

## Gotcha the demo author used to hit

**Historic, fixed 2026-09-06 — kept because the demos on disk still avoid it.** The old in-browser
compiler parsed the generated render function as plain JavaScript, so TypeScript syntax inside a
template expression (`dz!.cancel()`, `(e: UploadError) => …`) failed to compile even though a real
Vite app accepts it. `@vitejs/plugin-vue` hands the *whole* module — script and template — to
esbuild with `loader: 'ts'`; `src/sfc/compile.ts` now does the same with sucrase, so the two agree.
Typed handlers still read better in `<script setup>`, but the playground is no longer the reason.

The live gotcha is unchanged and unrelated: **a `ref` written inside a template expression is
unwrapped**, so passing one from the template hands the directive `undefined`. Build options that
carry a `ref` in `<script setup>`.

## Backlog

- [ ] **Per-demo assertions in the smoke test** — largely superseded by `pnpm interactions`, which
      proves the directive *did something* far more thoroughly than a post-render selector check
      could. What is still worth having is the cheap half: a static `assert` per manifest entry so
      the fast smoke pass catches a card that renders but never initialises, without paying for the
      full interaction run. Decide whether that is worth the second mechanism before building it.
- [x] **Interaction coverage** — `pnpm interactions` / `pnpm interactions:dist`, added 2026-08-23.
      A zero-dependency CDP driver (`scripts/interactions.mjs` + `scripts/lib/cdp.mjs`, Node 22's
      built-in `WebSocket`) rather than Playwright, matching `smoke.mjs`'s no-dependency rule. One
      spec per library under `scripts/interactions/`; each check gets a freshly loaded page.
      **`v-dropzone` is done — 48 checks, all 12 cards.** It immediately paid for itself: it found
      a `webkitGetAsEntry`-returns-null bug that made every synthetic drop deliver zero files, and
      confirmed two cards whose every button was a silent no-op under a green smoke.
      **`v-select-text` is done — 23 checks, all 12 cards, green on source and dist.** It is the
      library the harness matters most for: jsdom implements `Range` and `Selection` but has no
      layout, so only a browser can say whether a `display: none` subtree was skipped, whether a
      collapsed whitespace run painted as one space, or whether a click on a token selected the
      token. Two of its checks pin facts no unit test can reach — a `user-select: none` host makes
      a Range that stringifies to `""`, and Chrome mirrors a focused field's own selection into the
      document selection.
      Still to write: `v-observe`, `v-copy`, `v-scroll-into-view`, `v-teleport-to`,
      `v-fit-children`.
- [x] **`v-fit-children`: ship a `FitChildrenPlugin` + `DIRECTIVE_NAME`** — done 2026-08-09
      (v2.2.0, additive): the `INSTALLS` special case is gone, `installLibraries()` treats it like
      every tier-1 sibling.
- [ ] **Link each card to its README section** — acceptance: `DemoMeta` grows an optional `docs`
      anchor; the card header renders a link to `../<package>/README.md#<anchor>`. Makes the
      playground the entry point to the docs rather than a parallel universe.
- [ ] **Deploy it** — acceptance: `pnpm build` output published somewhere linkable (GitHub Pages
      off the eventual repo), so a README can say "try it" and a prospective user can, without
      cloning. Depends on the packages having a remote at all (see root `TASKS.md`).

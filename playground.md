# playground

Cross-package live demo app. Read [`CONVENTIONS.md`](./CONVENTIONS.md) first.

## Identity

`playground/` is a Vite + Vue 3 app with **one tab per published package** and **one card per
feature**, where every card's source is the real `.vue` file and can be edited in the browser while
it runs. 10 libraries, 131 demos as of 2026-09-13. pnpm is the package manager.

It is no longer a private harness: it has its own repository
(`github.com/ozJSey/npm-portfolio-playground`) and deploys to GitHub Pages at
<https://ozjsey.github.io/npm-portfolio-playground/>, which every package README links into by
`#<demo folder name>`. `package.json` stays `"private": true` — that is a Pages deploy, not an npm
publish. **Not every tab is a directive**: `vue-write-behind` is a composable and
`bigdecimal-string` is not even Vue, so neither registers anything — `directiveName: null` and no
`install` in `src/libraries.ts`'s `LIBRARY_SPECS`. Both still appear in `LIBRARY_MODULES`, so a demo
can import them and get the same module instance the app has.

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
  supposed to export — is a **failure**, not a fallback. Degrading quietly here is what let a
  deliberately broken `@ozjsey/v-fit-children` dist render all 97 cards and report `smoke:dist`
  green.
- **One package's failure stops at that package's tab (PG-22).** `src/libraries.ts` loads each
  library through its own `import()` inside its own `try`. Before that it was ten static imports at
  module scope, so a sibling that did not compile blanked **all ten tabs** and `smoke` reported only
  "No library tabs rendered" — naming nothing, and costing an agent part of a run over a defect in a
  package they were not touching. The failure is now painted on every tab, stamped on
  `<html data-playground-library-failures>`, published on `window.__PLAYGROUND_LIBRARY_FAILURES__`,
  and failed by `smoke`, `interactions` and `geometry` **by name**. This is a change of blast radius,
  not of loudness: a compiler/runtime mismatch is still fatal for the whole page. `pnpm typecheck:libs`
  catches the same class in ten seconds without a browser.
- **The instrument has to say why it failed.** Five harness defects in one month were one defect:
  the check failed in a way that did not name a cause (PG-14, PG-15, PG-18, PG-21, PG-22). So every
  script here answers *"when this fails, what does it print"* — `cdp.send` has a deadline and detects
  a mid-command reload instead of hanging forever; `smoke` and `geometry` count against
  `src/demos/*/manifest.ts` rather than against whatever the page painted; boot failures print the
  first page error; ports are taken from the OS rather than from a colliding constant; and
  `GEOMETRY_SELFTEST=empty-row|over-spill|silent-change` makes `geometry` prove it can go red before
  its green is worth anything. See `playground/README.md` → "When the harness fails, what does it
  say?".
- **Demos are copy-pasteable.** No playground-specific props, helpers or globals are injected into a
  demo. What you read is what you would paste into an app.
- **One Vue instance and one CodeMirror instance** across the playground, the library sources and
  the compiled demos — `resolve.dedupe` + exact aliases + tsconfig `paths` for Vue; direct deps +
  dedupe + a single `optimizeDeps.include` pass for the CodeMirror/Lezer family (a second
  `@codemirror/state` breaks extension `instanceof` checks). The smoke test's `?editors=open` pass
  guards the CodeMirror half permanently.
- **Green means green.** `pnpm smoke`, `pnpm smoke:dist`, `pnpm docs:check`, `pnpm typecheck` and
  `pnpm typecheck:libs` all pass, or the change is not done. For a layout-shaped library, `pnpm
  geometry` as well — and its three self-tests are what make that green mean anything.

## Definition of "covered"

A library's tab is complete when **every option, every binding form, every emitted event, every
`data-*` state and every documented caveat has a card**. Concretely, for each package: walk the
README's options table and exports table top to bottom; anything not reachable from a card is a gap.

Known deliberate exclusions, all because they do not render:
`dependency-grouper` (a CLI that rewrites `package.json` files on disk),
`vue-provide-seeker` (VS Code extension), `inhouse-agent` (training harness).
`v-trap-focus` was cancelled 2026-08-09 (crowded niche) and its tab removed with it.

`bigdecimal-string` was on that list as "pure arithmetic" until 2026-09-13 and should not have been.
Its completeness rule is the same but the shape is different: **one card per README claim, and each
card shows the same expression computed twice — plain JavaScript on the left, the library on the
right, both evaluated in the page.** A printed `0.30000000000000004` would be a claim about
JavaScript rather than a demonstration of one. Building the tab that way immediately contradicted
three README sentences; see `PROGRESS.md`, 2026-09-13.

## Two views per tab

Since 2026-09-13 every tab has a **Playground** and a **Documentation** view, switched in the tab
header. `#<library-id>` is the playground — published READMEs link to that form and
`tickets/_STANDARDS.md` fixes it, so it must keep meaning what it means — and `#<library-id>/docs`
is the documentation, which leaves `#<library-id>/<file>.vue` free for the per-card deep link
`DOCS-1` still owes.

**The documentation view renders `../<package>/README.md` and nothing else.** There is deliberately
no second copy: this repository has repeated, documented README-vs-source drift, and two hand-kept
copies of the same API docs would diverge in public. If the docs are wrong, the README is wrong.
`src/markdown.ts` is a ~150-line renderer against a corpus we own; swap it for a real Markdown
library the moment that stops being true.

**The documentation view is checked like a card** (`DOCS-3`, `_STANDARDS.md` #6). `pnpm docs:check` boots
the same server and the same Chrome, renders `#<library-id>/docs` for every package, and:

- compares the code blocks on screen with the blocks in the file — `src/markdown.ts` dropping one is
  a defect only visible on the rendered page, and it was doing exactly that to fences indented inside
  a list item;
- compiles every fenced block that declares a runnable language, **in the page**, through
  `src/doc-sample.ts` → `src/sfc/compile.ts` — the playground's own compiler, its own aliased
  packages, a real `window`. Nothing is executed;
- lints the compiled render function for the two defects DZ-5 shipped — a browser global called from
  a template expression, and `.value` read off a ref in one;
- resolves every link, packs the tarball to confirm npm gets *this* README, and reports names
  documented as API that the source has never heard of.

The fence's **language word is the block's declaration** and there is no second marker: `vue` and
`ts` must compile, `text` / `bash` / `json` are prose, and an untagged fence is a finding rather than
a silent skip. `src/markdown.ts` only matches a bare language word, so any suffix syntax would stop
the block rendering as code on the page the gate exists to protect.

`scripts/docs/negative-control.mjs` runs 17 known-broken samples through the same paths before any
package is checked and aborts the run if one of them passes. A green `pnpm docs:check` has therefore just
demonstrated it can go red.

The remaining `DOCS-1` work is untouched: the Pages base path, the service worker the six
upload-dependent `v-dropzone` cards need on a static host, and the per-card deep link.

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
- [x] ~~**`v-fit-children`: ship a `FitChildrenPlugin` + `DIRECTIVE_NAME`**~~ — **never shipped, and deliberately so.** Marked done 2026-08-09 and repeated in `CLAUDE.md` and the package brief, but neither export has existed in any version of the package. `src/directive.ts` documents the decision: registering a directive is the application's job. Do not "restore" them.
      (v2.2.0, additive): the `INSTALLS` special case is gone, `installLibraries()` treats it like
      every tier-1 sibling.
- [ ] **Link each card to its README section** — acceptance: `DemoMeta` grows an optional `docs`
      anchor; the card header renders a link to `../<package>/README.md#<anchor>`. Makes the
      playground the entry point to the docs rather than a parallel universe.
- [ ] **Deploy it** — acceptance: `pnpm build` output published somewhere linkable (GitHub Pages
      off the eventual repo), so a README can say "try it" and a prospective user can, without
      cloning. Depends on the packages having a remote at all (see root `TASKS.md`).

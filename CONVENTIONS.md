# Conventions

Every package in this portfolio follows these conventions. Each package's instruction file inherits from this document — sub-agents working on a package read both `CONVENTIONS.md` and `<package>.md`.

## TypeScript is non-negotiable

- **All source in `.ts`** — no `.js` source files outside generated `dist/`.
- **`strict: true` in `tsconfig.json`** — no exceptions, no escape hatches, no `// @ts-ignore`. If TypeScript can't express something, the design is wrong.
- **All consumer-facing types are exported by name** from the package entry. A consumer can write:

  ```ts
  import type { ObserveOptions, IntersectConfig, MutateEvent } from 'v-observe'
  ```

  …and not have to recreate any type to wire up handlers. Internal types (state bags, plugin internals) stay un-exported.

- **No `@types/<pkg>` companion package** — types ship inside the package itself. `package.json` declares `"types": "dist/<entry>.d.ts"` and the dist tarball includes the `.d.ts` (verified via `npm pack --dry-run`).

- **Generic types on directives where the consumer's data type matters** — e.g. `v-mutate` (now `v-observe`'s `mutate` config) emits `added: HTMLElement[]`; if a consumer parses items into a domain type they can pass it through callback signatures without casting. Default to `HTMLElement` so callers without an opinion get the right thing.

- **Take `bigdecimal-string` as the bar**: that package's README leads with "Written in TypeScript. Full type safety. No `@types` package needed." and its `dist/index.d.ts` exports every public symbol. Match that.

## Modern syntax

- **ES2020+ target** (`tsconfig.json` `"target": "ES2020"`). Optional chaining (`?.`), nullish coalescing (`??`), top-level await where useful, `Array.prototype.at`, logical assignment operators all in scope.
- **`const` by default**, `let` only when reassignment is needed. Never `var`.
- **Template literals** over string concatenation.
- **Spread/rest** over `Object.assign` / `arguments`.
- **`for…of`** over `forEach` when the loop body has early-exit needs (`break`, `return`, `await`). `forEach` is fine for fire-and-forget side effects.
- **`async`/`await`** over raw `.then()` chains. Mix only when the chain is genuinely lazier (e.g. background work that doesn't gate the next step).
- **Object shorthand** (`{ foo }` not `{ foo: foo }`).
- **Arrow functions** for callbacks; named `function` declarations only when hoisting is intentional or the function is recursive.

## Simple, stupid code

- **No clever abstractions.** A directive does one thing; its source should read top-to-bottom in one file (or one short module per public function, like `v-teleport-to`'s `src/calculate-position.ts`). No factory-of-factory layers, no DI containers, no event-bus indirection inside a 200-line directive.
- **No unnecessary helpers.** If a function is called once, inline it. Refactor when the second caller appears, not before.
- **No unused exports.** Anything not consumed publicly stays internal. Tree-shake friendliness matters less than readability — unused exports are noise.
- **Minimal dependencies.** Runtime deps must be justified per-package and listed in the package's instructions file. `vue` is the only allowed peerDep for v-* directives. Build deps (`tsup`, `vitest`, `typescript`, `jsdom`) are shared across packages.
- **No defensive programming for impossible states.** If the type system says `opts.to` is `HTMLElement | null`, handle the two cases — don't add a third branch for "what if it's a string". Trust the types.
- **Comments only when the WHY is non-obvious.** Don't comment WHAT the code does (the code shows that). Do comment WHY a workaround exists (specific browser bug, CSS spec quirk, race condition).

## Single-purpose modules — the copy-paste audience

**Most people do not `npm install` these packages — they open the source and paste it into their
project.** That is the primary distribution channel, and the layout has to serve it: someone should
be able to take `src/` wholesale, or lift one module, and understand it without reading the rest.

- **Every source file has one purpose**, named for it, with a header comment saying what it owns.
  A reader should be able to guess the file from the concern and the concern from the file.
- **The entry file is a re-export barrel only.** `v<Name>.ts` re-exports `src/index.ts`; `src/index.ts`
  exports the public surface and nothing else. Internals stay un-exported.
- **Dependencies point downward, no cycles.** Leaf modules (`types.ts`, `constants.ts`) import
  nothing. When two modules genuinely need each other, inject the dependency rather than importing
  back up (`v-copy`'s controller takes its copy executor as an argument).
- **Name the invariant in `ARCHITECTURE.md`.** Every package has one, with the module map and the
  rule the split protects — e.g. "`execute-scroll.ts` is the only place a scroll happens",
  "`process.ts` is the one pipeline", "`state.ts` owns every reflection". Splitting is worthless if
  the next change reintroduces a second copy of the thing.
- **Bundle output must not change.** tsup follows the single entry and tree-shakes; a split is a
  readability change, not a packaging one. Verify with a build + `npm pack --dry-run`.

As of 2026-08-09 every v-* package follows this. `dependency-grouper` is the shape the user points
to as the reference for why it matters.

## Repo layout (per v-* directive)

```
v-<name>/
├── LICENSE                  MIT
├── README.md                consumer-facing docs (Install / Register / Usage / Options / Events / Behavior / Types / License)
├── package.json             see "package.json template" below
├── tsconfig.json            strict, ES2020, DOM lib, declaration:true, sourceMap:true
├── vitest.config.ts         jsdom env, includes the .test.ts
├── playground.html          standalone browser playground that imports dist/ ESM
├── ARCHITECTURE.md          module map + the invariant the split protects
├── v<Name>.ts               build entry — re-export barrel over src/, nothing else
├── src/                     single-purpose modules (see above); index.ts is the public surface
├── v<Name>.test.ts          vitest spec — colocated
├── dist/                    git-ignored, npm-published
├── node_modules/            git-ignored
└── .gitignore
```

Every package uses the `src/` layout — per-concern modules, leaf files with no internal imports, an
`index.ts` barrel re-exporting the public surface, and a thin entry file. Typical module names:
`types`, `constants`, `state`, `resolve`, `directive`, `plugin`, `use-<name>`, plus whatever the
domain needs (`ghost` / `visibility` for `v-fit-children`, `intersect` / `resize` / `mutate` for
`v-observe`, `upload` / `picker` / `paste` for `v-dropzone`).

## Package naming — scoped, always

**Every package publishes under the `@ozjsey` scope**: `@ozjsey/v-<name>`. The scope is a user
scope (it matches the npm username), so it needs no org and costs nothing — but scoped packages
default to **restricted**, so `"publishConfig": { "access": "public" }` is mandatory or the publish
fails.

The *library* keeps its unscoped identity everywhere else: the directive is still `v-fit-children`
in a template, the export is still `vFitChildren`, the repo and the README title are unchanged.
Only the install specifier is scoped.

This supersedes the earlier position that the scope was a workaround for squatted names
(`TASKS.md` P1, `@ozjsey/v-copy`). It is now the default for all of them, directives especially —
consistency across the portfolio beats holding an unscoped name per package.

## package.json template

```jsonc
{
  "name": "@ozjsey/v-<name>",
  "version": "1.0.0",
  "description": "<one sentence — leads with the real restriction the directive fixes>",
  "author": "Ozgur Seyidoglu",
  "license": "MIT",
  "repository": { "type": "git", "url": "git+https://github.com/ozJSey/<repo>.git" },
  "homepage": "https://github.com/ozJSey/<repo>#readme",
  "bugs": { "url": "https://github.com/ozJSey/<repo>/issues" },
  "type": "module",
  "main": "dist/v<Name>.min.js",
  "module": "dist/v<Name>.min.js",
  "types": "dist/v<Name>.d.ts",
  "exports": {
    ".": {
      "import": "./dist/v<Name>.min.js",
      "types": "./dist/v<Name>.d.ts"
    }
  },
  "files": ["dist"],
  "scripts": {
    "build": "tsup v<Name>.ts --format esm --minify --dts --out-dir dist --clean && mv dist/v<Name>.js dist/v<Name>.min.js",
    "test": "vitest run",
    "prepublishOnly": "npm run build"
  },
  "keywords": ["vue", "vue3", "directive", "<verb>", "<domain>", "<search-term>", "<search-term>"],
  "peerDependencies": { "vue": "^3.2.0" },
  "devDependencies": {
    "jsdom": "^25.0.0",
    "tsup": "^8.5.1",
    "typescript": "^5.5.0",
    "vitest": "^2.0.0",
    "vue": "^3.5.0"
  }
}
```

`keywords` field is npm SEO. Pack it with the search terms a developer would type to find this functionality (not just the literal name). Example for `v-dropzone`: `["vue", "vue3", "directive", "dropzone", "drop-zone", "file-upload", "uploader", "drag-drop", "file-drop", "attachment", "paste-image"]`.

## Playground coverage

`playground/` (private, never published) is the cross-package live demo app: **one tab per v-\*
library, one card per feature**, sources aliased to each package's `.ts` — never `dist/`. See
[`playground.md`](./playground.md).

- **A public API change is not shipped until its tab is updated.** New option, new binding form, new
  event, new `data-*` state → new or extended card, in the same run as the tests and the README.
- **Every documented feature has a card.** Walk the README's options and exports tables; anything
  not reachable from a card is a gap.
- **`npm run smoke` and `npm run typecheck` pass** in `playground/` before a run is called done.
  Smoke renders every tab in headless Chrome and fails on any compile error.

- **Smoke is not correctness. Measure the result.** Smoke proves a demo *renders without throwing*
  — nothing more. In Run 19 it passed while three `v-fit-children` demos showed chips clipped in
  half and every "+N" badge wrapped to the next line; the run was called done on green tests plus a
  clean smoke. For any library whose whole job is a visual/layout outcome, assert the geometry:
  `pnpm geometry` (`playground/scripts/geometry.mjs`) drives the real page, sweeps every slider,
  and fails on a child past its host's content edge or a host clipping itself. Extend it per
  library rather than eyeballing a screenshot — three separate wrong guesses about
  `v-fit-children`'s host sizing were each settled in seconds by a direct DOM probe (the one that
  mattered: inline `width: 141.359px` computing to `134.281px`, because `flex: 1` means
  `flex-basis: 0%` and basis beats `width`).
- The per-package `playground.html` files stay — they prove the library works from an import map
  alone, and some `playground.smoke.test.ts` suites mirror them. The two are complementary.

## Quality bar

Per the user's standing memory: **proper docs, tests, builds, ignores — solid repos**.

Each package on first publish must have:
- ✅ `README.md` with copy-paste install + usage in the first 30 lines
- ✅ Test suite passing (vitest, jsdom env, ≥80% line coverage isn't a target — coverage of the actual edge cases is)
- ✅ `dist/` with both source map and `.d.ts`
- ✅ `.gitignore` excluding `node_modules`, `dist` (rebuilt on publish), playground browser cache
- ✅ `LICENSE` (MIT)
- ✅ `package.json` `files: ["dist"]` whitelist (verified via `npm pack --dry-run`)
- ✅ `peerDependencies.vue` set to the **measured** floor, never a habit. Read every `from 'vue'`
  import and take the highest version that introduced one: `getCurrentScope`/`onScopeDispose`/
  `effectScope` are **3.2.0**, `toValue`/`MaybeRefOrGetter` are **3.3.0**. `^3.0.0` is the wrong
  default — it was declared on four packages that could never run on 3.0 or 3.1 (PEER-1, 2026-09-17),
  so a consumer installed cleanly and then the import threw. If a test matrix claims to prove the
  range, its lowest rung must BE the floor and must be pinned exactly — a caret there resolves to
  latest and the gate proves nothing.
- ✅ Types exported by name from the entry
- ✅ A tab in `playground/` covering the full documented surface

## What "feels native" means in practice

Every v-* directive in this portfolio must satisfy these:

1. **No wrapper components.** Consumer never writes `<DropzoneRoot>` or `<TrapFocusBoundary>`. Just `<div v-dropzone="...">`.
2. **No companion composable for the common case.** A composable can exist for advanced/programmatic use, but the directive must work standalone for 95%+ of cases.
3. **Works inside `v-for`** without per-iteration setup. State keyed by element via `WeakMap`; cleanup via `unmounted` lifecycle hook.
4. **Works inside Nuxt SSR** without `<ClientOnly>` wrapping. Vue 3's directive lifecycle hooks only fire client-side per the framework contract — leverage that, don't fight it.
5. **CSS state hooks via `data-*` attributes** — consumer styles based on `[data-<directive>-state="..."]` selectors, no JS state mirror needed.
6. **Single config object** as the binding value, not arg+modifier soup. `v-foo="{ option1, option2 }"` not `v-foo:arg.modifier="..."`.

## What "fixes a real restriction" means

Before adding a feature, ask: **does this fix something the native API or VueUse leaves to the consumer?** If the answer is no, don't add it.

Examples of real restrictions (allowed):
- IntersectionObserver fires on every threshold cross — consumer must bucket. → `v-observe`'s `crossed` event with direction.
- HTML drag-drop fires `dragleave` on every child enter — consumer must counter. → `v-dropzone` handles internally.
- `<Teleport>` is verbose for one-off positioning — consumer wires markup. → `v-teleport-to` directive.

Examples of NOT-real restrictions (rejected):
- "Wrap `addEventListener('click')` in a directive." → That's just the binding syntax; native click already feels native.
- "Wrap localStorage." → Not DOM-element-scoped. Use a composable.

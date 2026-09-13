# Instructions

Strategic briefs for the packages in `~/development/npm/`. One file per package. Sub-agents working on a package should read both [`CONVENTIONS.md`](./CONVENTIONS.md) and the package's own brief before starting.

`CONVENTIONS.md` owns the cross-cutting rules: TypeScript-first, types exported by name, modern syntax, simple stupid code, "feels native" quality bar, `package.json` template, repo layout. Every package brief inherits from it.

Each per-package brief contains: identity / why this exists / TypeScript exports contract / API / prioritized backlog / gotchas. Tasks are written as acceptance-criteria bullets so a sub-agent can pick the top unchecked item and ship it end-to-end in one run, the same workflow `v-teleport-to` already uses with its internal `TASKS.md` / `PROGRESS.md`.

## Current briefs

| File | Package | Status |
|---|---|---|
| [`CONVENTIONS.md`](./CONVENTIONS.md) | (cross-cutting) | Living |
| [`v-observe.md`](./v-observe.md) | `v-observe` | Greenfield — replaces archived `v-intersect`, `v-mutate`, `v-resize` |
| [`v-teleport-to.md`](./v-teleport-to.md) | `v-teleport-to` | Published v1.0.0 — v2.0 native-teleport plan |
| [`v-dropzone.md`](./v-dropzone.md) | `v-dropzone` | Greenfield — new package |
| [`playground.md`](./playground.md) | `playground/` (private) | Live — 7 libraries, 77 demos |
| [`v-fit-children.md`](./v-fit-children.md) | `v-fit-children` | Published 2.1.0, 2.2.0 pending publish — conservative posture, v3 proposal parked |

## Pending briefs (todo)

For directives already published at v1.0.0 that meet the bar — these need briefs that pin the differentiation vs VueUse and capture polish/v1.x maintenance work:

- `v-copy`, `v-scroll-into-view`, `v-select-text` (`v-fit-children` got its brief 2026-08-09)
  (`v-ripple` / `v-scroll-lock` cancelled 2026-08-02, `v-trap-focus` cancelled 2026-08-09 — all in
  `_archive/`; `v-longpress` was never ours)

For non-directive packages:

- `bigdecimal-string` (the type-export reference example), `dependency-grouper`, `inhouse-agent`, `vue-provide-seeker`

## Conventions

- **A public-API change is not shipped until the playground covers it.** `playground/` has a tab per
  v-* library and a card per feature; adding an option means adding a card in the same run, exactly
  like updating tests and the README. See [`playground.md`](./playground.md).
- **One task per run, fully shipped.** The `v-teleport-to` workflow has proven this — picking one acceptance-criteria item, implementing it, writing tests, building, validating, and updating `PROGRESS.md` in one pass produces auditable forward motion.
- **Acceptance criteria belong in the task line.** A task without acceptance criteria is a wish. The criteria specify what tests pin, what the public API surface change is, what bundle-size delta is expected.
- **Breaking changes get a `## P0 — vX.0` section** at the top of the brief, with a one-paragraph rationale and a migration guide as a checklist item.
- **Anything VueUse already does the same way is half-ass per this portfolio's bar.** If a brief proposes a feature that VueUse exposes identically, reject it — find the differentiation or drop the feature.

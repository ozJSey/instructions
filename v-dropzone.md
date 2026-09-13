# v-dropzone

> Read [`CONVENTIONS.md`](./CONVENTIONS.md) first.

**Status:** Greenfield (folder bootstrapped, no code yet). New package — biggest npm-search keyword in this space.

## Why this exists

HTML drag-and-drop is famously awful:

- **4 events to wire** — `dragenter`, `dragover`, `dragleave`, `drop`. All four need `event.preventDefault()` (or files won't drop). Forget one and the browser navigates away to open the file natively.
- **`dragleave` fires on every child enter** — moving the cursor between children of the drop zone fires phantom leave events. Consumer must implement an enter/leave counter.
- **No native UI affordance for "drop active"** — consumer wires hover state manually with class toggles or refs.
- **Manual file-type filtering** — `accept` attribute only works on `<input>`, not on drop. Consumer matches MIME types in JS.
- **No size/count validation built in** — every dropzone library re-implements this.
- **Paste-from-clipboard is separate** — `paste` event with `clipboardData.items` is a different API entirely. Most drop UIs need both paths.
- **No upload primitive** — consumers reach for separate libraries (`axios`, `fetch`, `xhr-onprogress` packages) and re-implement progress / cancel / retry.

`v-dropzone` is the directive that owns the entire pipeline: drop / paste / click-to-pick → validate → upload → state. Consumer writes CSS for `[data-dropzone="..."]` states; everything else is config.

## Naming for npm SEO

Search terms a developer would type to find this package: `dropzone`, `vue dropzone`, `file upload vue`, `drag drop vue`, `paste image vue`, `attachment vue`, `uploader vue`. Pack `keywords` in `package.json`:

```json
"keywords": [
  "vue", "vue3", "directive",
  "dropzone", "drop-zone",
  "file-upload", "uploader",
  "drag-drop", "file-drop", "drag-and-drop",
  "attachment", "paste-image",
  "multipart", "xhr-upload"
]
```

## TypeScript exports (must-haves)

`dist/vDropzone.d.ts` exports every consumer-facing type:

```ts
import type {
  DropzoneOptions,        // top-level config
  DropzoneHandler,        // (files: File[]) => void | Promise<void>
  DropzoneRejectReason,   // 'type' | 'size' | 'count'
  DropzoneRejectEvent,    // { files, reasons }
  UploadConfig,           // url-based or function-based upload config
  UploadProgressEvent,    // { file, loaded, total, percent }
  UploadResult,           // success | error discriminated union
  DropzoneApi,            // imperative API exposed via `ref` option
  DropzoneState,          // 'idle' | 'active' | 'rejected' | 'uploading' | 'success' | 'error'
} from 'v-dropzone'
```

Generic on the upload response: `UploadResult<TResponse = unknown>` so consumers parsing JSON responses get type-safe `.response`.

## API

```vue
<!-- bare: drop files, get File[] -->
<div v-dropzone="onFiles" />

<!-- full options -->
<div v-dropzone="{
  accept: 'image/*',
  multiple: true,
  maxSize: 5_000_000,
  maxCount: 5,
  paste: true,
  clickToPick: true,
  on: handleFiles,
  onReject: handleReject,
}" />

<!-- with built-in upload -->
<div v-dropzone="{
  upload: {
    url: '/api/upload',
    method: 'POST',
    headers: () => ({ Authorization: `Bearer ${token.value}` }),
    fieldName: 'file',
  },
  onProgress: (file, percent) => {},
  onUploaded: (file, response) => {},
  onError: (file, error) => {},
}" />

<!-- programmatic API exposed via ref -->
<script setup lang="ts">
import { ref } from 'vue'
import type { DropzoneApi } from 'v-dropzone'
const api = ref<DropzoneApi>()
</script>

<template>
  <div v-dropzone="{ ref: api, on: handleFiles }" />
  <button @click="api?.open()">Browse files</button>
  <button @click="api?.upload()">Upload all</button>
</template>
```

## Auto-managed state attribute

`[data-dropzone="..."]` reflects current state — pure CSS hooks, no JS state mirror needed:

| State | Trigger |
|---|---|
| `idle` | default |
| `active` | drag is over the zone (counter > 0) |
| `rejected` | last drop/paste failed type/size/count validation |
| `uploading` | at least one file is currently uploading |
| `success` | all uploads completed (auto-clears to `idle` after `successDuration: 1500` ms) |
| `error` | at least one upload failed |

Plus per-file CSS variables when uploading: `--dropzone-progress` (overall, 0..1) and `--dropzone-files-pending` (count).

## P0 — Bootstrap

- [ ] **Scaffold package** — package.json, tsconfig.json, vitest.config.ts, README.md, LICENSE (MIT), playground.html, vDropzone.ts (typed stub), vDropzone.test.ts. Mirror `v-trap-focus` repo layout exactly.

## P0 — Drag-drop core

- [ ] **Wire all 4 drag events with the enter/leave counter** — `dragenter` increments, `dragleave` decrements; `data-dropzone="active"` set when counter > 0, cleared on `drop` or counter reaches 0. All four handlers `preventDefault` + `stopPropagation`. Tests pin: cursor moving across children does NOT flip state to idle (counter > 0); leave-then-re-enter restores active; drop clears counter to 0.

- [ ] **`accept` filter with MIME pattern matching** — supports `'image/*'`, `'image/png,image/jpeg'`, file extensions like `'.pdf,.docx'`. Validates dropped files; non-matching files trigger `data-dropzone="rejected"` and fire `onReject({ files, reasons: ['type'] })`. Tests pin: wildcard matches subtype, extension matches case-insensitively, multi-pattern allows any-of, no-match fires reject with no `on` call.

- [ ] **`multiple`, `maxSize`, `maxCount` validation** — reject with cumulative reasons (a single drop can be rejected for multiple reasons). `multiple: false` rejects when N > 1. `maxSize: bytes` rejects oversize files individually (allowed files in same drop still go through). `maxCount: N` rejects when total exceeds. Tests pin each rule + composition.

- [ ] **Click-to-pick fallback** — `clickToPick: true` (default `false`) attaches a click listener that opens a hidden `<input type="file" accept multiple>` programmatically. Same `accept` / `multiple` config flows through. Tests pin: click triggers picker (mocked via `vi.spyOn(HTMLInputElement.prototype, 'click')`), picked files flow through validation just like dropped files.

- [ ] **Paste-from-clipboard support** — `paste: true` (default `false`) attaches a `paste` listener on the directive's element (or `document` if specified via `pasteOn: 'document'`). Reads `event.clipboardData.items` for files. Tests pin: pasting an image fires the handler with the File; non-file paste is ignored; `pasteOn: 'document'` catches pastes outside the zone.

## P0 — Upload pipeline

- [ ] **URL-based upload config** — `upload: { url, method?, headers?, fieldName?, formDataExtras? }`. Builds a `FormData` (one file per request by default, OR all files in one request via `batched: true`). Uses `XMLHttpRequest` (not `fetch`) for progress events. `onProgress(file, percent)` fires per upload. Tests pin: progress callback fires with monotonically increasing percent, success fires `onUploaded(file, parsedResponse)`, server 500 fires `onError(file, error)` with status code.

- [ ] **Function-based upload config** — `upload: async (file, signal) => Response | T` for custom transports (S3 presigned URLs, GraphQL multipart, gRPC). Directive awaits each, surfaces errors uniformly. `signal: AbortSignal` lets the function honor cancellation. Tests pin: async upload completes with returned value, thrown error fires `onError`, abort signal flips on cancel.

- [ ] **`DropzoneApi` programmatic methods** — `ref` option binds a Vue ref to a per-element imperative API:
  - `open(): void` — opens file picker
  - `upload(files?: File[]): Promise<UploadResult[]>` — uploads dropped (or specified) files
  - `cancel(file?: File): void` — cancels one or all in-flight uploads (calls `XHR.abort` / `AbortController.abort`)
  - `retry(file: File): Promise<UploadResult>` — retries a failed upload
  - `state: ComputedRef<DropzoneState>` — reactive current state
  - `pending: ComputedRef<File[]>` — files queued but not yet uploaded
  - `uploading: ComputedRef<File[]>` — files currently uploading
  - `failed: ComputedRef<File[]>` — files that errored
  
  Tests pin each method's behavior + state transitions; cancel mid-upload aborts and fires `onError` with abort reason; retry on a failed file restarts and updates state.

- [ ] **CSS variables during upload** — directive writes `style.setProperty('--dropzone-progress', String(overallPercent))` and `--dropzone-files-pending`. Cleared on `idle` / unmount. Tests pin updates during simulated progress + cleanup.

## P0 — State attribute lifecycle

- [ ] **`data-dropzone` reflects state with auto-clear timing** — transitions: `idle` ↔ `active` (drag), `idle` → `rejected` (validation fail; auto-clears to `idle` after `rejectDuration: 1500` ms), `idle` → `uploading` → `success` (auto-clears to `idle` after `successDuration: 1500` ms) / `error` (sticky until next drop or explicit `api.dismissError()`). Both durations configurable. Tests pin each transition + auto-clear timing + sticky-error behavior.

## P1 — Polish

- [ ] **README with copy-paste recipe for the 5 most common cases** — drop + display, drop + auto-upload, drop + validation, paste-from-clipboard, programmatic open. Each ≤ 15 lines.

- [ ] **Playground demo with all features** — `playground.html` standalone import of `dist/vDropzone.min.js`. Live demo for each P0 feature.

- [ ] **Bundle size budget** — ESM ≤ 6 KB minified. Validated in CI. Document any third-party dep in `package.json` `dependencies` (default: zero runtime deps; only `vue` peerDep).

## P2 — Future

- [ ] **Image preview helper** — `previews: true` option asks the directive to read each accepted image into a `URL.createObjectURL` and exposes them via `api.previews`. Auto-revokes URLs on unmount. Tests pin URL allocation + revocation.

- [ ] **Chunked uploads** — for large files, split into chunks of `chunkSize: bytes` and POST each with `Content-Range`. Resume support via `resumable: true` (skip already-uploaded chunks based on server response). Tests pin chunk math + resume logic with mocked server.

- [ ] **`<input type="file">` parity layer** — composable `useDropzone(options)` for cases where the directive's element binding is awkward (e.g. dropzone area is one element but the click-to-pick lives on a sibling button). Punted until requested.

## Gotchas

- `dragover` MUST `preventDefault()` or the drop event never fires. This is the most common bug in custom drop zones — pin it in a test.
- iOS Safari doesn't fire `paste` events for images outside `<input>` / `<textarea>`. Document the limit in README; users wanting full paste support need a focusable element.
- `XMLHttpRequest` `progress` events don't fire for cross-origin requests without `Access-Control-Allow-Origin`. Document that the upload server must allow CORS for progress to work.
- The directive does NOT add a visual style. Consumer is responsible for CSS — directive just exposes the `data-dropzone` state hooks. README must lead with this so users don't expect a styled box.

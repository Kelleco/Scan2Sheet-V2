# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

Scan2Sheet is a single-file, client-side mobile web app. **The entire application is `index.html`** — markup, styles, and logic. There is no build system, package manager, backend, or test suite. All third-party libraries (Tailwind, html5-qrcode, Google Identity Services, Google API client) load from CDNs at runtime.

It pairs an **Install No.** (QR code, 1–999) with a **Barcode** (Code 39, 7–13 digits), accumulates the pairs into a table, and exports them as CSV either to a local download or to a user-selected Google Drive folder.

## Running / developing

Serve over `http://localhost` (or HTTPS) — do **not** open via `file://`. The camera (`getUserMedia`) and Google OAuth both require a secure context, and `localhost` counts as secure.

```sh
python -m http.server 8000      # then open http://localhost:8000
```

Production is deployed via GitHub Pages (origin: `github.com/Kelleco/scan2sheet`). The Google OAuth Client ID and API Key are hardcoded near the bottom of the `<script>` block; the **Authorized JavaScript origin** in the Google Cloud Console must list whatever origin the app is served from, or Drive upload fails silently at the token step.

## Architecture

The core is a **3-step state machine** driven by the global `step` variable, all inside the single inline `<script>`:

- **Step 1** — `onScanSuccess` accepts only a QR code, validates 1–999, strips leading zeros via `normaliseInstallNo`, checks `isDuplicateInstallNo`, then advances to step 2.
- **Step 2** — accepts only a Code 39 barcode. Error correction: a read is buffered in `barcodeReadBuffer` and must be seen `REQUIRED_READS` times *in a row* before advancing; any differing value resets the buffer. Step 2 uses a shorter scan debounce (`DEBOUNCE_C39`) than step 1 (`DEBOUNCE_QR`) so the repeated reads accumulate fast.
- **Step 3** — manual Accept/Rescan confirmation, then the pair is committed.

`updateWorkflowUI()` re-renders the `#workflow-status` banner from `step` on every transition (including the step-2 dot-progress indicator). Format detection depends on the integer at `decodedResult.result.format.format` from html5-qrcode — not the decoded text.

State and persistence:

- `scannedItems` is the single source of truth — an array of `{ installNo, barcode }`, newest first (`unshift`).
- Persisted to `localStorage` under the key `scan2sheet_records` on every mutation via `saveToStorage()`; loaded on page load.
- Uniqueness is enforced on **both** `installNo` and `barcode` independently (`isDuplicateInstallNo` / `isDuplicateBarcode`), across camera scans, manual entry, and edits (edits pass an `excludeIndex` to skip the row being edited).

Three input/output paths share the same validators and the same `addRecordToTable` sink:

1. Camera scan (the state machine above).
2. Manual entry form — bypasses the state machine, calls the validators directly.
3. Google Drive — `saveToDrive` requests a token via GIS, opens the Picker for folder selection, then `pickerCallback` does a multipart upload to the Drive `files` endpoint. Local CSV export builds the same CSV string client-side and triggers a download.

Functions invoked from inline `onclick` handlers in template strings (`resetWorkflow`, `rescanBarcode`, `acceptBarcode`, `openEditModal`, `deleteRecord`) **must** be assigned to `window` or the buttons silently break.

## Conventions / gotchas

- The table is re-rendered wholesale by `updateTableUI()` (innerHTML from a `.map`) — there is no incremental DOM diffing; row indices in `onclick="...(${i})"` are positional into `scannedItems`.
- A `MutationObserver` reapplies `playsinline`/`webkit-playsinline` to the `<video>` element that html5-qrcode injects, to stop iOS forcing fullscreen — preserve this if touching camera init.
- `showToast` is the only user feedback channel; the Drive success toast temporarily becomes clickable to open the uploaded file.
- All user-facing copy uses en-dash ranges (e.g. `1–999`, `7–13`); match this when editing strings.

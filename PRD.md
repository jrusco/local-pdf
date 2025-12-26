# PRD: Privacy-First Browser PDF Toolkit (MVP + Architecture Decisions)

## Executive Summary

Build a privacy-first, static-only, offline-capable browser PDF toolkit offering an initial MVP of four tools: Merge PDF, Compress PDF, PDF → Images, and Images → PDF. The project is open-source, free, and hosted on GitHub Pages. The design emphasizes client-side processing, minimal and fast UI (Vue 3), and robust offline support via a Service Worker.

Goals:
- Fully client-side processing (no server uploads)
- Offline capability for all features
- Small initial bundle and lazy-load heavy modules
- Open-source dependencies only

## MVP Tools (initial launch)

1. Merge PDF
   - Multi-file input (drag-and-drop), preserve page order, optional reordering UI
   - Produce a single PDF download
2. Compress PDF
   - Preset levels (Low / Medium / High) and advanced slider
   - Two-tier approach: lightweight compression via `pdf-lib` reconstruction + optional stronger compression via qpdf WASM (lazy-load)
3. PDF → Images
   - Export each page as JPG/PNG, quality/DPI options
   - Use `pdf.js` rendering to Canvas, then export images via Canvas APIs
4. Images → PDF
   - Batch image upload (JPG/PNG), page sizing options (fit-to-page / original)
   - Build PDF using `pdf-lib` by embedding images per page

Tools listed for future development (see Roadmap).

## Library Evaluation & Recommendation

Requirement: a streamlined, consistent library strategy across the project while using only open-source dependencies and enabling offline-first operation.

Candidates analyzed:
- `pdf.js` — Strong at rendering PDFs to canvases/pages (ideal for PDF → Image). Not designed primarily for structural PDF editing (merge/split/rebuild).
- `pdf-lib` — Lightweight, MIT-licensed, capable of creating and modifying PDFs (merge, embed images, remove metadata). Does not render pages to raster images.
- qpdf (WASM) — Powerful stream-level PDF optimization and compression. Wasm builds exist; size may be moderate. License must be verified for the chosen OSS license.

Trade-offs and final recommendation:
- Use a small, explicit split of responsibilities rather than forcing a single library to do both jobs. Practically this means:
  - Primary manipulation and generation: `pdf-lib` (MIT). Use it for Merge and Images → PDF, metadata stripping, linearization work, and lightweight stream/object rewriting.
  - Rendering for raster export: `pdf.js` (open-source, suitable license). Use it only for PDF → Images rendering to Canvas.
  - Optional advanced compression: rely on `pdf-lib`-based heuristics and image re-encoding for typical compression needs.

Rationale:
- `pdf-lib` covers most structural operations (merge, create from images) reliably and is compact and easy to use in the browser.
- `pdf.js` is the de-facto standard for rendering PDF pages to Canvas and is required for high-quality PDF→image exports.
- Using both is common in client-side PDF tooling; each is used for what it does best. To preserve the “streamlined” intent, treat `pdf-lib` as the canonical manipulation library and `pdf.js` as a rendering dependency used only where necessary.

License notes: both `pdf-lib` and `pdf.js` are open-source. Confirm the exact license text during dependency selection (pdf-lib MIT, pdf.js has an open-source license).

## Offline & Static-Only Feasibility (Detailed)

Constraint: no backend — host on GitHub Pages. Offline mode is a critical feature.

Feasibility summary:
- All core operations can run entirely client-side if required dependencies and WASM modules are cached locally via a Service Worker.
- The project will operate in two modes:
  1. Online (initial load and when fetching lazy modules): download and cache necessary assets.
 2. Offline (after assets cached): all selected features work without internet.

For each MVP tool:
- Merge PDF: fully offline. `pdf-lib` runs in browser; no network required once library is cached.
- Compress PDF: basic compression (metadata removal, object reassembly) via `pdf-lib` — offline. Advanced compression with qpdf WASM — offline if the WASM binary is cached by the Service Worker; otherwise requires initial online fetch.
- PDF → Images: requires `pdf.js` and its worker; both must be cached. Rendering to Canvas is fully offline once cached.
- Images → PDF: uses `pdf-lib` and Canvas APIs — fully offline once cached.

Service Worker strategy to guarantee offline behavior:
- Use a PWA Service Worker (recommend `vite-plugin-pwa` or Workbox) to precache:
  - App shell (index.html, JS bundles, CSS)
  - `pdf-lib` bundle and types
  - `pdf.js` core and worker
  - WASM binaries (qpdf WASM) — mark as critical if you opt to ship it
  - Static assets (icons, fallback UI)
- Use runtime caching for large/lib-on-demand artifacts with a network-first fallback on first load, then serve from cache.
- Ensure Service Worker scope matches GitHub Pages base path (configure `base` in Vite and SW `scope`).
- Provide an explicit offline UX: detect offline status and show which heavy modules are cached or still downloading.

Storage & persistence:
- For long-lived offline use, rely on the browser cache (via SW) and IndexedDB (if temporary file storage is needed across sessions). Avoid storing user PDFs on remote servers.

Memory / performance constraints:
- Implement soft file-size limits (initially 150MB) and provide informative error messages.

## Technical Architecture

- Frontend: Vue 3 + TypeScript.
- Bundler / Dev: Vite (fast dev server, native ESM, great code-splitting).
- PDF manipulation: `pdf-lib` (primary for merge & images→pdf).
- PDF rendering: `pdf.js` (for PDF→images). Load `pdf.js` worker via dynamic import and cache it.
- Image conversion: Canvas API (native) for rasterization and encoding; use `createImageBitmap` where supported for performance.
- Optional WASM: qpdf WASM for advanced compression (lazy-load + cache). Fallback to pure-js heuristics if license or size concerns arise.
- Service Worker / PWA: `vite-plugin-pwa` (Workbox under the hood) to precache core libs and provide offline capability.
- Hosting: GitHub Pages (static site). Build pipeline: GitHub Actions to run `vite build` and publish to `gh-pages` branch.

Bundle & lazy-load strategy
- Keep initial bundle minimal: core app shell, main Vue runtime, lightweight UI components.
- Lazy-load heavy libs on demand via dynamic imports:
  - `pdf-lib` chunk (load on first use of Merge/Images→PDF/compress)
  - `pdf.js` + worker (load on first use of PDF→Images)
  - qpdf WASM (load only when user selects “Advanced compression”)
- Precache selected modules for offline use but allow on-demand fetching to keep initial page load fast.

Security & privacy
- All processing happens client-side; do not send files or telemetry by default.
- Use HTTPS by default (GitHub Pages supports HTTPS).
- Use Content Security Policy headers where possible to protect from XSS (limited on GH Pages but can be added via meta tag).

## UI/UX Notes (brief)
- Minimalist single-purpose tool pages with consistent dropzone pattern.
- Clear status & progress indicators for large operations.
- Offline indicators and caching status for heavy modules.
- Accessibility: keyboard navigation, ARIA labels for main controls.

## Compression Algorithm Recommendation

Two-tiered approach (best trade-off for a hobby-but-production-ready project):

1. Lightweight default (no WASM):
   - Use `pdf-lib` to remove metadata, optionally linearize documents, and recompress embedded images by converting them to canvas and re-encoding at reduced quality for image-heavy PDFs. This avoids shipping a large WASM and covers most common user cases.
2. Optional advanced path (WASM):
   - Offer qpdf WASM as an optional, lazy-loaded module for deeper stream-level optimization. Cache the WASM on first successful download for future offline use. Verify qpdf WASM license and size trade-offs before including it by default.

Notes: image re-encoding loses fidelity for image-heavy PDFs; for vector or font-heavy PDFs, stream-level compression (qpdf or Ghostscript) can be significantly better. Make this clear in UI and docs.

## Constraints & Acceptance Criteria

- All MVP tools must work offline after first-time caching.
- No file uploads to third-party servers.
- Default file size support: up to 50MB in memory; show clear message for larger files.
- Tooling: Vue + Vite build must produce a production bundle under ~1MB initial payload (heavy libs lazy-loaded).
- Licensing: all runtime dependencies must be open-source; project license recommended: MIT or Apache-2.0.

## Roadmap (To Be Developed Later)

- Split PDF (page ranges) — `pdf-lib`
- Rotate PDF (selective rotation) — `pdf-lib` + Organize UI
- Organize Pages (thumbnail reorder, selective delete, rotate) — `pdf-lib`
- Add Watermark (text/image watermarking) — `pdf-lib` + Canvas for previews

## Implementation Milestones & Next Steps

1. Validate library licenses (`pdf-lib`, `pdf.js`, any WASM libs) and pick project license.
2. Scaffold project with Vite + Vue 3 + TypeScript and `vite-plugin-pwa`.
3. Implement Merge & Images→PDF using `pdf-lib` with full offline SW caching.
4. Implement PDF→Images using `pdf.js`, worker, and SW caching.
5. Implement lightweight Compress (pdf-lib heuristics) and optional qpdf WASM integration behind an advanced toggle.
6. Add tests (unit + e2e for core flows) and CI to run builds and publish to GitHub Pages via GitHub Actions.

## Acceptance Tests (examples)

- Merge: upload 3 PDFs, reorder, merge → single PDF downloads and opens locally.
- PDF→Images: upload a 5-page vector PDF → download 5 JPG files with selectable quality.
- Images→PDF: upload 4 JPGs → generate multi-page PDF with expected page sizes.
- Offline: after first load and caching, refresh browser offline → all tools still execute (assuming lazy assets were previously cached).

## Risks & Mitigations

- Large WASM/bundles: mitigate with lazy-loading and optional advanced modules.
- Browser memory limits: set a conservative file-size recommendation and provide graceful errors.
- License mismatch: verify licenses early; if qpdf license is incompatible, rely on `pdf-lib` heuristics.

---

File: [PRD.md](PRD.md)

If you want, I can now:
- create the Vite + Vue scaffold and CI pipeline,
- or update the todo list to mark this PRD complete and start implementation.

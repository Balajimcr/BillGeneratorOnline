# AGENTS.md — BillGeneratorOnline

Operating guide for AI coding agents working in this repository.
Read this file before proposing or writing any change.
All project documentation is written in English.

---

## 1. Project overview

BillGeneratorOnline is a mobile-first **Progressive Web App (PWA)** that generates Indian business documents — rent receipts, invoices, quotations, estimates, and proforma invoices — and exports them to print-ready PDF.

The long-term goal is a clone-and-extend of billgenerator.in, with an India-first feature set (INR currency, DD-MM-YYYY dates, GST/GSTIN fields where applicable).

**Current state — pre-scaffolding.** The only runnable code is `rent_receipt_generator.html`, a self-contained vanilla-JS + Bootstrap + jsPDF/html2canvas rent receipt generator. There is no `package.json`, no `src/` directory, no build pipeline, and no test suite yet. Everything else in the repo is planning documentation (`README.md`, `plan.md`, `CLAUDE.md`, and this file).

**Phase 1 is local-only.** No backend, no authentication, no payment gateway, and no cloud storage exist or should be assumed. Do not add network calls to a server unless the task explicitly requires it.

---

## 2. Files in the repository

| File | Purpose | Notes |
|------|---------|-------|
| `README.md` | Human-facing project description | One-line summary of the product. |
| `plan.md` | Phased implementation plan | Source of truth for scope, architecture, data model, and roadmap. |
| `CLAUDE.md` | Companion guide for Claude Code | Mirrors much of this file. Read both. |
| `Agents.md` | This file | Agent-facing operating guide. |
| `rent_receipt_generator.html` | Legacy working implementation | ~995 lines of vanilla JS; the functional spec for the Rent Receipt document type. |

There is no `package.json`, `tsconfig.json`, `vite.config.ts`, `tailwind.config.js`, test config, or CI config yet.

---

## 3. Technology stack

The stack below is **resolved and planned** in `plan.md` and `CLAUDE.md`. It is not yet implemented.

| Concern | Decision |
|---------|----------|
| Framework | React 18 + TypeScript + Vite |
| UI | TailwindCSS + shadcn/ui |
| PDF (template) | `@react-pdf/renderer` |
| PDF (capture fallback) | `html2canvas` + `jsPDF` |
| PWA | `vite-plugin-pwa` (Workbox) |
| Local storage | IndexedDB via `idb`, behind a `DocumentRepository` interface |
| UI state | Zustand |
| Server state | React Query (future, for cloud sync only) |
| Forms | React Hook Form + Zod |
| Routing | React Router v6 |
| Testing | Vitest + React Testing Library |
| Lint / typecheck | ESLint + TypeScript |

> A previous plan proposed Vue 3 / Pinia. That was overridden in favor of React. If the human asks to switch to Vue, treat it as a project-level reversal and escalate rather than silently switching.

---

## 4. Build and test commands

These commands are **planned** but not yet wired because the project has not been scaffolded:

```bash
npm run dev        # Vite dev server
npm run build      # Production build
npm run preview    # Serve build locally — validate PWA/offline here
npm run test       # Vitest unit and integration tests
npm run lint       # ESLint + TypeScript typecheck
```

To run a single test file once scaffolded: `npm run test -- path/to/file.test.ts`

Before any command works, the first patch must create `package.json` and the Vite / TypeScript / Tailwind / shadcn / PWA configuration.

---

## 5. Code organization

### Current layout

```
.
├── README.md
├── plan.md
├── CLAUDE.md
├── Agents.md
└── rent_receipt_generator.html   # legacy reference implementation
```

### Target layout (from plan.md)

```
src/
  app/            # routing, providers, PWA registration
  documents/      # PER-TYPE: schema (Zod) + form config + pdf renderer
    rent-receipt/
    gst-invoice/
    ...
  templates/      # shared visual styles: modern | professional | minimal | retail
  pdf/            # react-pdf primitives + html2canvas/jsPDF fallback
  storage/        # DocumentRepository interface + IndexedDB impl
  components/     # shadcn/ui-based reusable UI
  hooks/
  lib/            # formatters (currency, date), qr, validators
legacy/           # rent_receipt_generator.html (reference, read-only intent)
```

A document type is considered complete only when it has:

1. A Zod schema
2. A form configuration
3. A PDF renderer
4. Unit / integration tests
5. An entry in the document registry

---

## 6. Legacy file: rent_receipt_generator.html

This file is the **functional specification** for the Rent Receipt document type. Treat it as read-only reference material. Do not delete it until React parity is verified.

Key behaviors to preserve when porting:

- Financial year auto-detection and dropdown generation (April–March fiscal years)
- Date formatting in `DD MMM YYYY` style for preview/export
- Monthly, weekly, and yearly billing period options
- Bulk receipt generation for monthly periods (single PDF or ZIP of individual PDFs)
- Signature capture: URL, upload from device, and gallery placeholder
- Four template styles (Template 1–4)
- Revenue stamp image embedded in the receipt
- `html2canvas` → `jsPDF` A4 export pipeline
- Required-field validation before export

---

## 7. Development conventions

### Hard rules

1. **No backend assumption.** All storage goes through the `DocumentRepository` interface. The Phase 1 implementation is `IndexedDbDocumentRepository`. A future `CloudDocumentRepository` must implement the same interface.
2. **No PII leaves the device in Phase 1.** Do not add analytics that ship form data, and do not POST customer details to third parties.
3. **Every document type is data, not code.** Adding a new document type means adding a registry entry + Zod schema + renderer. Never fork the whole app per document type.
4. **Preserve the legacy file.** Port `rent_receipt_generator.html` behavior into the React engine; do not delete the HTML file until parity is verified.
5. **Currency = INR (₹). Dates = DD-MM-YYYY.** Indian context is non-negotiable. (The legacy preview currently uses `DD MMM YYYY`; match the target locale format in the React port unless instructed otherwise.)
6. **Offline-first.** The app must be able to generate a PDF with the network off.

### Coding style

- Write TypeScript. Avoid `any`.
- Keep diffs minimal. Do not speculate or refactor unrelated code.
- No magic constants without a named explanation.
- Prefer composition over large components.
- Mobile-first layout: single column up to 640 px, two-pane form/preview from 1024 px.
- Tap targets should be at least 44 px on mobile.

---

## 8. Testing strategy

No tests exist yet. The planned strategy from `plan.md` is:

- **Unit tests (Vitest):** Zod schema validation (valid and invalid inputs), currency/date formatters, QR/UPI string builder, repository CRUD against a fake IndexedDB.
- **Component tests (Testing Library):** `DynamicForm` renders fields from config; preview updates on input; export button is enabled only when the schema is valid.
- **Integration tests:** fill form → save → reload from IndexedDB → duplicate → delete.
- **PWA validation:** Lighthouse PWA pass; manifest + service worker present; offline PDF generation verified manually in `npm run preview`.
- **PDF tests:** snapshot of react-pdf document structure; smoke test that export produces a non-empty A4 blob.

### Bug workflow

1. Reproduce with a test.
2. Fix the code.
3. Verify the test passes.

Never patch blind.

---

## 9. Security considerations

- **Local-only data in Phase 1.** Customer names, addresses, PAN numbers, amounts, and signatures stay in the browser’s IndexedDB / LocalStorage. Do not send them to any server.
- **Signature and logo uploads.** Treat uploaded images as data URLs or Blobs; do not upload them to external services for processing unless explicitly authorized.
- **No third-party POSTs for core flows.** PDF generation must work offline, so avoid dependencies that require network access during export.
- **Sanitize rendered HTML.** When porting the legacy preview, prefer React rendering or `@react-pdf/renderer` over `dangerouslySetInnerHTML` to avoid XSS from user input.
- **HTTPS required for PWA install** when deployed, but this is a deployment concern, not code.

---

## 10. Deployment

- Phase 1 is a static SPA suitable for Vercel, Netlify, or GitHub Pages.
- HTTPS is required for service worker registration and PWA install.
- `vite-plugin-pwa` will generate the manifest and Workbox service worker with precaching of the app shell and runtime caching of fonts/icons.

---

## 11. Workflow for contributors

Before coding:

- Restate the task in your own words.
- List the files you expect to change.
- State assumptions.
- Wait for human approval.
- If a change touches more than 3 files or crosses multiple subsystems, propose a decomposition first.

After every change, report:

- **What changed** (concise)
- **What could break** (regression risks)
- **Suggested tests** (concrete)

### Definition of done

- [ ] TypeScript: no `any` introduced; `tsc` clean.
- [ ] Tests cover the schema, the PDF render, and one edge case.
- [ ] Mobile (≤ 380 px) layout verified — minimal typing, large tap targets.
- [ ] Offline generation works.
- [ ] No new third-party network dependency for core flows.
- [ ] Change report (changed / could-break / tests) included.

---

## 12. Out of scope until explicitly requested

Do not scaffold or implement any of the following unless the human explicitly asks for it:

- Authentication or user accounts
- Razorpay / payments
- S3 / cloud sync
- Multilingual support
- AI field-fill
- GST reporting dashboards

These are tracked in `plan.md` as Phase 3+ work.

---

## 13. Immediate next steps

From `plan.md`, Phase 1 should proceed in small patches (≤ 3 files each):

1. Scaffold `package.json` + Vite / TypeScript / Tailwind / shadcn / PWA config.
2. Implement `storage/DocumentRepository.ts` + `IndexedDbDocumentRepository.ts` + tests.
3. Create the template registry and port `rent-receipt` (schema + form config + renderer) from the legacy HTML file + tests.
4. Wire `GeneratorPage` (form → preview → export).
5. Add PWA manifest / service worker and verify offline PDF generation.

When in doubt, read `plan.md` and `CLAUDE.md` before acting.

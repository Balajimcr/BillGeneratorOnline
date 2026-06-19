# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

Operating guide for AI/automated contributors working in `BillGeneratorOnline`.
Read this before proposing or writing any change.

---

## 1. What this project is

A mobile-first **Progressive Web App** that generates Indian business documents
(invoices, receipts, quotations, estimates, proforma invoices) and exports them
to print-ready PDF. Clone-and-extend of billgenerator.in.

**Current state:** Pre-scaffolding. The only working code is `rent_receipt_generator.html`
(a self-contained vanilla-JS + Bootstrap + jsPDF/html2canvas implementation, ~994 lines).
No `src/`, `package.json`, or React app exists yet. All architecture below is the
**target**, not the current state.

**Phase 1 is local-only.** No backend, no auth, no payment gateway, no cloud
storage. Do not add a network call to a server unless the task explicitly says so.

See `plan.md` for the full phased roadmap, component hierarchy, and data model.

---

## 2. Resolved technical decisions (do not re-litigate)

| Concern        | Decision                                  | Why |
|----------------|-------------------------------------------|-----|
| Framework      | React 18 + TypeScript + Vite              | shadcn/ui is React-native; richer PDF ecosystem |
| UI             | TailwindCSS + shadcn/ui                    | Requested; consistent primitives |
| PDF (template) | `@react-pdf/renderer`                      | Declarative, deterministic A4 output |
| PDF (capture)  | `html2canvas` + `jsPDF` (fallback only)    | Pixel-faithful; already proven in legacy file |
| PWA            | `vite-plugin-pwa` (Workbox)                | Requested; standard |
| Local storage  | IndexedDB via `idb`, behind a repository interface | Swappable for cloud later |
| State          | Zustand (UI/session) + React Query (later, for cloud) | Minimal; avoids Redux boilerplate |
| Forms          | React Hook Form + Zod                      | Schema-driven dynamic forms + validation |
| Routing        | React Router v6                            | Standard SPA routing |
| Colors/Fonts   | Primary `#FF7F50`; Montserrat headings / Nunito Sans body | Per design doc |

> A previous plan proposed Vue 3 / Pinia. **Overridden** in favor of React.
> If the human wants Vue, escalate — do not silently switch.

---

## 3. Hard rules

1. **No backend assumption.** Storage goes through `DocumentRepository`
   (interface). Phase 1 impl is `IndexedDbDocumentRepository`. A future
   `CloudDocumentRepository` must satisfy the same interface.
2. **No PII leaves the device in Phase 1.** No analytics that ship form data,
   no third-party POST of customer details.
3. **Every document type is data, not code.** A new type = a new entry in the
   template registry + a Zod schema + a renderer. Never fork the whole app per type.
4. **Legacy file is the spec for Rent Receipt.** `rent_receipt_generator.html`
   is the reference for fields, layout, and behavior. Key features to preserve:
   - Financial year auto-detection and range generation
   - Signature: draw (canvas), upload image, or type name
   - Multi-template selection (4 styles)
   - Monthly receipt loop (generate receipts for each month in a range)
   - html2canvas → jsPDF A4 export pipeline
   Do not delete the legacy file until React parity is verified.
5. **Currency = INR (₹). Dates = DD-MM-YYYY.** Indian context is non-negotiable.
6. **Offline-first.** The app must fully generate a PDF with the network off.

---

## 4. Target directory structure

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

A document type is "done" only when it has: schema + form config + renderer +
unit tests + an entry in the registry.

---

## 5. Commands (once `package.json` is scaffolded)

```
npm run dev        # Vite dev server
npm run build      # production build
npm run preview    # serve build — validate PWA/offline here
npm run test       # Vitest unit/integration
npm run lint       # ESLint + TypeScript typecheck
```

To run a single test file: `npm run test -- path/to/file.test.ts`

When validating PWA: build → preview → DevTools > Application (manifest +
service worker) → toggle offline → confirm a PDF still generates.

---

## 6. Workflow

Before coding:
- Restate the task, list impacted files, state assumptions, wait for approval.
- If a change touches **> 3 files** or multiple subsystems, stop and propose a
  decomposition first.

For bugs: reproduce with a test → fix → verify. Never patch blind.

After every change, report:
- **What changed** (concise)
- **What could break** (regression risks)
- **Suggested tests** (concrete)

Prefer minimal diffs. Preserve existing style. No speculative refactors. No
magic constants without a named explanation.

---

## 7. Definition of done

- [ ] TypeScript: no `any` introduced; `tsc` clean.
- [ ] Tests cover the schema, the PDF render, and one edge case.
- [ ] Mobile (≤ 380px) layout verified — minimal typing, large tap targets.
- [ ] Offline generation works.
- [ ] No new third-party network dependency for core flows.
- [ ] Change report (changed / could-break / tests) included.

---

## 8. Immediate next steps (Phase 1, ≤ 3 files per patch)

1. `package.json` + Vite/TS/Tailwind/shadcn/PWA scaffold.
2. `storage/DocumentRepository.ts` + `IndexedDbDocumentRepository.ts` + tests.
3. Template registry + `rent-receipt` (schema + form config + renderer) ported
   from `legacy/rent_receipt_generator.html` + tests.
4. `GeneratorPage` wiring (form → preview → export).
5. PWA manifest/SW + offline verification.

---

## 9. Out of scope until explicitly requested

Auth, user accounts, Razorpay/payments, S3/cloud sync, multi-language,
AI field-fill, GST reporting dashboards. These are Phase 3+ (see `plan.md`).

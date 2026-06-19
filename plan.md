# plan.md — BillGeneratorOnline

Phased implementation plan. Phase 1 ships a working local-only PWA. Later phases
are designed-for but gated behind explicit approval.

---

## 0. Current state

- Repo contains **one** file: `rent_receipt_generator.html` — a self-contained
  vanilla-JS + Bootstrap + jsPDF/html2canvas rent receipt generator (~994 lines).
- It already proves: dynamic form → live preview → html2canvas → jsPDF A4 export,
  financial-year handling, signature upload/draw, multi-template selection.
- **Migration intent:** preserve it under `legacy/`, port its behavior into the
  React template engine as the first document type, then deprecate.

---

## 1. Functional breakdown

### Document types (registry-driven)
Prioritized for India first (repo intent + existing code), Western set second.

**Phase 1 (India receipts):**
Rent Receipt, Fuel Receipt, Cash Receipt, Payment Receipt, Salary Slip/Receipt.

**Phase 2 (invoices/quotes):**
GST Invoice, Tax Invoice, Retail Invoice, Service Invoice, Sales Quotation,
Service Quotation, Cost Estimate, Work Estimate, Proforma Invoice.

Each type = `{ id, label, category, schema (Zod), formConfig, renderer, defaultTemplate }`.

### Core workflow
1. Pick document type (category grid homepage).
2. Dynamic form (RHF + Zod), auto-fill repeated fields from last use (local).
3. Live preview (debounced).
4. Choose template style: modern | professional | minimal | retail.
5. Export PDF (one click) / Print.
6. Save → list → edit / duplicate / delete (IndexedDB).

### Cross-cutting features
- Logo upload, signature (draw/upload/type), QR (UPI payment string), watermark.
- INR formatting, DD-MM-YYYY dates, GST/GSTIN fields where the type requires.

---

## 2. Architecture

```
UI (React + shadcn/ui)
        │
   Form layer (RHF + Zod)  ──►  live DocumentModel (in-memory)
        │                              │
        │                       PDF layer
        │                  ┌── react-pdf renderer (default)
        │                  └── html2canvas+jsPDF (capture fallback)
        │
  Storage layer  ──►  DocumentRepository (interface)
                          └ IndexedDbDocumentRepository  (Phase 1)
                          └ CloudDocumentRepository      (Phase 3, same interface)
```

The `DocumentRepository` interface is the single seam that lets cloud/auth be
added later without touching UI or PDF code.

### Component hierarchy (Phase 1)
```
<App>
 ├─ <HomePage>            category grid → type selection
 ├─ <GeneratorPage>
 │   ├─ <DynamicForm>      driven by formConfig
 │   ├─ <TemplatePicker>   modern/professional/minimal/retail
 │   ├─ <LivePreview>      renders selected template
 │   └─ <ExportBar>        PDF / Print / Save
 ├─ <DocumentsPage>       saved list: edit/duplicate/delete
 └─ <InstallPrompt>       PWA add-to-home-screen
```

### Routing
```
/                      home (type grid)
/new/:typeId           generator
/documents             saved list
/documents/:id         open saved (edit)
```

---

## 3. Data model & storage

```ts
// Stored shape (IndexedDB store: "documents")
interface StoredDocument {
  id: string;                 // uuid
  typeId: string;             // 'rent-receipt' | 'gst-invoice' | ...
  templateStyle: TemplateStyle;
  data: Record<string, unknown>; // validated by the type's Zod schema
  createdAt: number;
  updatedAt: number;
}

interface DocumentRepository {
  list(): Promise<StoredDocument[]>;
  get(id: string): Promise<StoredDocument | null>;
  save(doc: StoredDocument): Promise<void>;
  delete(id: string): Promise<void>;
  duplicate(id: string): Promise<StoredDocument>;
}
```

- **IndexedDB** (`idb`) holds documents + last-used field defaults.
- **LocalStorage** holds only lightweight prefs (theme, last template).
- No schema migration framework in Phase 1; version field reserved for later.

---

## 4. UI design

- Mobile-first: single column ≤ 640px, two-pane (form | preview) ≥ 1024px.
- Tap targets ≥ 44px; numeric keypads on amount fields; date pickers in
  DD-MM-YYYY; minimize free typing via dropdowns and saved defaults.
- Primary color `#FF7F50` (per Doc 2); Montserrat headings / Nunito Sans body.
- User flow: Home grid → Form (live preview) → Export. ≤ 3 taps to a PDF.

---

## 5. Testing strategy

- **Unit (Vitest):** each Zod schema (valid/invalid), currency & date
  formatters, QR/UPI string builder, repository CRUD against fake IndexedDB.
- **Component (Testing Library):** DynamicForm renders fields from config;
  preview updates on input; export button enabled only when schema valid.
- **Integration:** fill → save → reload from IndexedDB → duplicate → delete.
- **PWA validation:** Lighthouse PWA pass; manifest + SW present; offline PDF
  generation verified manually in `preview` build.
- **PDF:** snapshot of react-pdf document structure; smoke test that export
  produces a non-empty A4 blob.

---

## 6. Deployment

- Static SPA → Vercel / Netlify / GitHub Pages (no server needed Phase 1).
- HTTPS required for service worker + PWA install.
- `vite-plugin-pwa` generates manifest + Workbox SW with precache of app shell
  and runtime cache for fonts/icons.

---

## 7. Phased roadmap

| Phase | Scope | Gate |
|-------|-------|------|
| **1** | Scaffold (Vite/TS/Tailwind/shadcn/PWA), repository + IndexedDB, template engine, Rent + Fuel + Cash/Payment receipts, PDF export, offline | none — start here |
| **2** | GST/Tax/Retail/Service invoices, quotations, estimates, proforma; 4 template styles; QR UPI; WhatsApp share | after Phase 1 sign-off |
| **3** | Auth + accounts, `CloudDocumentRepository`, sync/backup, Razorpay premium, branding removal, GST reports, multilingual, AI field-fill | explicit business decision |

---

## 8. Competitive improvements (vs billgenerator.in) — backlog

- Auto-save drafts (debounced to IndexedDB).
- QR UPI payment block on invoices/receipts.
- WhatsApp share of generated PDF (`navigator.share` / wa.me link).
- Better, intentional templates (4 distinct styles) rather than one look.
- Multilingual (Hindi + regional) — Phase 3.
- AI-assisted field filling — Phase 3, opt-in, no PII to third parties without consent.

---

## 9. Open decisions (need human input before codebase generation)

1. **Stack:** confirm React/TS/Vite (this plan) vs Vue 3 (Doc 2).
2. **Phase 1 = local-only**, no auth/backend/payments — confirm.
3. **Type priority:** start with India receipt set (Rent → Fuel → Cash/Payment → Salary)?

No application code is written until 1–3 are answered.

---

## 10. Immediate next steps (Phase 1, in order, ≤ 3 files per patch)

1. `package.json` + Vite/TS/Tailwind/shadcn/PWA config scaffold.
2. `storage/DocumentRepository.ts` + `IndexedDbDocumentRepository.ts` + tests.
3. Template registry + `rent-receipt` (schema + form config + renderer) ported
   from `legacy/rent_receipt_generator.html` + tests.
4. `GeneratorPage` wiring (form → preview → export).
5. PWA manifest/SW + offline verification.

Each step ships with its change report (changed / could-break / tests).

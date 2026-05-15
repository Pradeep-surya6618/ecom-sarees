# E-commerce Saree Template — Design Spec

**Date:** 2026-05-15
**Status:** Approved, ready for implementation planning
**Author:** Brainstorming session with avinashincrix@gmail.com

## Purpose

Build a full-stack e-commerce web app for selling sarees (reference: shreedevitextile.com) on Next.js 16 + React 19, designed from day one as a **reusable template**: a new client gets a fork, edits `brand.config.ts`, swaps logo files, points new env vars at their AWS account, and ships in 2–4 days.

This spec covers **subsystem #1 only — the foundation** (template/branding core, DynamoDB data layer, app shell). The remaining 9 subsystems are listed in the roadmap for context; each will receive its own spec + plan when reached.

## Locked-in decisions

| Concern | Decision |
|---|---|
| Template reuse model | Fork-per-client. Each client gets their own git fork, AWS account, deployment. No multi-tenant logic. |
| UI stack | Tailwind v4 + shadcn/ui (components copied into repo, not a dependency). |
| Region | India-first. INR. |
| Payment | Razorpay (UPI / cards / netbanking / wallets). |
| Shipping | Shiprocket aggregator (Delhivery, Bluedart, Ekart, etc. behind one API). |
| Auth | Email + password (bcrypt) + Google OAuth, via NextAuth.js v5. Guest checkout allowed. |
| Theme mode | Light only. Dark mode deferred (not designed in). |
| Media hosting | AWS S3 + CloudFront. Admin uploads via presigned URLs. `next/image` reads through CloudFront. |
| Search | DynamoDB GSIs + faceted filters (category, color, fabric, occasion, price). No free-text search. Suitable to ~10k products. |
| Database | DynamoDB single-table. DynamoDB Local via Docker for dev. |
| Deployment | Vercel default; AWS Amplify documented as alternate. Each client's AWS account holds DynamoDB + S3. |
| Runtime | Next.js 16.2.6, React 19.2.4, App Router, React Compiler enabled, dev port 4000. |

## End-to-end roadmap (all subsystems)

Each item below will get its own spec + plan cycle. Build order is the order listed.

**Foundation**
1. **Template/branding core + DynamoDB data layer + app shell** ← this spec.

**Storefront**
2. **Catalog** — Product, Category, Attribute entities. GSIs for: by-category, by-collection, by-featured, by-price-range. PLP with faceted filters. PDP with image gallery, variants, related products.
3. **Cart & wishlist** — Guest cart in `localStorage`; signed-in cart in DynamoDB under `USER#<id>` PK; merge on login.
4. **Auth** — NextAuth.js v5 with `Credentials` + `Google` providers, DynamoDB adapter.
5. **Checkout & payment** — Multi-step (Address → Shipping → Pay → Confirm). Razorpay Checkout embedded; server creates order, verifies signature on webhook, writes Order entity. Idempotent.
6. **Order tracking** — Order history, status timeline, PDF invoice, email receipts.

**Operations**
7. **Shipping (Shiprocket)** — Rate API at checkout, auto-create shipment on "ready to ship", tracking webhook updates `Order.shippingStatus`. Manual override fields.
8. **Admin panel** — Separate `/admin` route group. Role-based access. Product CRUD with image upload, order management, customer list, inventory, sales dashboard.
9. **Media pipeline** — S3 bucket per client (env-configured), CloudFront distribution, admin presigned PUT, `next/image` loader pointed at CloudFront. Variants generated on upload.

**Cross-cutting**
10. SEO (metadata API, sitemap, JSON-LD product schema), analytics (GA4 + server-side purchase event), transactional email (Resend default, SES alternate), error tracking (Sentry).

**Rough timeline (one engineer, sequential):** Foundation 1–1.5 wk → Catalog 1 wk → Cart 3 d → Auth 3 d → Checkout 1 wk → Shipping 4 d → Order tracking 3 d → Admin 1.5 wk → Media + polish ongoing. **~6–8 weeks to launchable v1.** Subsequent clients ~2–4 days each.

---

## Foundation sub-project — detailed design

### What it delivers

A Next.js 16 app with:
- A single `brand.config.ts` controlling everything client-specific (name, colors, fonts, logo paths, contact, social, feature flags).
- A working DynamoDB single-table data layer with a repository pattern.
- A responsive light-mode app shell (header / footer / mobile nav) wired to brand tokens.
- Running locally against DynamoDB Local in Docker.

### File structure

```
src/
  app/
    layout.js              // root layout, loads brand + fonts
    page.js                // homepage placeholder (real content in catalog sub)
    globals.css            // Tailwind v4 + brand CSS variables
  brand/
    brand.config.ts        // THE template knob
    brand.types.ts         // Zod schema validating the config at build time
    fonts.ts               // next/font/google loaders, swapped via config
  components/
    ui/                    // shadcn components copied in
    layout/
      site-header.tsx
      site-footer.tsx
      mobile-nav.tsx
      logo.tsx             // reads brand.logo.* paths
    providers/
      brand-provider.tsx   // exposes brand via React context
  lib/
    db/
      client.ts            // DynamoDB DocumentClient singleton
      schema.ts            // table name + key helpers (pk/sk builders)
      repositories/
        base.ts            // shared query/transact helpers
        // entity repos added per subsystem
    env.ts                 // typed env parser (Zod), fails fast on boot
  styles/
    tokens.css             // CSS variables: --brand-primary, --brand-fg, etc.
public/
  brand/
    logo.svg               // swapped per client
    favicon.ico
    og-default.png
docker-compose.yml         // DynamoDB Local + admin UI
scripts/
  seed-dev-db.ts           // creates table + seeds tiny dev dataset
  bootstrap-prod-table.ts  // one-shot table creation for new client
.env.local.example         // every env var documented
docs/
  TEMPLATE.md              // how to rebrand for a new client
  ARCHITECTURE.md
```

### Brand / theme system — the template heart

**`brand.config.ts` is the single source of truth a new client edits:**

```ts
export const brand = {
  name: "Shree Devi Textiles",
  shortName: "ShreeDevi",
  tagline: "Authentic sarees since 1985",
  contact: { email, phone, whatsapp, addressLines: [...] },
  social: { instagram, facebook, youtube },
  colors: {
    primary:   "oklch(0.55 0.18 25)",
    secondary: "oklch(0.85 0.05 80)",
    accent:    "oklch(0.65 0.15 145)",
    fg:        "oklch(0.15 0 0)",
    bg:        "oklch(0.99 0 0)",
    muted:     "oklch(0.92 0 0)",
  },
  fonts: { heading: "Playfair_Display", body: "Inter" },
  logo: { src: "/brand/logo.svg", width: 160, height: 40, alt: "Shree Devi Textiles" },
  features: { wishlist: true, reviews: false, blog: false },
};
```

**How it drives the UI:**
- Zod schema (`brand.types.ts`) validates at import — typos = build fail, not runtime mystery.
- A small server function reads `brand.colors` and emits CSS variables into `<head>` via the root layout. shadcn components reference those variables (`--primary`, `--background`, etc.), so changing one file rebrands every component.
- Fonts use `next/font/google` with the family pulled from `brand.fonts`. Allowed values validated by Zod against a curated allowlist of supported Google Fonts families (defined alongside the schema) so a typo or unavailable font fails the build.
- `features` map gates subsystems — a client with `reviews: false` gets the route and nav link removed at build time.

**What goes in `.env` vs `brand.config.ts`:**
- **`.env`** — secrets and infra: `AWS_REGION`, `DYNAMO_TABLE_NAME`, `S3_BUCKET`, `CLOUDFRONT_URL`, `RAZORPAY_KEY_ID`, `RAZORPAY_KEY_SECRET`, `SHIPROCKET_EMAIL`, `SHIPROCKET_PASSWORD`, `NEXTAUTH_SECRET`, `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `RESEND_API_KEY`.
- **`brand.config.ts`** — identity and design: name, copy, colors, fonts, social links, feature flags, logo paths.
- **Rule:** secret or environment-specific → env. Designer/PM should edit it → config.

### DynamoDB single-table design

One table (name via `DYNAMO_TABLE_NAME` env), keyed by generic `PK` (string) + `SK` (string). Entities differentiated by key prefix.

| Entity | PK | SK | Notes |
|---|---|---|---|
| Product | `PRODUCT#<id>` | `META` | core fields + denormalized category |
| ProductImage | `PRODUCT#<id>` | `IMAGE#<order>` | one row per image |
| Category | `CATEGORY#<slug>` | `META` | |
| CategoryProduct | `CATEGORY#<slug>` | `PRODUCT#<id>` | reverse lookup |
| User | `USER#<id>` | `META` | |
| UserByEmail | `USERBYEMAIL#<email>` | `META` | unique-key shadow row |
| Order | `ORDER#<id>` | `META` | |
| OrderByUser | `USER#<id>` | `ORDER#<created-at>#<id>` | history sorted by date |
| Cart | `USER#<id>` | `CART` | one row per user, items as list |
| Address | `USER#<id>` | `ADDRESS#<id>` | |

GSIs (define both at table creation; entities populate them as needed):
- **GSI1** (`GSI1PK`, `GSI1SK`) — generic reverse lookup. Catalog uses `CATEGORY#<slug>` → products sorted by price or recency.
- **GSI2** (`GSI2PK`, `GSI2SK`) — by-status indexes (e.g., `ORDERSTATUS#PENDING` → orders sorted by date).

**Repository pattern:** `lib/db/repositories/base.ts` exposes generic `getItem`, `putItem`, `query`, `transactWrite`. The foundation also includes a **minimal Product and Category repository** (needed by the seed script and by the catalog subsystem next). All other entity repos (User, Order, Cart, Address) are added per their owning subsystem.

**Local dev:** `docker-compose.yml` runs DynamoDB Local + dynamodb-admin UI. `pnpm seed:dev` creates the table and inserts ~10 sample sarees via the Product/Category repos.

### App shell

- **Root layout** loads fonts, brand provider, injects CSS vars, sets `<html lang>` from brand.
- **Header**: logo (left), category nav (center, desktop only), search/cart/account icons (right), sticky on scroll. Mobile collapses to logo + hamburger + cart.
- **Mobile nav**: shadcn `Sheet` drawer with categories + account links.
- **Footer**: 4-column on desktop (about, categories, customer service, social) → accordion on mobile. All text/links from `brand.config`.
- **Container system**: Tailwind container, max-width `7xl`, responsive horizontal padding.
- **Breakpoints**: Tailwind defaults (`sm 640 / md 768 / lg 1024 / xl 1280`).
- **Cart + account icons** render placeholder badges (count `0`) so the chrome is final now and not redesigned when later subsystems wire them up.

### Data flow (foundation only)

- Server components read brand + env at module load (cached).
- Server components read DynamoDB directly via the repository layer (no API routes needed for RSC reads).
- Client components that need brand values get them via `BrandProvider`, hydrated from the server tree.
- Writes (later subsystems) go through Route Handlers or Server Actions — never client → DynamoDB directly.

### Error handling

- **Env parsing**: Zod schema in `lib/env.ts`. Missing or invalid env vars throw at module load → fail loudly on boot.
- **Brand config**: Zod schema validates at import. Build fails if misconfigured.
- **DB layer**: Repositories throw typed `DbError`. Route handlers catch at boundary, log full error, return generic message to client. No `try/catch` inside business logic — only at boundaries.
- **Route segments**: `error.js` per major segment renders a branded error page, not a stack trace.

### Testing

- **Vitest** for unit tests: brand config schema validation, env parsing edge cases, repository CRUD against DynamoDB Local (real DB, no mocks).
- **Playwright** smoke test: homepage renders, header shows brand name, mobile nav opens. Grows with each subsystem.
- "Foundation done" = `pnpm test` passes + `pnpm dev` loads the shell with the seed brand applied.

### Explicit non-goals for this sub-project

- No product listing or detail page logic (catalog subsystem).
- No cart, no auth, no checkout, no admin, no shipping.
- No Razorpay or Shiprocket code (env vars defined and validated, but nothing reads them yet).
- No S3 upload UI.
- No email, SEO, or analytics wiring.

### Acceptance criteria

The foundation is complete when:

1. `pnpm dev` boots; landing page shows branded header + footer with seed brand applied.
2. Changing `brand.config.ts` (colors, name, logo path) rebrands every visible surface on next reload — no other file touched.
3. `docker-compose up` brings up DynamoDB Local; `pnpm seed:dev` creates the table and inserts sample data; a repository read-test passes.
4. `pnpm build` succeeds; misconfigured brand or env fails the build with a clear Zod error.
5. `docs/TEMPLATE.md` explains the rebrand-for-new-client steps in ≤10 numbered actions.
6. Lighthouse on the shell-only homepage (mobile profile): Performance ≥ 90, Accessibility ≥ 95, Best Practices ≥ 95, SEO ≥ 90.

## Next steps

After this spec is approved, hand off to the `writing-plans` skill to produce a step-by-step implementation plan for the foundation sub-project. The plan will sequence:
1. Project bootstrap (deps, Tailwind v4, shadcn init, Zod, AWS SDK v3, NextAuth scaffold deps).
2. `brand.config.ts` + Zod schema + CSS variable injection.
3. App shell components (header, footer, mobile nav, logo).
4. Env parser.
5. DynamoDB client + base repository + docker-compose + seed script.
6. Tests for each of the above.
7. `TEMPLATE.md` write-up.
8. Acceptance check against the criteria above.

Catalog (sub-project #2) and all later sub-projects are out of scope for this plan and will each get their own brainstorm → spec → plan cycle.

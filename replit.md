# FuelFlow

Egyptian smart fuel network web app for drivers and B2B fleet managers. Find network stations, manage a digital fuel wallet (Meeza / Visa / Apple Pay / InstaPay), track per-vehicle consumption, pay contactlessly via QR / NFC, and monitor B2B fleet operations.

All amounts are in EGP (Egyptian Pound). The UI is fully bilingual English / Arabic with full RTL support and a header toggle. Persists choice in `localStorage` (`fuelflow.lang`). UI never uses emojis.

## Internationalization

- `artifacts/fuelflow/src/lib/translations.ts` — single source of truth for both `en` and `ar` strings.
- `artifacts/fuelflow/src/lib/i18n.tsx` — `LanguageProvider` + `useLanguage()` hook exposing `language`, `dir`, `setLanguage`, `t(key, vars?)`, `formatCurrency`, `formatNumber`, and `dateLocale` (date-fns ar / enUS). On change it updates `<html dir>`, `<html lang>`, toggles `.rtl` class, and persists to `localStorage`.
- Currency: `formatCurrency(value)` returns `EGP 1,234.56` in EN and `1,234.56 ج.م` in AR. Numbers always use Latin digits.
- Sidebar (`app-shell.tsx`) docks dynamically to the start side based on `dir`.
- Tailwind logical properties (`ms-`/`me-`/`ps-`/`pe-`/`start-`/`end-`) are used so layout flips automatically. The Leaflet map is wrapped in `dir="ltr"` so map controls remain LTR even in AR mode.
- Language selector is a **dropdown** in the top-bar showing "EN" / "ع" and listing "English" / "العربية" with a checkmark on the active choice.
- For QA / screenshots, `?lang=ar` or `?lang=en` query param forces the initial language.

## Architecture

This is a pnpm monorepo with the following key artifacts and libraries:

- `artifacts/fuelflow` — React + Vite web app, Tailwind v4, wouter routing, tanstack-query, framer-motion, Recharts, react-leaflet, qrcode. App shell with sidebar + topbar (wallet pill, notifications bell). Theme: deep midnight + warm amber accent.
- `artifacts/api-server` — Express 5 + Drizzle + Pino backend. All routes under `/api`.
- `artifacts/mockup-sandbox` — design canvas (Replit-provided).
- `lib/api-spec` — OpenAPI 3 spec is the source of truth (`openapi.yaml`). Generates Zod schemas and React Query hooks.
- `lib/api-zod` — generated Zod schemas (`@workspace/api-zod`). Single re-export `export * from "./generated/api"`.
- `lib/api-client-react` — generated typed React Query hooks (`@workspace/api-client-react`).
- `lib/db` — Drizzle schema for Postgres. Tables under `lib/db/src/schema/`: stations, wallets + wallet_transactions, vehicles, transactions + fueling_sessions, support_tickets + notifications, fleet_company + fleet_drivers + fleet_vehicles + fleet_transactions + fleet_alerts.
- `scripts` — `pnpm --filter @workspace/scripts run seed-fuelflow` re-seeds the database with realistic Egyptian data: 8 stations across Cairo, New Cairo, Heliopolis, 6th of October, and Alexandria; 3 personal vehicles with Arabic plates; ~60 personal transactions in EGP; "Nile Logistics Co." fleet of 6 trucks (MSR plates) with ~200 transactions; Egyptian drivers; alerts; notifications; support tickets. Default Cairo coords used for distance sort: `30.0444, 31.2357`.

## Authentication (Clerk)

- Clerk is provisioned via Replit-managed white-label auth (`setupClerkWhitelabelAuth()`).
- `CLERK_SECRET_KEY`, `CLERK_PUBLISHABLE_KEY`, `VITE_CLERK_PUBLISHABLE_KEY` env vars are set by Replit.
- Frontend: `@clerk/react`, `@clerk/themes` (shadcn). `ClerkProvider` wraps the whole router; `publishableKeyFromHost` is NOT used — `VITE_CLERK_PUBLISHABLE_KEY` is used directly so Clerk JS loads from the standard CDN in dev.
- Routes: `/sign-in/*?` and `/sign-up/*?` are dedicated Clerk-rendered pages with FuelFlow branding (amber theme, logo, custom copy). `HomeRoute` at `/` shows the landing page for signed-out users and the dashboard for signed-in users. All other routes redirect to `/sign-in` when signed out.
- API server: `clerkMiddleware` from `@clerk/express` is mounted with host-based publishable key resolution. `clerkProxyMiddleware` is mounted at `/api/__clerk` (only active in production).
- User profile (name, email, avatar) is shown in the sidebar footer via `useUser()`.
- CSS layer order: `@layer theme, base, clerk, components, utilities` declared before `@import 'tailwindcss'`. `@clerk/themes/shadcn.css` imported without layer annotation. TailwindCSS optimization disabled (`optimize: false`) to prevent layer reordering in production builds.

## Reports & Export

- Reports page (`/reports`) has an **Export** dropdown button with:
  - "Export as Excel (.xlsx)" — uses `xlsx` library to write a multi-sheet workbook (Monthly Trends + Yearly Overview).
  - "Export as PDF" — uses `jspdf` + `jspdf-autotable` to produce a styled PDF with amber header, per-month table, and summary.
- Both are client-side (no server call) and trigger a browser download.

## Invoices (Egyptian ETA format)

- Invoice page (`/invoices`) accessible from the sidebar under main nav.
- User inputs buyer name, address, and optional tax registration number.
- Period selector filters which transactions are included.
- Generates a PDF invoice compliant with Egypt Tax Authority (ETA) e-invoice standards:
  - Seller info (FuelFlow Egypt, Tax Reg No: 345-678-901)
  - Buyer info (from form)
  - Unique invoice UUID (`FF-{timestamp}-EGY`)
  - Line items table (station, date, liters, price/L, amount)
  - Subtotal, VAT (14%), Total
  - ETA compliance notice in footer
- Uses `jspdf` + `jspdf-autotable`, client-side download.

## Email Notifications

- API routes at `/api/email/send-invoice` and `/api/email/send-report` (POST).
- Uses `nodemailer` with SMTP configured via env vars: `SMTP_HOST`, `SMTP_PORT`, `SMTP_USER`, `SMTP_PASS`, `SMTP_SECURE`, `SMTP_FROM`.
- Returns `503` with a descriptive message if SMTP vars are not set.
- `send-invoice` renders an HTML email with invoice details (items, subtotal, VAT 14%, total).
- `send-report` renders an HTML email with monthly consumption summary (stats, table).
- To enable: set the SMTP env vars in the Replit Secrets panel.

## Routes

Driver UX:
- `/` Dashboard – MTD KPIs, vehicle breakdown chart, recent transactions, quick actions
- `/stations` Network station map (Leaflet) + searchable list
- `/wallet` Balance, recharge modal (Mada / Visa / Apple Pay / STC Pay), wallet ledger
- `/vehicles` List, add, detail page with consumption charts
- `/fuel` Pay-at-pump: pick vehicle + station → generate QR with 5-min expiry, dev-simulate completion
- `/transactions` Filterable transaction history
- `/reports` Monthly + yearly Recharts visualizations
- `/support` List + create tickets (complaint / suggestion / question)
- `/notifications` Read/unread feed, mark-read

B2B:
- `/fleet` Company KPIs, alert banners, top consumers, monthly chart, vehicle table

## Settlement Module

- Settlement page (`/settlements`) accessible from the sidebar under main nav (Banknote icon).
- Per-station financial summaries calculated from real `transactions` table data.
- Calculation formula:
  - Total Sales = sum of completed transactions for the station in the period
  - App Commission = Total Sales × commission_rate (default 2.5%)
  - Gateway Fees = Total Sales × gateway_fee_rate (default 1.5%)
  - Net Payable = Total Sales − Commission − Gateway Fees − Adjustments
- Three-step flow: **Preview** (live calculation without saving) → **Save** → **Update Status**
- Status tracking: Pending / In Progress / Paid (color-coded badges)
- Summary cards showing total EGP value per status bucket
- Filters: by station and by status
- Export as Excel (.xlsx) or PDF with full deduction breakdown and totals
- DB: `settlements` table in `lib/db/src/schema/settlements.ts`
- API endpoints (all under `/api`):
  - `GET /settlements` — list with optional `station_id`, `date_from`, `date_to`, `status`, `period` filters
  - `GET /settlements/calculate` — live preview without saving (requires `station_id`, `date_from`, `date_to`)
  - `POST /settlements` — create and persist a settlement
  - `PATCH /settlements/:id` — update status and/or notes
- Commission/fee rates are configurable per settlement in the dialog (defaults: 2.5% / 1.5%).

## Running

Workflows are managed by Replit; do not run `pnpm dev` from root.

- API server (8080) → `artifacts/api-server: API Server`
- Web app (22013, served at `/`) → `artifacts/fuelflow: web`

To re-seed the database: `pnpm --filter @workspace/scripts run seed-fuelflow`
To regenerate API hooks/Zod after editing `openapi.yaml`: `pnpm --filter @workspace/api-spec run codegen`

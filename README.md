# ERP System

A multi-tenant SaaS ERP for small/medium businesses — sales invoicing, warehouse/inventory management, expense tracking, customer & supplier management, multi-branch operations, a role-based team system, and dashboard analytics.

- **Backend:** Node.js / Express / MongoDB — folder structure & full API route tables in [`backend/README.md`](backend/README.md)
- **Frontend:** Next.js (App Router) — folder structure & page-by-page breakdown in [`frontend/README.md`](frontend/README.md)
- **Deep single-file reference** (for AI assistants and full onboarding): [`claude.md`](claude.md) and [`doc/system-report.txt`](doc/system-report.txt)
- **Sprint history:** [`sprint/`](sprint/)

---

## Table of Contents

1. [Modules & Features](#modules--features)
2. [Multi-Tenancy & Multi-Branch Architecture](#multi-tenancy--multi-branch-architecture)
3. [Permission System](#permission-system)
4. [Database Models](#database-models)
5. [Business Logic Rules](#business-logic-rules)
6. [Tech Stack](#tech-stack)
7. [Repository Layout](#repository-layout)
8. [Running Locally](#running-locally)
9. [Deployment](#deployment)

---

## Modules & Features

### Authentication
Email + OTP registration (`POST /auth/register` sends a 6-digit code, `POST /auth/verify-otp` completes account creation), JWT session in an HttpOnly cookie (7-day expiry), forgot-password / reset-password via emailed token link, and a `GET /auth/me` auto-login check on every page load. Staff accounts are created by an admin (never self-register) and can be deactivated (`isActive: false`), which blocks both login and every subsequent API call.

### Dashboard *(admin only)*
KPI cards (total revenue, expenses, net profit, invoice counts), three tabs — **P&L**, **Cash Flow**, **Analytics** — with revenue-over-time and expense-breakdown charts, payment-method breakdown, top products, and low-stock alerts. All figures respect the active branch filter.

### Branches *(admin only)*
Create/edit/delete branches, each with its own name, unique code (e.g. `CAI`, `ALX`), address, phone, and three independent atomic invoice counters (sales, warehouse, expense). A branch can override the workspace's VAT settings or inherit them. The branch detail page shows per-branch financial stats and lets an admin assign/unassign staff. A sidebar switcher lets an admin view "All Branches" or scope the whole UI to one.

### Team & Roles *(admin only, or staff with `team.manage`)*
Two sub-systems:
- **Roles** — named permission templates (admin-only to manage). Editing a role's permissions cascades immediately to every staff member currently assigned to it.
- **Staff** — individual accounts with a name, email, password, an optional role (or hand-picked permissions), and a `branchIds` array: `[]` makes them a cross-branch manager who sees everything, `[oneId]` locks them to a single branch, `[id, id, …]` lets them switch between several via the sidebar.

### Invoices (Sales)
Create invoices with dynamic line items (name, price, quantity), automatic subtotal/total/remaining calculation, partial-payment tracking, and status auto-detection (`paid` / `partial` / `unpaid`). Payment methods: cash, visa, transfer, wallet. Invoice numbers are branch-coded and sequential (`CAI-0001`), falling back to a timestamp-based ID when no branch is set. Creating an invoice auto-decrements matching product stock and logs a `StockMovement`. Client name/phone auto-links to an existing `Customer` or creates one. Filterable by status, payment method, date range, phone, and invoice number; exportable to PDF (single) or Excel (bulk).

### Products & Warehouse
Product catalog with selling price, cost price, stock level, unit, low-stock threshold, and an auto-generated SKU. The Warehouse page shows summary stats (total products, low/out-of-stock counts, stock value) and a full movement history (incoming/outgoing, with reason: purchase, invoice sale, manual adjustment, or return). Adding or removing stock manually creates both a `StockMovement` record and an auto-generated `WarehouseInvoice`. A product can optionally link to a default supplier.

### Warehouse Invoices *(read-only)*
Auto-generated documents for every stock movement (incoming purchases, outgoing dispatches, and initial stock set at product creation). Never created directly from the frontend — they're a byproduct of the Warehouse module and product creation. Supports payment recording for incoming/purchase documents and PDF export.

### Suppliers
Supplier directory (name, email, phone, place, notes) with per-branch uniqueness, a per-supplier financial dashboard (total cost, paid, outstanding, invoice counts by status), and full purchase-invoice history. Deleting a supplier unlinks it from any products rather than blocking the deletion.

### Customers
Customer directory (name, unique phone, notes) with lifetime stats (total invoices, billed, paid, outstanding) computed per customer, and a detail page listing their full invoice history. Customers are workspace-scoped, not branch-scoped — the same customer can appear across branches. Deleting a customer unlinks it from invoices rather than blocking the deletion. Exportable to Excel.

### Expenses
A **create-only** form (title, category, amount, paid amount, payment method, date, note) — intentionally has no list view of its own. Every expense automatically generates a linked `ExpenseInvoice`; the frontend never creates one directly. Categories: rent, utilities, salaries, supplies, transport, marketing, maintenance, other.

### Expense Invoices *(read-only)*
The list/detail view for everything the Expenses module has generated — filterable by status/category/date, with KPI stats and PDF export. No create or delete endpoint exists; expenses are the only entry point.

### Settings
Profile editing (name, password) for every user, plus workspace-level VAT configuration (enable/disable, rate, default 15%) for admins — with an optional per-branch override.

---

## Multi-Tenancy & Multi-Branch Architecture

**Workspaces.** Every `admin` user owns an independent workspace; their own `_id` **is** the workspace key. Staff belong to an admin's workspace via a `workspaceId` field. Every data document is tagged `{ userId: <workspaceId>, ... }`. The rule enforced throughout the backend: **never** query with a raw `req.user._id` — always resolve through `getWorkspaceId(req.user)` (`workspaceId || _id`), so the same code path is correct for both admins and staff.

**Branches.** A workspace can have multiple branches. Most data models carry an optional `branchId`. Three helpers in `backend/src/utils/workspaceHelper.js` drive all of it:

| Helper | Used for | Behavior |
|---|---|---|
| `getBranchFilter(user, activeBranchId)` | Reads | Admin with no branch chosen → sees everything; admin with a branch chosen → filtered. Staff with `branchIds: []` → cross-branch manager, sees everything. Staff with one branch → always locked to it. Staff with several → whichever they picked in the sidebar, falling back to their first branch if nothing valid was chosen. |
| `getWorkspaceQuery(user, activeBranchId)` | Reads | `{ userId, ...getBranchFilter(...) }` — the standard list-query shape used by nearly every controller. |
| `resolveBranchId(user, requestedBranchId)` | Writes | The write-side mirror of `getBranchFilter` — decides which `branchId` gets stamped on a newly created record, so writes land in the same branch reads are scoped to. |

Invoice numbering is atomic per branch: `Branch` documents carry three independent counters (`invoiceCounter`, `warehouseInvoiceCounter`, `expenseInvoiceCounter`), incremented via `$inc` and formatted as `{CODE}-0001`.

**On the frontend**, the active branch lives in `useApi()`'s `activeBranchId` (persisted to `localStorage`) and must be passed as `branchId` on every data-fetching and data-creating API call — otherwise the sidebar's branch switcher has no effect on that page.

---

## Permission System

24 permission keys across 8 groups. Admins bypass every check; staff need the exact key in their `permissions[]` array.

| Group | Keys |
|---|---|
| Reports | `reports.view` |
| Invoices | `invoices.view`, `invoices.create`, `invoices.delete` |
| Customers | `customers.view`, `customers.create`, `customers.edit`, `customers.delete` |
| Products | `products.view`, `products.create`, `products.edit`, `products.delete` |
| Warehouse | `warehouse.view`, `warehouse.manage`, `warehouse-invoices.view` |
| Expenses | `expenses.view`, `expenses.create`, `expenses.delete`, `expense-invoices.view` |
| Suppliers | `suppliers.view`, `suppliers.create`, `suppliers.edit`, `suppliers.delete` |
| Team | `team.manage` (staff CRUD — not role CRUD, which is admin-only) |

**Source of truth:** `backend/src/models/Role.js` exports `ALL_PERMISSIONS` / `PERMISSION_LABELS` / `PERMISSION_GROUPS`, validated on every role/staff permission write by `teamController.js`. Each backend route additionally enforces its own key via `checkPermission("key")` middleware. The frontend keeps a third, hand-maintained copy of the grouped keys in `team/page.jsx` for the checkbox UI — there's no shared import, so it can drift if a key is added without updating both sides.

Updating a role cascades its new permissions to every staff member assigned to it immediately (`User.updateMany`). Unassigning a staff member's role clears both `roleId` and `permissions` together.

---

## Database Models

| Model | Purpose |
|---|---|
| `User` | Admin or staff account — `role`, `workspaceId`, `roleId`, `permissions[]`, `branchIds[]`, `isActive`, workspace-level `vatEnabled`/`vatRate` |
| `Branch` | Per-workspace branch — unique name/code, 3 atomic invoice counters, optional VAT override |
| `Role` | Named permission template + the permission-system source of truth (see above) |
| `Customer` | Name, unique phone (per workspace), notes |
| `Supplier` | Name, email, phone, place, notes — unique per branch |
| `Product` | Name, auto-generated SKU, price, cost price, stock, unit, low-stock threshold, optional default supplier |
| `Invoice` | Sales invoice — items, total, paidAmount, remainingAmount, status, payment method; pre-save hook computes totals |
| `StockMovement` | Audit log for every stock change — type, quantity, reason, snapshot of cost/selling price, resulting stock |
| `WarehouseInvoice` | Auto-generated document per stock movement — incoming purchase or outgoing dispatch |
| `Expense` | Title, category, amount, paidAmount, remainingAmount, status, date |
| `ExpenseInvoice` | Auto-generated 1:1 document per `Expense`, kept in sync on every expense update |
| `Otp` | Short-lived registration OTP, keyed by email |

Full field-by-field definitions are in `backend/README.md` §Folder Structure and `claude.md` §5.

---

## Business Logic Rules

- **Invoice creation:** items → pre-save totals/status → branch-coded invoice number → stock auto-decremented per matching product with a `StockMovement` logged (`reason: "invoice_sale"`).
- **Stock incoming/outgoing:** updates `Product.stock`, logs a `StockMovement`, and auto-creates a matching `WarehouseInvoice` — never created directly by the frontend.
- **Expense creation:** creates the `Expense`, then automatically creates/syncs a linked `ExpenseInvoice` — the frontend never posts to expense-invoices directly.
- **Role permission cascade:** `PUT /team/roles/:id` updates the role, then `User.updateMany({ workspaceId, roleId }, { $set: { permissions } })` pushes the change to every staff member using it, instantly.
- **Staff role unassignment:** setting `roleId: null` also clears `permissions: []` in the same write — the two are never allowed to drift apart.
- **Payment status:** `paid` when `paidAmount >= total`, `partial` when `0 < paidAmount < total`, `unpaid` when `paidAmount === 0` — computed in a pre-save hook on `Invoice`, `Expense`, and `WarehouseInvoice` alike.
- **VAT:** workspace-level default (`User.vatEnabled` / `vatRate`, default 15%), with an optional per-branch override (`Branch.vatEnabled`/`vatRate`, `null` = inherit); applied at invoice-creation time based on the active branch.

---

## Tech Stack

| Layer | Stack |
|---|---|
| Backend runtime | Node.js (ESM), Express v5 |
| Database | MongoDB Atlas via Mongoose v9 |
| Auth | JWT (7-day) in an HttpOnly cookie, bcrypt password hashing |
| Email | Nodemailer via Gmail (OTP + password reset) |
| Frontend framework | Next.js 16 (App Router), React 19 |
| Styling | Tailwind CSS v4 |
| HTTP client | Axios (`withCredentials: true`) |
| Charts | Chart.js + react-chartjs-2 |
| Exports | jsPDF + jspdf-autotable (PDF), xlsx/SheetJS (Excel) |
| Animation | Framer Motion |
| Deploy | Vercel — backend and frontend as separate projects |

Neither package has an automated test suite.

---

## Repository Layout

```
erb-system-v1/
├── README.md              ← this file
├── claude.md               AI master reference — full system in one file
├── doc/
│   └── system-report.txt   full system report, same purpose as claude.md
├── sprint/                 sprint planning docs (bug lists, fix write-ups)
├── backend/                Express API — see backend/README.md
└── frontend/               Next.js dashboard — see frontend/README.md
```

For the complete file tree of each app (models, controllers, routes, pages, components), see `backend/README.md` and `frontend/README.md`.

---

## Running Locally

Both apps run independently and talk to each other over HTTP; run them in two terminals.

```bash
# Terminal 1 — backend (http://localhost:5000)
cd backend
npm install
cp .env.example .env      # fill in MONGO_URI, JWT_SECRET, EMAIL_USER, EMAIL_PASS
npm run dev

# Terminal 2 — frontend (http://localhost:3000)
cd frontend
npm install
cp .env.local.example .env.local   # NEXT_PUBLIC_API_URL=http://localhost:5000/api
npm run dev
```

Optional: `cd backend && npm run seed` wipes and reseeds a full demo workspace (admin login, 2 branches, 4 roles, 6 staff, suppliers, products, invoices, expenses — see `backend/src/seed.js` for exact credentials and counts).

---

## Deployment

Backend and frontend deploy as **separate** Vercel projects. `backend/vercel.json` routes all traffic to `src/server.js`. The frontend's `NEXT_PUBLIC_API_URL` must point at the deployed backend's `/api` base URL.

---

## Documentation Notes

- `claude.md` and `doc/system-report.txt` are kept in sync and describe the same system at the same level of detail — either is a complete single-file reference, including the full API route tables, exact model field lists, and known-dead-code notes that this README summarizes rather than repeats.
- `sprint/001sprint.txt` documents the most recent hardening pass (branch-scoping fixes, RBAC tightening); its outcomes are folded into `claude.md`'s changelog.

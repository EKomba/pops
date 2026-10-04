# Pop's BBQ — Web Ordering App

Online ordering and delivery site for Pop's BBQ Memphis Style (Menomonee Falls, WI). Customers browse the menu on a single-page marketing site, build a plate by picking meats and sides, pay through Stripe Checkout, and the kitchen works the queue from a password-protected employee console.

The repo is a two-part JavaScript application:

| Part | Folder | Stack | Dev port |
|---|---|---|---|
| Frontend | `client/` | React 19 + Vite, React Router | 5173 |
| Backend | `server/` | Express 5 REST API, MySQL | 5000 |

---

## Table of Contents

- [Technologies](#technologies)
- [Repository Layout](#repository-layout)
- [The Process (end-to-end flows)](#the-process-end-to-end-flows)
- [Frontend](#frontend)
- [Backend](#backend)
- [Database Schema](#database-schema)
- [Methods & Patterns Used](#methods--patterns-used)
- [Getting Started](#getting-started)
- [Known Quirks & Gotchas](#known-quirks--gotchas)

---

## Technologies

### Frontend

| Technology | Version | Used for |
|---|---|---|
| React | 19.1 | UI components, hooks-based state |
| React DOM | 19.1 | Rendering, `createRoot`, `StrictMode` |
| React Router DOM | 7.9 | Client-side routing (`BrowserRouter`) |
| Vite | 7.1 | Dev server, HMR, production bundling, `/api` proxy |
| `@vitejs/plugin-react` | 5.0 | JSX transform + Fast Refresh (Babel) |
| ESLint | 9.36 | Linting with `react-hooks` and `react-refresh` plugins |
| Plain CSS | — | Hand-written stylesheets, CSS custom properties, no framework |
| Google Fonts (Rye) | — | Western/BBQ display typeface, imported in `App.css` |

### Backend

| Technology | Version | Used for |
|---|---|---|
| Node.js (ESM) | — | Runtime; `"type": "module"` so everything uses `import` |
| Express | 5.1 | HTTP server, routers, JSON body parsing |
| `mysql2/promise` | 3.15 | MySQL driver, connection pooling, prepared statements |
| `cors` | 2.8 | Cross-origin access for the Vite dev server |
| `dotenv` | 17.2 | Loads secrets from `server/.env` |
| Stripe | 20.0 | Hosted Checkout Sessions for card payment |
| `@sendgrid/mail` | 8.1 | Primary transactional email provider for contact form |
| `nodemailer` | 7.0 | SMTP fallback / Ethereal test inbox for contact form |

### External services

- **Stripe Checkout** — hosted payment page; the app never handles card data.
- **Managed MySQL over TLS** — the pool is created with an explicit CA certificate (`server/ca (1).pem`), the pattern used by hosted MySQL providers.
- **FormSubmit.co** — the live contact form posts here directly from the browser (see [Known Quirks](#known-quirks--gotchas)).
- **SendGrid / SMTP** — wired up server-side as the intended replacement for FormSubmit.

---

## Repository Layout

```
pops/
├── .gitignore                  # ignores node_modules/ and .env everywhere
├── README.md
│
├── client/                     # ---------- FRONTEND ----------
│   ├── index.html              # Vite HTML entry; title, favicon, #root mount
│   ├── vite.config.js          # React plugin + dev proxy /api -> :5000
│   ├── eslint.config.js        # flat ESLint config
│   ├── package.json
│   ├── public/vite.svg
│   └── src/
│       ├── main.jsx            # React entry: createRoot + StrictMode
│       ├── App.jsx             # Router + global cart state + cart operations
│       ├── App.css             # ~1,130 lines: all shared/customer styling
│       ├── Menu.jsx            # API client (NOT a component) — menu/meats/sides fetchers
│       ├── assets/
│       │   ├── popsbbq-logo.png
│       │   └── BBQ-Pics/       # 9 food/location photos for hero, gallery, map
│       ├── components/         # Presentational + feature components
│       ├── pages/              # Route-level screens (+ their scoped CSS)
│       └── hooks/
│           └── useEmployeeAuth.js
│
└── server/                     # ---------- BACKEND ----------
    ├── server.js               # Express app; mounts 7 routers on :5000
    ├── db.js                   # MySQL connection pool (TLS, 10 connections)
    ├── ca (1).pem              # CA cert for the managed MySQL TLS handshake
    ├── server.log              # stray log file, not meaningful
    ├── package.json
    └── routes/
        ├── menuRoutes.js       # menu item CRUD
        ├── meatRoutes.js       # meat availability
        ├── sideRoutes.js       # side availability
        ├── orderRoutes.js      # order creation (transactional) + kitchen queue
        ├── authRoutes.js       # shared-password employee login
        ├── paymentRoutes.js    # Stripe Checkout Session creation
        └── contactRoutes.js    # contact email via SendGrid or SMTP
```

---

## The Process (end-to-end flows)

### 1. Customer ordering flow

This is the main path through the app, and it's worth reading carefully because **the order is not written to the database until after payment succeeds.**

```
  Home (/)
    │  BuildOrder loads GET /api/menu, /api/meats, /api/sides
    │  Customer clicks "+" on a menu item
    ▼
  OptionsModal
    │  Reads the item's description text to decide how many meats/sides
    │  are required (e.g. "choose 2 meats and 1 side"), then enforces
    │  exactly that count before letting the customer continue
    ▼
  Cart lives in App.jsx useState — one entry per unit while shopping
    │  "Checkout" calls consolidateOrder(): collapses identical
    │  item+meats+sides combinations into one row with a quantity
    ▼
  Checkout (/checkout)
    │  Validates name / phone / address
    │  Writes the whole order to localStorage["pending_order"]
    │  POST /api/payments/create-checkout-session { total_amount }
    ▼
  Stripe-hosted Checkout page  (customer leaves the site)
    │  success_url redirects back with ?session_id=...
    ▼
  OrderConfirmation (/order-confirmation)
    │  Reads localStorage["pending_order"], removes it,
    │  POST /api/orders  ──► order persisted, status = 'pending'
    │  (restores localStorage if the POST fails, so nothing is lost)
    ▼
  Order now visible to staff in the employee console
```

**Why localStorage?** The Stripe redirect leaves the SPA entirely, which destroys all React state. Parking the order in `localStorage` lets it survive the round trip, and deferring the `POST /api/orders` until the return trip means unpaid/abandoned carts never reach the database.

### 2. Employee / kitchen flow

```
  /employee-menu  or  /employee-orders
    │  useEmployeeAuth checks sessionStorage["employee_authenticated"]
    │  Not set? render a password gate instead of the page
    ▼
  POST /api/auth/login { password }
    │  Server compares against process.env.EMPLOYEE_PASSWORD
    │  Success → sessionStorage flag set, page renders
    ▼
  ┌─────────────────────────────┬──────────────────────────────────┐
  │ /employee-menu              │ /employee-orders                 │
  │ GET /api/menu/all           │ GET /api/orders/pending          │
  │     /api/meats/all          │   (order + items + meats + sides │
  │     /api/sides/all          │    assembled server-side)        │
  │                             │                                  │
  │ MenuManager edits locally,  │ "Mark as Complete" →             │
  │ "Save All" diffs against    │ PATCH /api/orders/:id/status     │
  │ the server copy and PATCHes │   { status: "completed" }        │
  │ only what changed; POST to  │ → card disappears from the queue │
  │ add, DELETE to remove       │                                  │
  └─────────────────────────────┴──────────────────────────────────┘
```

Availability toggles are the link between the two sides of the app: unchecking a meat sets `is_available = 0`, and the customer-facing `GET /api/meats` filters on that flag, so it vanishes from the ordering modal immediately.

### 3. Contact flow

`ContactSection` renders a validated form (including an email-confirmation field compared client-side) and submits it with a **native HTML POST to `https://formsubmit.co/...`**, using FormSubmit's `_next` parameter to bounce the visitor back to `/?contact=success`. The component reads that query parameter on mount, shows a thank-you banner, and scrubs the parameter out of the URL with `history.replaceState`.

The backend has a fuller implementation of the same feature at `POST /api/contact` (SendGrid, falling back to SMTP, falling back to an Ethereal test inbox) that the frontend does not currently call.

---

## Frontend

### Entry & shell

| File | Responsibility |
|---|---|
| `index.html` | Vite entry document. Sets the page title, logo favicon, and `#root`. Also loads the Square sandbox SDK via `<script>` — a leftover from an abandoned payment experiment. |
| `src/main.jsx` | Mounts `<App />` into `#root` with `createRoot` inside `<StrictMode>`. |
| `src/App.jsx` | The app's single source of truth. Declares the routes and **owns the global cart** (`order` state) plus the three functions that mutate it: `changeQuantity`, `removeFromOrder`, and `consolidateOrder` (which merges duplicate line items by comparing `id` plus JSON-stringified meat/side selections). |
| `src/Menu.jsx` | Despite the `.jsx` name, this is a **plain API client module** with no JSX: `getMenu()`, `getMeats()`, `getSides()`. Each `fetch`es `http://localhost:5000/api/...` and returns parsed JSON. |

### Routes

| Path | Component | Purpose |
|---|---|---|
| `/` | `pages/Home.jsx` | The marketing + ordering single page. Composes Header, Home, About, BuildOrder, Portfolio, Contact, Footer in order and passes cart props down to `BuildOrder`. |
| `/checkout` | `pages/Checkout.jsx` | Customer details form, editable line items, order total, and the "Place Order" button that hands off to Stripe. |
| `/order-confirmation` | `pages/OrderConfirmation.jsx` | Stripe landing page. Shows the session ID and performs the deferred `POST /api/orders` exactly once. |
| `/employee-menu` | `pages/EmployeeMenu.jsx` | Password-gated menu administration. Fetches menu/meats/sides in parallel with `Promise.all` and supplies save/create/delete/toggle handlers to `MenuManager`. |
| `/employee-orders` | `pages/EmployeeOrders.jsx` | Password-gated kitchen queue. Renders each pending order as a card with customer details, itemized meats/sides, total, and a "Mark as Complete" button. Tracks in-flight completions in a `Set` so individual buttons can disable independently. |

### Components

| File | What it does |
|---|---|
| `components/Header.jsx` | Sticky site header. Three behaviors: an `IntersectionObserver` that highlights the nav link for whichever section is on screen; a scroll listener that shrinks the header past 50px; and `scrollIntoView`-style smooth scrolling that offsets by the header's measured height (via `useRef`) so anchors don't land under it. On `/checkout` it swaps the whole nav for a single "Back To Home" link. |
| `components/HomeSection.jsx` | Hero panel with a background photo and a "Click to Order" button that programmatically clicks the `#services` nav link to reuse the header's smooth-scroll logic. |
| `components/AboutSection.jsx` | Restaurant story, address, hours list, and a clickable static map image. Both the heading link and the map build a Google Maps directions URL with `encodeURIComponent`. |
| `components/BuildOrder.jsx` | The ordering interface and the most logic-heavy component. Loads the menu on mount, groups items by `category` with `reduce`, and renders a two-column layout: menu on the left, live cart on the right. Derives an `expandedOrder` with `useMemo` (one row per unit, so quantity-3 shows as three editable rows), resolves meat/side IDs to names for display, supports per-row Edit (reopens the modal) and Remove (decrements the consolidated quantity), and navigates to `/checkout` with `useNavigate`. |
| `components/OptionsModal.jsx` | Meat/side picker shown when adding or editing an item. Its `getDinnerRules()` helper **derives the combo rules from the menu item's own description string** by finding the words "meat"/"side" and parsing the two characters before them as a number. Validates that the selection count matches exactly before adding to the cart. |
| `components/PortfolioSection.jsx` | Photo gallery; imports the eight BBQ images into an array and maps them into a CSS grid. |
| `components/ContactSection.jsx` | Controlled contact form (single `form` state object updated by `name`), email-match validation, FormSubmit.co submission, and success-banner handling from the `?contact=success` query param. |
| `components/Footer.jsx` | Logo, address, Facebook link, copyright. |
| `pages/MenuManager.jsx` | The editor table used by `/employee-menu`. Keeps `local`/`localMeats`/`localSides` draft copies synced from props via `useEffect`, so edits are staged in memory. `saveAll()` **diffs drafts against the original props** and sends a PATCH only for genuinely changed rows and only for changed fields. The "Save All" button gets a `dirty` class whenever any draft differs, giving unsaved-changes feedback. Also renders the add-item row and per-row delete with a `confirm()` guard, grouping menu items under their category headings. |

### Hooks

| File | What it does |
|---|---|
| `hooks/useEmployeeAuth.js` | Shared login logic for both employee pages. Seeds `isAuthenticated` from `sessionStorage` (lazy `useState` initializer) so a refresh doesn't force a re-login, exposes `handleLogin` (POSTs to `/api/auth/login`), `logout`, and the password input / error / loading state. Using `sessionStorage` means the session ends when the browser tab closes. |

### Styling

| File | Scope |
|---|---|
| `src/App.css` | ~1,130 lines covering everything customer-facing: the Rye font import, the `:root` color variables (red/black/white BBQ palette), header and nav, panels, gallery grid, contact form with floating labels, the `bo-*` BuildOrder classes, the `m-*` modal classes, the `checkout-*` classes, confirmation card, footer, and several `@media` breakpoints at 800/768/600/480px. |
| `src/pages/EmployeeMenu.css` | 220 lines scoped to the menu editor: table layout, the add-item row widths, availability checkboxes, dirty-state save button, and the login overlay/box. |
| `src/pages/EmployeeOrders.css` | 160 lines scoped to the kitchen queue: order card, header, customer block, item details, total, and the complete button's disabled state. |

---

## Backend

### Core

| File | Responsibility |
|---|---|
| `server.js` | The whole server setup in 25 lines: enable `cors()`, enable `express.json()`, mount the seven routers under their `/api/*` prefixes, and listen on port 5000. No error-handling middleware, no logging middleware. |
| `db.js` | Creates and exports a single shared `mysql2/promise` **connection pool** (limit 10, unbounded queue) from `DB_*` environment variables, with TLS configured by reading `./ca (1).pem` off disk. Calls `dotenv.config()` — and because every route module imports `db.js`, this is what loads the `.env` file for the entire process. |
| `ca (1).pem` | The certificate authority bundle the MySQL TLS handshake validates against. |

### API reference

#### `routes/menuRoutes.js` → `/api/menu`
| Method | Path | Purpose |
|---|---|---|
| `GET` | `/` | Customer menu — only `is_available = TRUE`, ordered by category then id. Returns `{ items: [...] }`. |
| `GET` | `/all` | Admin menu — every item regardless of availability. Returns a bare array. |
| `POST` | `/` | Create an item. Validates that `name` is a non-empty string, `price` parses to a non-negative number, and `category` is a string; responds `201` with the inserted row. |
| `PATCH` | `/:id` | Partial update of `name` / `price` / `is_available`. Builds the `SET` clause dynamically from only the keys present in the body and `400`s if none were sent. |
| `DELETE` | `/:id` | Validates the id is a positive integer, `404`s if the row is missing, then deletes and returns the deleted row. |

#### `routes/meatRoutes.js` → `/api/meats` and `routes/sideRoutes.js` → `/api/sides`
Two near-identical routers for the combo options:
| Method | Path | Purpose |
|---|---|---|
| `GET` | `/` | Available options only — what the ordering modal shows customers. |
| `GET` | `/all` | Every option — what the employee editor shows. |
| `PATCH` | `/:id` | Toggle `is_available`, returning the updated row. |

#### `routes/orderRoutes.js` → `/api/orders`
| Method | Path | Purpose |
|---|---|---|
| `POST` | `/` | **Creates an order inside a database transaction.** Checks out a dedicated connection, `beginTransaction()`, inserts the `orders` row, then for each line item inserts an `order_items` row and fans out its `chosenMeats`/`chosenSides` arrays into the `order_meats`/`order_sides` join tables. Commits and releases on success; **rolls back and releases on any error**, so a half-written order can never survive. Returns the new `order_id`. |
| `GET` | `/pending` | Builds the kitchen queue. Selects `status = 'pending'` orders newest-first, then for each order fetches its items (joined to `menu_items` for the name) and for each item fetches its meats and sides, flattening them to name arrays. The frontend therefore receives one fully nested payload and needs no follow-up requests. |
| `PATCH` | `/:id/status` | Sets an order's `status` (used to mark orders `completed`). |

#### `routes/authRoutes.js` → `/api/auth`
| Method | Path | Purpose |
|---|---|---|
| `POST` | `/login` | Compares the submitted password against `EMPLOYEE_PASSWORD`. Returns `503` if the variable isn't configured (and warns at startup), `400` if no password was sent, `401` on mismatch, `{ success: true }` on match. A single shared staff password — no user accounts, no tokens. |

#### `routes/paymentRoutes.js` → `/api/payments`
| Method | Path | Purpose |
|---|---|---|
| `POST` | `/create-checkout-session` | Converts the dollar total to integer cents with `Math.round(total * 100)` and creates a Stripe Checkout Session (`mode: "payment"`, card only) as a single "Pops BBQ Order" line item. Returns the hosted `session.url` for the browser to redirect to. `success_url` / `cancel_url` point at `http://localhost:5173`. |

#### `routes/contactRoutes.js` → `/api/contact`
| Method | Path | Purpose |
|---|---|---|
| `POST` | `/` | Sends the contact form as an email. Requires `email` and `message`. **Three-tier provider strategy:** use SendGrid if `SENDGRID_API_KEY` is set; otherwise use SMTP if `SMTP_HOST`/`SMTP_USER`/`SMTP_PASS` are set; otherwise create a throwaway Ethereal test account and return a `preview` URL so the mail can be inspected in development. Builds both text and HTML bodies and sets `replyTo` to the visitor. |

---

## Database Schema

There are no migration files in the repo, so this schema is reconstructed from the queries in `server/routes/`. Seven tables:

**Catalog tables** — what the restaurant sells, all editable from the employee console:

```
menu_items                      meats                  sides
├── id            PK            ├── id            PK   ├── id            PK
├── name                        ├── name               ├── name
├── description  ← combo rules  └── is_available        └── is_available
├── price           parsed from
├── category        this text
└── is_available
```

**Order tables** — an immutable record of what was bought:

```
orders
├── id                PK
├── customer_name
├── customer_phone
├── customer_address
├── total_amount
├── status                 'pending' by default, set to 'completed' by staff
└── created_at
      │
      │  one order has many items
      ▼
order_items
├── id                PK
├── order_id          FK → orders.id
├── menu_item_id      FK → menu_items.id
├── price                  captured at order time
└── quantity
      │
      │  one item has many meats and many sides
      ├──────────────────────────┐
      ▼                          ▼
order_meats                       order_sides
├── order_item_id  → order_items  ├── order_item_id  → order_items
└── meat_id        → meats        └── side_id        → sides
```

The two join tables are what make a BBQ combo representable: one `order_items` row ("2 Meat Dinner") can own several `order_meats` and `order_sides` rows. Storing `price` on `order_items` means later menu price changes don't rewrite order history.

---

## Methods & Patterns Used

### Frontend patterns

- **Lifted state / single source of truth** — the cart lives in `App.jsx` and is passed down as props, so Home, BuildOrder, and Checkout all read and write the same array without a state library.
- **Derived state with `useMemo`** — `expandedOrder` in `BuildOrder` turns the compact cart into one row per unit on every change instead of storing both shapes.
- **Custom hook for cross-page logic** — `useEmployeeAuth` keeps identical login behavior on two pages in one place.
- **Lazy `useState` initializer** — `useState(() => sessionStorage.getItem(...))` reads storage once on mount rather than on every render.
- **Draft/commit editing with field-level diffing** — `MenuManager` stages edits locally, then compares drafts to the originals and sends only changed rows and changed fields, which keeps the PATCH payloads minimal and powers the dirty-state button.
- **Controlled components with inline validation** — Checkout and ContactSection hold every input in state and clear each field's error as soon as the user types in it.
- **Parallel data loading** — `Promise.all` for the three independent admin fetches.
- **State survival across an external redirect** — `localStorage` carries the pending order through the Stripe hop, with restore-on-failure so a network error doesn't lose the order.
- **Per-item async tracking with a `Set`** — `completingOrders` lets one order's button show "Completing…" without disabling the rest.
- **`IntersectionObserver` for scroll-spy** — native browser API instead of scroll-position math for active nav highlighting.
- **`useRef` for DOM measurement** — reading the live header height so smooth-scroll offsets stay correct when the header shrinks.
- **Route-aware navigation** — the header swaps its nav for a back link on the checkout page.
- **Data-driven combo rules** — `getDinnerRules()` parses requirements out of the menu item's description, so staff can define a new combo from the admin UI alone, with no code change.
- **Grouping with `reduce`** — menu items bucketed by category for rendering in both the customer and admin views.
- **Asset imports through the bundler** — images are `import`ed so Vite hashes and optimizes them.
- **CSS custom properties + scoped stylesheets** — a shared palette in `:root`, with per-page CSS files for the employee screens.

### Backend patterns

- **Modular routers** — one `express.Router()` per resource, mounted by prefix in `server.js`, so `server.js` stays a 25-line manifest of the API surface.
- **Connection pooling** — one shared pool (limit 10) created at startup and reused for every request, rather than connecting per query.
- **Prepared statements everywhere** — all values go through `db.execute(sql, [params])` placeholders, which is the project's SQL-injection defense. Note that `menuRoutes.js` builds its PATCH `SET` clause dynamically, but only from a hard-coded allowlist of column names.
- **ACID transactions for multi-table writes** — order creation checks out its own connection and wraps the orders → order_items → order_meats/order_sides inserts in a transaction with rollback on error.
- **Server-side aggregation** — `/api/orders/pending` assembles the complete nested order payload so the client makes one request instead of N+1.
- **Two-tier read endpoints** — every catalog resource exposes a filtered `GET /` for customers and an unfiltered `GET /all` for staff, which keeps the availability rule on the server.
- **Soft-delete-style availability flags** — `is_available` hides items without destroying rows or breaking historical orders.
- **Partial updates** — PATCH handlers treat `undefined` fields as "leave alone", so the client can send just the delta.
- **Graceful provider degradation** — the contact route tries SendGrid, then SMTP, then an Ethereal test inbox, so the feature works in development with zero configuration.
- **Fail-safe configuration checks** — the auth route warns at startup and returns `503` rather than silently accepting any password if `EMPLOYEE_PASSWORD` is missing.
- **Secrets in environment variables** — `dotenv` plus a `.gitignore`d `.env`; no credentials in source.
- **TLS to the database** — the pool validates the server certificate against a bundled CA.
- **Hosted payment handoff** — Stripe Checkout means no card data ever touches this server, which keeps PCI scope minimal.
- **Currency as integer cents** — dollar amounts are converted with `Math.round(x * 100)` before going to Stripe, avoiding float rounding errors.

### Development workflow

The project was built by a team of ~6 (89 commits since 2025-10-20) working on **feature branches merged into `main` through GitHub pull requests** — visible in the history as branches like `ethan77`, `alex123`, and `Zack-Branch`.

---

## Getting Started

### Prerequisites

- Node.js 18+ (ESM and Express 5)
- Access to a MySQL database with the [schema above](#database-schema)
- A Stripe account (test mode is fine)

### 1. Backend

```bash
cd server
npm install
```

Create `server/.env`:

```ini
# MySQL
DB_HOST=your-mysql-host
DB_PORT=3306
DB_USER=your-user
DB_PASSWORD=your-password
DB_NAME=your-database

# Employee console (shared staff password)
EMPLOYEE_PASSWORD=choose-a-password

# Stripe
STRIPE_SECRET_KEY=sk_test_...

# Contact email — all optional; omit to use the Ethereal test inbox
SENDGRID_API_KEY=
CONTACT_DESTINATION=where-contact-mail-should-go@example.com
CONTACT_FROM=verified-sender@example.com
SMTP_HOST=
SMTP_PORT=587
SMTP_SECURE=false
SMTP_USER=
SMTP_PASS=
```

Start it **from inside `server/`** — `db.js` reads the certificate with the relative path `./ca (1).pem`:

```bash
npm start           # node server.js  →  http://localhost:5000
```

Expect `MySQL Pool Initialized` followed by `Server running on port 5000`.

### 2. Frontend

In a second terminal:

```bash
cd client
npm install
npm run dev         # → http://localhost:5173
```

Other client scripts: `npm run build` (production bundle to `dist/`), `npm run preview` (serve the build), `npm run lint`.

### 3. Try it

- **Customer:** <http://localhost:5173> → scroll to Order → add an item → Checkout → pay with Stripe test card `4242 4242 4242 4242`.
- **Staff:** <http://localhost:5173/employee-orders> → enter `EMPLOYEE_PASSWORD` → the order you just placed should be waiting.

---

## Known Quirks & Gotchas

Things that will confuse the next person reading this code:

1. **`client/src/Menu.jsx` is not a component.** It's an API client module and contains no JSX. A `.js` name would be more honest.
2. **Two different ways to call the API.** The customer path (`Menu.jsx`, `Checkout.jsx`, `OrderConfirmation.jsx`) uses absolute `http://localhost:5000/...` URLs, while the employee path (`useEmployeeAuth`, `EmployeeMenu`, `EmployeeOrders`) uses relative `/api/...` URLs through the Vite proxy in `vite.config.js`. **Only the relative calls will work in production** — the absolute ones hardcode localhost and must be changed before deploying.
3. **Stripe redirect URLs are hardcoded to localhost.** `paymentRoutes.js` sets `success_url`/`cancel_url` to `http://localhost:5173`; these need to come from an environment variable for any real deployment.
4. **The backend contact route is dead code today.** `ContactSection.jsx` posts straight to FormSubmit.co, so the SendGrid/nodemailer implementation in `contactRoutes.js` is never reached. Switching the form to `POST /api/contact` would activate it.
5. **Square is a leftover.** `index.html` still loads the Square sandbox SDK and `server/package.json` still lists the `square` package, but nothing imports or uses either. Both are safe to remove.
6. **`stripe` is in the wrong dependency section.** `server/package.json` lists `stripe` under `devDependencies` even though `paymentRoutes.js` requires it at runtime, so a `npm install --production` deploy would crash on startup. It belongs in `dependencies`.
7. **`dotenv` loading depends on import order.** `server.js` never calls `dotenv.config()` itself; it happens as a side effect of `db.js` being imported by the first router. `authRoutes.js` and `paymentRoutes.js` read `process.env` at module scope and only work because they're imported *after* that. Calling `dotenv.config()` at the top of `server.js` would make this robust instead of accidental.
8. **Combo rules are parsed from English.** `getDinnerRules()` in `OptionsModal.jsx` reads the two characters preceding the word "meat"/"side" in an item's description. It's flexible for staff but brittle: a description that says "ten meats", or omits the number, or has no description at all (which throws on `.toLowerCase()`), will not behave as expected.
9. **Employee auth is a shared password, not a real session.** The password check happens server-side, but a successful login only sets a `sessionStorage` flag in the browser and the API routes carry no auth middleware — so the employee console is gated in the UI rather than at the API layer. Adding token-based auth middleware is the top hardening priority before any public deployment.
10. **`server/server.log`** contains the single line `stdout is not a tty` and can be deleted — `client/.gitignore` ignores `*.log`, but the server folder has no such rule.

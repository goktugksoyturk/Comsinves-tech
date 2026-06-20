# Architecture

A technical overview of how **Comsinves** is structured and how data flows through it.

<p align="center">
  <img src="architecture.png" alt="Comsinves architecture diagram" width="720" />
</p>

---

## 1. Frontend structure

Comsinves is a **Next.js (App Router)** application written in **TypeScript** and **React 19**, styled with **Tailwind CSS v4** and a small custom design system.

```
src/
├── app/                     # App Router: routes, layouts, API route handlers
│   ├── api/                 # Server-side route handlers (market-data proxy)
│   ├── auth/                # OAuth callback & password recovery
│   ├── dashboard/           # Portfolio overview
│   ├── assets/              # Per-asset detail & management
│   ├── transactions/        # Buy/sell history
│   ├── performance/         # Performance-over-time charts
│   ├── insights/            # Allocation, momentum & risk signals
│   ├── targets/             # Goals & milestones
│   ├── news/                # Asset news feed
│   ├── settings/            # Currency, theme, language
│   ├── onboarding/          # Guided first-run setup
│   ├── login/               # Auth screen (login / sign up)
│   ├── layout.tsx           # Root layout + global providers
│   └── page.tsx             # Public landing page
├── components/              # UI components (app, dashboard, landing, navigation…)
├── lib/                     # Business logic & integrations
│   ├── supabase.ts          # Supabase client + auth helpers
│   ├── cloudStorage.ts      # Cloud (+ localStorage fallback) persistence
│   ├── calculations.ts      # Portfolio math: value, P&L, allocation, insights
│   ├── prices.ts            # Price fetching & currency conversion
│   ├── i18n.ts              # English / Turkish translations
│   └── ...
├── types/portfolio.ts       # Core domain types
└── hooks/                   # Reusable hooks (e.g. toasts)
```

**Routing model.** Public routes (`/`, `/login`, `/privacy`, `/terms`, `/auth/*`) are reachable without a session. The application routes (`/dashboard`, `/assets`, `/transactions`, `/performance`, `/insights`, `/targets`, `/news`, `/settings`) render inside an authenticated app shell that guards access and provides the navigation chrome.

**State management.** Global portfolio state lives in a single **React Context** (a `PortfolioProvider`). It holds the portfolio data and exposes derived calculations to every screen, so any change to assets, prices, or transactions immediately re-renders dependent views. There is no external state library — Context plus memoized derivations is enough for the app's size.

**Visuals.** Charts use **Recharts**. The marketing landing page uses **Three.js / React Three Fiber**, **Framer Motion**, and **GSAP** for animated 3D scenes, kept separate from the data-heavy app screens.

---

## 2. Backend & database flow

Comsinves has **no separate backend service**. Its "backend" is two things:

1. **Supabase** — managed PostgreSQL + Auth, accessed directly from the client via `@supabase/supabase-js`.
2. **Next.js API route handlers** (`src/app/api/*`) — lightweight serverless functions that proxy external market-data providers.

### Database schema (Supabase / PostgreSQL)

| Table | Key columns | Purpose |
|---|---|---|
| `portfolios` | `user_id` (PK, FK → `auth.users`), `payload` (JSONB), `updated_at` | Stores the entire portfolio (assets, transactions, snapshots, multiple portfolios) as a single JSON document per user. |
| `user_preferences` | `user_id` (PK, FK → `auth.users`), `full_name`, `portfolio_name`, `theme`, `preferred_currency`, `updated_at` | Per-user display settings. |

**Row-Level Security (RLS)** is enabled on these tables. Policies restrict every `select` / `insert` / `update` to rows where `user_id = auth.uid()`, so a user can only ever read or write their own data — enforced by the database, not just the client.

**Why JSONB?** The portfolio model (assets with categories, transactions, daily snapshots, and support for multiple portfolios per user) evolves frequently. Storing it as a JSONB `payload` avoids a churn of schema migrations while still living in Postgres with RLS protection.

### Persistence flow

```
PortfolioProvider  ──save──▶  cloudStorage  ──▶  Supabase `portfolios` (if signed in)
                                          └──▶  localStorage (fallback / guest)
        ▲                                          │
        └──────────────── load (with safe merge) ──┘
```

A merge guard ensures that an empty or failed load never overwrites existing good data.

---

## 3. Authentication flow

Authentication is handled by **Supabase Auth**, supporting **email/password** and **Google OAuth**, plus password recovery.

**Email / password**

1. User submits the login or sign-up form (`/login`).
2. Supabase verifies credentials and returns a session; the session is persisted and auto-refreshed by the Supabase client.
3. The authenticated app shell renders protected routes.

**Google OAuth**

1. User clicks **Continue with Google**.
2. Supabase redirects to Google, then back to **`/auth/callback?code=…`**.
3. The callback route exchanges the code for a session, syncs preferences, and routes new users to `/onboarding`.

**Session & route protection.** On load, the app checks for an active session; unauthenticated visitors to protected routes are redirected to the login screen. RLS provides a second, server-enforced layer so data access is safe even independent of client-side checks.

> A localStorage-based guest/offline mode also exists for trying the app without an account. It is intended for demo use only and is **not** a substitute for the Supabase-backed authentication described above.

---

## 4. External API flow

Market data is never fetched directly from the browser. Instead, the client calls **internal Next.js route handlers**, which call the upstream providers. This keeps caching, retries, and fallbacks in one place.

| Route | Source(s) | Returns |
|---|---|---|
| `/api/coingecko-price` | CoinGecko | Current crypto price + 24h change |
| `/api/coingecko-history` | CoinGecko | Historical crypto price on a date |
| `/api/quote` | Yahoo Finance → Stooq → fallback | Current stock/BIST quote + change |
| `/api/stock-history` | Yahoo Finance | Historical stock price |
| `/api/asset-series` | CoinGecko / Yahoo | Price series for charts (7/30/90/365d) |
| `/api/asset-news` | RSS feeds | Recent news items for an asset |

**Resilience patterns built into these routes:**

- **In-memory caching** with TTLs — short for live prices, long for historical data (which doesn't change).
- **Rate-limit handling** — exponential backoff and retry on `429` responses (notably from CoinGecko's free tier).
- **Multi-provider fallback** — quotes try Yahoo Finance, then Stooq, then a safe fallback, so a single provider outage doesn't blank the dashboard.

**Currency conversion** between USD / TRY / EUR is applied for display. (Conversion currently uses static rates; live FX is a planned improvement.)

---

## 5. Data lifecycle

End-to-end, a holding moves through the system like this:

1. **Add an asset / transaction.** The user records a holding or a buy/sell in the UI. It's added to the portfolio state in the `PortfolioProvider`.
2. **Fetch live data.** The client requests current and historical prices via the `/api/*` routes, which cache and normalize provider responses.
3. **Compute metrics (client-side).** `calculations.ts` derives portfolio value, cost-basis, unrealized & realized P&L, allocation by class, per-asset breakdown, and rule-based risk insights (concentration, diversification, momentum).
4. **Snapshot.** A daily portfolio-value snapshot is recorded, building the time series that powers the performance charts.
5. **Persist.** State is saved to the Supabase `portfolios` table (and localStorage as a fallback), scoped to the user by RLS.
6. **Render.** Screens read derived values from Context and render them with Recharts and the component library.

This loop repeats reactively: any change to holdings, prices, or settings re-derives the metrics and updates every dependent view.

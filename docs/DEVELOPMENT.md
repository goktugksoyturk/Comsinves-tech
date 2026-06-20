# Development

How to configure, run, and deploy **Comsinves** locally.

> The application source is maintained in a **private repository**. This document records the setup so the project's engineering can be understood and reproduced by anyone with access.

---

## Prerequisites

- **Node.js 20+** (Next.js 16 / React 19)
- **npm** (or pnpm / yarn)
- A **Supabase** project (free tier is sufficient) for auth and the database

---

## Environment variables

Create a `.env.local` file in the app directory with the following keys. These are read at build/runtime via `process.env`:

| Variable | Required | Description |
|---|---|---|
| `NEXT_PUBLIC_SUPABASE_URL` | ✅ | Your Supabase project URL |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | ✅ | Supabase anonymous (public) key |
| `NEXT_PUBLIC_SITE_URL` | ✅ | Site origin used for OAuth redirects (e.g. `http://localhost:3000` locally, `https://comsinves.tech` in production) |

```bash
# .env.local
NEXT_PUBLIC_SUPABASE_URL=https://YOUR-PROJECT.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=YOUR-ANON-KEY
NEXT_PUBLIC_SITE_URL=http://localhost:3000
```

> All three are `NEXT_PUBLIC_*` values and safe to expose to the browser. The Supabase **anon key** is designed to be public — data is protected by **Row-Level Security**, not by hiding the key. Never commit a service-role key or any private credential.

---

## Supabase setup

1. Create a new project in the [Supabase dashboard](https://supabase.com/).
2. Run the project's `supabase-schema.sql` in the **SQL Editor** to create the `portfolios` and `user_preferences` tables and their **RLS policies**.
3. Under **Authentication → Providers**, enable **Email** and **Google** (add your Google OAuth client ID/secret).
4. Under **Authentication → URL Configuration**, add your redirect URLs (e.g. `http://localhost:3000/auth/callback` and the production equivalent).
5. Copy the project URL and anon key into `.env.local`.

---

## Install & run

```bash
# install dependencies
npm install

# start the dev server (http://localhost:3000)
npm run dev

# build for production
npm run build

# run the production build locally
npm run start

# lint
npm run lint
```

A Turbopack dev script is also available for faster local builds (`npm run dev:turbo`, if defined).

---

## Project layout (high level)

```
src/app/        # routes, layouts, and /api route handlers
src/components/ # UI components
src/lib/        # Supabase client, persistence, price fetching, calculations, i18n
src/types/      # domain types
supabase-schema.sql   # database tables + RLS policies
```

See **[ARCHITECTURE.md](ARCHITECTURE.md)** for how these fit together.

---

## Deployment

The app deploys to **Vercel**:

1. Import the repository into Vercel.
2. Add the three environment variables above in **Project Settings → Environment Variables** (set `NEXT_PUBLIC_SITE_URL` to the production domain).
3. Vercel builds with `npm run build` and serves the App Router output, including the `/api/*` route handlers as serverless functions.
4. Make sure the production domain's `/auth/callback` URL is registered in Supabase's auth redirect settings.

**Analytics.** Vercel Analytics and Speed Insights are wired into the root layout and activate automatically on Vercel deployments.

---

## Notes & gotchas

- **Free market-data tiers rate-limit aggressively.** The `/api/*` routes cache responses and back off on `429`s; expect occasional fallback data during heavy use.
- **Currency conversion uses static rates** today — fine for display, but not a live FX source. This is a known limitation (see the README's Future Improvements).
- **No automated tests yet.** Adding a test suite and CI is on the roadmap.

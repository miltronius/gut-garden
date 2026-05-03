# Gut Garden

A food tracker app focused on the "30 plants a week" gut microbiome principle (American Gut Project / Tim Spector / ZOE research). Tracks plant diversity, with rich macro/micronutrient breakdowns and (eventually) AI camera-based food recognition.

## Tech Stack

- **Framework**: TanStack Start (React, Vite, file-based routing, server functions)
- **Hosting**: Cloudflare Workers (deploy via `wrangler`, not Cloudflare Pages)
- **Backend**: Supabase (Postgres, Auth, Storage, Edge Functions) — region `eu-central-2` (Zurich)
- **Styling**: Tailwind CSS
- **Data fetching**: TanStack Query
- **Vision API** (Level 2+): Anthropic Claude Sonnet — loosely coupled, swappable
- **Nutrition data**: Open Food Facts + USDA seed data
- **Mobile strategy**: PWA first, Capacitor wrapper later if app stores warranted (not React Native rewrite)

## Architecture Principles

- **Backend layer via TanStack Start server functions.** The browser does NOT use the Supabase client directly. Server functions verify the user's session JWT and act on their behalf using a server-side Supabase client.
- **`service_role` key is server-only** — never reaches the browser. Used only inside server functions for admin operations.
- **RLS is enabled on every table** as defense-in-depth, even though server functions enforce auth in user-space.
- **Anthropic API key is server-only**, called from server functions or Supabase Edge Functions.
- **Loosely coupled** to swappable services: the vision API and even the hosting layer (Cloudflare today, Infomaniak possible later for Swiss data sovereignty) should not be load-bearing in the architecture.

## Schema (locked-in design)

Full schema doc lives at `docs/SCHEMA.md` (to be ported into the repo).

Tables:

- `plants` — master reference of edible plants. Read-only for users, seeded by us.
- `foods` — separate from `plants` to handle credit-tier rules (whole apple = full credit, apple juice = no credit) while still mapping to a single plant.
- `meals` — immutable history of logged meals.
- `meal_ingredients` — items in a meal.
- `templates` + `template_ingredients` — reusable meal definitions (parallel tables to meals; deliberately not unified because lifecycles differ — meals are immutable history, templates are mutable definitions).
- `user_profiles` — extends `auth.users`. Anonymous users supported via Supabase `signInAnonymously()`; upgrade flow is just an UPDATE on the row, every FK keeps working.
- `friend_codes` — separate table for fast lookup of "paste a code to add a friend" path (Level 4).

## Phase 1 Plan (in order)

1. **Foundation** (current step) — Scaffold TanStack Start, TypeScript, Tailwind, Supabase client split (browser/server), env structure, `wrangler.jsonc` for Cloudflare Workers.
2. **Database** — Migrations from the schema doc, RLS policies, seed scripts for `plants` + `foods`.
3. **Auth** — Anonymous signin (so friends can use it instantly), optional email upgrade.
4. **Port the prototype** — React prototype built in claude.ai → real pages with Supabase persistence. Prototype features: progress ring dashboard for 30 plants, plant logging with search and category filtering, macro/micro nutrition breakdowns, camera placeholder.
5. **PWA polish** — Manifest, service worker, install prompt, offline-friendly logging.

Levels 2–5 are explicitly deferred:
- Level 2: AI camera food logging
- Level 3: Rich nutrition insights and configurable goals (e.g., 100 plants/month)
- Level 4: Social features (friend codes, group leaderboards) and scaling
- Level 5: Native mobile via Capacitor (only if PWA hits real limits)

## Conventions

- TypeScript strict mode.
- Server functions for any data access — never call Supabase directly from the browser.
- Anonymous-first auth: friends shouldn't need to sign up to try the app.
- Cost-conscious: free tiers everywhere possible (Cloudflare Workers free tier, Supabase free tier early on).
- Single-file React components for small things; folder-per-component when files start having siblings.

## Working With This Codebase

- When scoping new work: lock in Phase 1 before pulling Level 2 features forward. The boring part — reliable logging, clear dashboard, fast UX — is what makes daily use stick. AI camera comes after the core works.
- Cloudflare Workers runtime is V8 isolates with web standards APIs (not full Node). Most modern packages work via `nodejs_compat` flag, but verify before adding heavy Node-only dependencies.
- Supabase project lives in Zurich (`eu-central-2`) for Swiss/EU user data residency.
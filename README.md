# Vendoura Hub Accelerator

Platform management system for the Vendoura Hub founder acceleration program —
program oversight, founder tracking, milestone reporting, and cohort analytics
for founders and admins.

Built from the [System Logic Blueprint](https://www.figma.com/design/yjNFYqqC19pCF6Ab574M5J/System-Logic-Blueprint)
design system.

## The problem

The accelerator runs structured cohorts: founders commit to weekly activity, submit
reports, hit milestones, and are tracked against program KPIs. Spreadsheets can't do
this safely — and a demo-only app that loses data on refresh can't either. The
Accelerator needed **real authentication, persistent per-founder data, and role-based
access control**, not a local prototype.

## What it provides

- **Founder workspace** — onboarding, dashboard, weekly commits, reports, profile management
- **Admin console** — founder directory, cohort analytics, weekly tracking, system settings
- **Program management** — applications, waitlist, milestones, revenue tracking
- **Secure auth** — email/password signup and sign-in for founders *and* admins
- **Row-level security** — the database enforces that founders only see their own data
- **Profile photos** — file uploads to Supabase Storage (validated, size-limited)

## Architecture

```
┌───────────────────────────────────────────────────────────┐
│  React 18 + Vite + TypeScript + Tailwind + Radix UI (SPA)  │
│  ├─ src/pages/founder   (dashboard, commit, report, ...)   │
│  ├─ src/pages/admin     (directory, analytics, settings)   │
│  └─ src/lib             (authManager, founderService,      │
│                          adminService, profilePhotoService) │
└───────────────┬───────────────────────────────────────────┘
                │  @supabase/supabase-js (anon key + JWT)
┌───────────────▼───────────────────────────────────────────┐
│  Supabase (backend-as-a-service)                           │
│  ├─ Auth           email/password, JWT sessions            │
│  ├─ Postgres       founder_profiles, admin_users,          │
│  │                  weekly_commits, weekly_reports,         │
│  │                  applications, waitlist, system_settings │
│  ├─ Row Level Security  per-user + admin access policies    │
│  └─ Storage        profile-photos bucket (public reads)     │
└───────────────────────────────────────────────────────────┘
```

**Auth model.** `src/lib/authManager.ts` is the single source of truth for sessions:
it calls Supabase Auth, resolves the role (`founder` vs `admin`) from the
`founder_profiles` / `admin_users` tables, and verifies the active user against the
stored session context. The database, not the client, decides what a role can access.

**Client-side storage is a cache, not a datastore.** `localStorage` holds only
non-authoritative session context (active role, device id, admin session cache) and
the Supabase auth token. All real data lives in Supabase Postgres.

## Data model (Supabase)

| Table | Purpose | RLS |
|-------|---------|-----|
| `founder_profiles` | Founder identity, business, onboarding, stage | Own row only |
| `admin_users` | Admin identity + `admin_role` | Admins / super admins |
| `weekly_commits` | Weekly activity submissions | Own row only |
| `weekly_reports` | Weekly progress reports | Own row only |
| `applications` | Public program applications | Insert by anyone, read by admins |
| `waitlist` | Public waitlist signups | Insert by anyone |
| `system_settings` | Program status + settings | Admins only |

Schema is versioned in `supabase-schema.sql`; RLS and migration fixes live in the
`*.sql` scripts at the repo root (see `SUPABASE_MIGRATION_STATUS.md`).

## Getting started

```bash
npm i
npm run dev    # http://localhost:3000
```

Environment variables (`.env.local`, gitignored):

```
VITE_SUPABASE_URL=https://<project>.supabase.co
VITE_SUPABASE_ANON_KEY=<anon key>
```

Then, once:

1. Run `supabase-schema.sql` in the Supabase SQL editor (creates tables + RLS).
2. Run `setup-profile-photos-storage.sql` (creates the storage bucket + policies).
3. Create your first admin: add a user in Supabase Auth, then insert their row into
   `admin_users` (see `SUPABASE_MIGRATION_STATUS.md` for the exact SQL).

## Deployment

SPA rewrite rules are preconfigured for Render (`render.yaml`), Vercel (`vercel.json`),
and Netlify (`netlify.toml`) so deep links don't 404 on refresh.

## Contributing

1. Route all data access through `src/lib/founderService.ts` / `adminService.ts` — never query `supabase` directly from pages
2. Keep auth logic in `authManager.ts` and role checks in `ProtectedRoute`
3. Prefer Supabase RLS over client-side filtering for access control
4. Run `npm run build` and fix type errors before pushing

## Status

Supabase migration is ~90% complete — auth, data services, RLS, and storage are
implemented and building cleanly; remaining work is production testing and the
first admin bootstrap (tracked in `SUPABASE_MIGRATION_STATUS.md`).

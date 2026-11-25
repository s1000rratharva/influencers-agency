# Creator–Business Collaboration Marketplace

A two-sided marketplace connecting social media creators with businesses for
paid collaborations. This repository currently implements the **creator-facing
platform**; the business side exists as folder structure only and will be
built in a later phase.

## Tech stack

| Layer | Technology |
| --- | --- |
| Framework | Next.js 16 (App Router, Turbopack) |
| Language | TypeScript (strict) |
| Styling | Tailwind CSS v4 |
| UI | shadcn/ui (Radix primitives) + Lucide icons |
| Backend | Next.js Server Components / Server Actions / Route Handlers |
| Database | Supabase PostgreSQL (with Row Level Security) |
| Auth | Supabase Auth (`@supabase/ssr` cookie sessions) |
| Storage | Supabase Storage |
| Forms | React Hook Form + Zod |
| Deployment | Vercel |

## Getting started

### 1. Install dependencies

```bash
npm install
```

### 2. Configure environment variables

Copy `.env.example` to `.env.local` and fill in the values:

```bash
cp .env.example .env.local
```

| Variable | Purpose | Required for |
| --- | --- | --- |
| `NEXT_PUBLIC_SUPABASE_URL` | Supabase project URL (public) | everything |
| `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY` | Supabase publishable/anon key (public) | everything |
| `SUPABASE_SECRET_KEY` | Supabase secret key (**server-only**) | privileged server operations |
| `NEXT_PUBLIC_SITE_URL` | App base URL for auth redirects/emails | auth flows |
| `DATABASE_URL` | Direct DB connection (migrations tooling only) | running migrations |

> Never prefix `SUPABASE_SECRET_KEY` or `DATABASE_URL` with `NEXT_PUBLIC_`.

### 3. Configure Supabase Auth

In the Supabase Dashboard → **Authentication → URL Configuration**, set:

- Site URL: `http://localhost:3000` (or your production URL — must match
  `NEXT_PUBLIC_SITE_URL`)
- Additional redirect URLs (add the production equivalents when deploying):

  ```text
  http://localhost:3000/auth/confirm
  http://localhost:3000/auth/confirm?next=/reset-password
  ```

GoTrue rejects redirect URLs that are not allow-listed and silently falls back
to the Site URL, which breaks email confirmation and password-reset links.
The local allow-list is kept in `supabase/config.jsonc`.

### 4. Set up the database

Migrations live in `supabase/migrations/` and are applied in filename order.

**Option A — Supabase CLI (recommended):**

```bash
supabase login                 # once, with your Supabase access token
supabase link --project-ref jpebrwagpsvutixdyrhi
supabase db push               # apply migrations to the hosted project
```

**Local development stack** (requires Docker):

```bash
supabase start                 # local Postgres + Auth + Storage + Studio
supabase db reset              # re-apply all migrations + seed data
```

**Option B — SQL Editor:** open each file in `supabase/migrations/` in order
in the Supabase Dashboard SQL Editor and run them top to bottom.

### 5. Run the dev server

```bash
npm run dev
```

Open http://localhost:3000.

#### Seed data

`supabase/seed.sql` creates demo accounts (password `password123`):

| Email | Role | Notes |
| --- | --- | --- |
| `mia@example.com` | creator | Fully onboarded, active collaboration, unread messages |
| `rahul@example.com` | creator | Partial onboarding (step 4), pending invitation |
| `acme@example.com` | business | Owns the demo campaigns (UI not built yet) |

## Commands

| Command | Description |
| --- | --- |
| `npm run dev` | Start development server (Turbopack) |
| `npm run build` | Production build |
| `npm run start` | Serve the production build |
| `npm run lint` | Run ESLint |
| `npx tsc --noEmit` | Type-check without emitting files |

## Implemented in Phase 3

- **Auth:** role-aware signup (`creator` / `business`), login, logout,
  email confirmation (PKCE and OTP links), resend confirmation, forgot-password
  and reset-password flows. All inputs are re-validated with Zod in Server
  Actions; `next` redirect targets are restricted to same-origin paths.
- **Route gating:** `src/proxy.ts` refreshes Supabase cookies and redirects
  logged-out visitors away from `/creator/*`; signed-in users are redirected
  away from `/login` and `/signup`.
- **Creator onboarding:** 12 validated steps with save/resume, review/edit,
  social-account add/remove, save & exit, and a server-side completion check
  before the dashboard unlocks. Partial progress is stored in
  `creator_profiles.onboarding_step` (completed-step count, never decreases).
- **Dashboard:** read-only creator dashboard with real profile, focus areas and
  social-account data. Campaign discovery, applications and messaging are
  intentionally deferred to later phases.

## Architecture

```text
src/
├── app/            # Routes (App Router). Route groups: (marketing), (auth);
│                   # feature areas: creator/ (implemented), business/ (placeholder)
├── components/     # UI only — no data fetching or business logic
│   ├── ui/         # shadcn/ui primitives
│   ├── shared/     # components reused across feature areas
│   ├── creator/    # creator feature components
│   └── business/   # placeholder for the future business side
├── lib/
│   ├── supabase/   # browser client, server client, proxy session helper
│   ├── auth/       # role helpers, guards
│   ├── validations/# Zod schemas (server-side validation)
│   └── utils/      # general utilities
├── services/       # database/business logic (repositories), used by pages/actions
├── types/          # shared TypeScript types incl. generated DB types
└── proxy.ts        # Next.js 16 proxy — refreshes auth session cookies
```

### Key decisions

- **`proxy.ts` instead of `middleware.ts`** — Next.js 16 deprecated the
  middleware file convention; `proxy.ts` runs the same role (session refresh)
  on the Node.js runtime.
- **Server components by default** — client components are used only where
  interactivity is required (forms, dialogs, live updates).
- **Services own all DB access** — pages and actions call service modules,
  never issue ad-hoc queries, so authorization logic stays in one layer.
- **RLS is the security boundary** — every user-facing query runs with the
  end user's session; the secret key never leaves the server.
- **Role-based access** — `profiles.role` (creator | business | admin) drives
  route guards; creator and business areas are isolated from each other.

## Development phases

- [x] **Phase 1** — Project setup, dependencies, folder structure, Supabase configuration
- [x] **Phase 2** — Database schema, migrations, RLS policies
- [x] **Phase 3** — Authentication and creator onboarding
- [ ] **Phase 4** — Creator dashboard and profile
- [ ] **Phase 5** — Social accounts and portfolio
- [ ] **Phase 6** — Campaign discovery and applications
- [ ] **Phase 7** — Collaborations and messaging
- [ ] **Phase 8** — Notifications and verification
- [ ] **Phase 9** — Testing, security review, deployment preparation

## Deployment (Vercel)

1. Push the repository to GitHub and import it in Vercel.
2. Add all environment variables from `.env.example` in Vercel project settings.
3. Update Supabase Auth URL configuration with the production domain.

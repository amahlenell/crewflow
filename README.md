# CrewFlow

Job and crew management for trade businesses (roofing, electrical, plumbing, landscaping, construction). Replaces WhatsApp/spreadsheets/paper with one system: who's working where, what needs doing, what's done.

## Stack
Next.js (App Router) + TypeScript + Tailwind + shadcn/ui · Supabase (Postgres, Auth, Storage) · Vercel

## Setup

1. **Create a Supabase project** at supabase.com.
2. **Run the migrations**, in order, against your project (via `supabase db push` with the Supabase CLI, or paste each file into the SQL Editor in order):
   - `supabase/migrations/0001_init_schema.sql`
   - `supabase/migrations/0002_rls_policies.sql`
   - `supabase/migrations/0003_signup_functions.sql`
   - `supabase/migrations/0004_storage.sql`
3. **Copy `.env.example` to `.env.local`** and fill in your Supabase URL + anon key (Project Settings → API).
4. `npm install`
5. `npm run db:types` (optional, needs Supabase CLI) to generate real types into `src/lib/types/database.ts`, replacing the placeholder.
6. `npm run dev`

## What's built (Sprint 1: Foundation)

- Next.js project structure, Tailwind config wired to the CrewFlow design tokens (colors, radius, type) pulled directly from the Figma export
- Full database schema: `companies`, `users`, `customers`, `jobs`, `job_assignments`, `job_checklists`, `photos`, `comments`, `activity_logs`, `notifications`, `invitations` — with enums, indexes, and `updated_at` triggers
- Row Level Security policies on every table enforcing tenant isolation (company A can never read/write company B's rows) and role permissions (owner/manager vs worker)
- Tenant-scoped Storage bucket + policies for job photos
- Signup flow as two atomic RPCs: `create_company_and_owner` and `accept_invitation`
- Auth middleware (session refresh + route protection)
- Server Actions for login/signup/logout/password reset/company creation
- Dashboard route group with role-based redirect (workers → `/my-jobs`, not the admin shell)
- First two real pages wired end-to-end: Login and Dashboard (dashboard queries are live Supabase counts/activity, not mock data)

## What's built (Sprint 2: Authentication) — complete

- Real Manrope (headings) / Inter (body) fonts via `next/font/google`, plus the full type scale (H1 48px → Caption 12px) and weight usage, taken directly from the design system reference
- Logo asset saved to `public/logo.png`
- Signup, Login, Forgot Password, Reset Password pages — all wired to the Server Actions from Sprint 1, no placeholder forms
- `AuthForm` client wrapper (`useActionState`) so validation errors from Supabase (bad password, existing account, etc.) actually render in the UI instead of disappearing
- Company creation / onboarding page, calling the `create_company_and_owner` RPC
- Invite acceptance page at `/invite/[token]`: prompts login/signup if logged out (preserving the token through the redirect), then calls `accept_invitation()` once authenticated
- Middleware updated so invite links work correctly whether the visitor is logged in or not

## What's built (Sprint 4: Worker Experience) — complete

- `(worker)` route group: mobile-first layout (max-width shell, dark top bar, no sidebar), guarded so only `role = worker` lands here — owners/managers get bounced to `/dashboard` and vice versa, enforced server-side
- `/my-jobs`: worker's assigned jobs only, via `job_assignments` join (RLS backs this up regardless)
- `/my-jobs/[jobId]`: job detail scoped to that worker — checklist (can toggle, not add/remove, matching the RLS split between worker and owner/manager), status control, comments
- Real photo upload: `uploadPhoto` server action streams the file to the `job-photos` Storage bucket at `{company_id}/{job_id}/...`, inserts the `photos` row, and logs activity — not a fake "upload" button
- Photos are stored in a **private** bucket, so display uses short-lived signed URLs (`getPhotoUrl`, 1hr expiry) rather than public URLs — `PhotoGrid` is shared between the worker view and the owner/manager job detail page, so both now show real thumbnails instead of the earlier placeholder colored boxes

## What's built (Sprint 5: Polish) — complete, and Sprint 1–5 now cover the full MVP spec

- **Notifications**: real `notifications` rows created on worker assignment and on new comments (to everyone assigned except the commenter). Bell icon in the Topbar is now a live dropdown — unread dot, list, mark-one-read, mark-all-read — backed by an async server component that queries the `notifications` table per request, not a static placeholder icon.
- **Search**: `/jobs` and `/customers` now support real text search (`ilike` against title/address or name/phone/email) via the `q` query param, combinable with the existing filters.
- **Filters**: `/jobs` gained a priority filter alongside status, with all three filters (status, priority, q) composable and preserved across links.
- **Error handling**: `error.tsx` boundaries for both the `(dashboard)` and `(worker)` route groups (catch render/query failures with a "Try again" reset, not a blank white screen), a proper `not-found.tsx`, and `loading.tsx` skeletons for Dashboard, Jobs list, and My Jobs — so navigating between pages shows a real skeleton instead of a flash of nothing.
- **Toast feedback**: wired `sonner` (already a dependency, unused until now) into status changes, worker assignment/removal, checklist toggling, and photo upload — actions that don't redirect were previously failing silently on error; now every one of them surfaces success or failure.

## Where CrewFlow stands after Sprint 5

Every MVP feature from the original brief is implemented in code: auth (signup/login/logout/reset), company creation, invitations, jobs (CRUD, status, priority, assignment), checklists, photo upload to private per-tenant Storage, comments, activity timeline, staff/customer management, dashboard metrics, notifications, search, and filters — all behind RLS.

**This has still not been run against a live Supabase project or `npm install`'d** — this sandbox has no network access. That verification pass is now the actual next step, not another sprint of new code:

1. Create the Supabase project, run all 4 migrations in order
2. `npm install`, `npm run dev`
3. Full walkthrough: signup → create company → invite a worker (copy the link, since email-send is still a stub) → accept invite as the worker → owner creates a job, assigns the worker, adds a checklist item → worker sees it on `/my-jobs`, uploads a photo, toggles the checklist, comments → owner sees the notification and the real photo
4. Specifically test the RLS boundary with two separate companies, not just the UI redirects

## Known gaps (unchanged, still worth doing before charging customers)

- Invite emails aren't sent automatically — copy-link workaround only
- No automated tests
- No rate limiting / abuse protection on the invite or photo-upload endpoints
- `Database` type is a placeholder (`any`) until `npm run db:types` is run against the real project

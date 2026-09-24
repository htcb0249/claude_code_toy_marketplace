# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

A toy marketplace web app (React + TypeScript + Vite + shadcn/ui, Tailwind CSS) backed by Supabase (Postgres + Auth + Storage + Realtime). Originally scaffolded with Lovable. Users can list toys for sale, browse/search public listings, save items, and message sellers via a conversation/chat system with read receipts and online presence.

## Commands

```
npm run dev        # start Vite dev server on port 8080
npm run build       # production build
npm run build:dev   # development-mode build
npm run lint         # eslint
npm run preview      # preview a production build
```

There is no test suite configured in this project.
TypeScript is non-strict (`strict`, `strictNullChecks` and `noImplicitAny` are all off) and ESLint turns off `no-unused-vars`, so `npm run lint` catches little. To type-check, run `npx tsc -p tsconfig.app.json --noEmit`.

### Supabase (local development)

```
supabase login
supabase start                 # starts local stack -> http://127.0.0.1:54323 (Studio)
supabase db reset              # re-applies all migrations + seed.sql
```
`seed.sql` is intentionally empty (`auth.users` can't be pre-seeded), so local data comes from signing up through the app.

The frontend (`src/integrations/supabase/client.ts`) auto-switches between the local Supabase instance (`127.0.0.1:54321`) and the hosted production project based on `window.location.hostname`, so no `.env` toggling is needed to develop locally. Only exact `localhost` / `127.0.0.1` count as local; opening the dev server via a LAN IP (Vite binds to all interfaces) or any other hostname talks to **production**. Keys are hardcoded there (there are no env vars). Despite its "auto-generated, do not edit" header, this file contains hand-written switching logic.

Adding a migration:
```
supabase migration new xxxxx_table   # edit the generated SQL in supabase/migrations/
supabase db reset                    # re-apply locally
```
After changing tables or RPC signatures, regenerate `src/integrations/supabase/types.ts` (`supabase gen types typescript --local > src/integrations/supabase/types.ts`).

Deploying to the remote project:
```
supabase login
supabase link --project-ref $SUPABASE_PROJECT_REF
supabase db push
```

MCP servers (Playwright, context7, Sentry, Supabase) can be configured by copying `.mcp.json.example` to `.mcp.json` and filling in the placeholder tokens/refs.

## Architecture

### Data layer: Supabase RPC functions, not direct table queries

Almost all reads/writes from the frontend go through Postgres RPC functions defined in `supabase/migrations/` (mostly `00000000_consolidated_migration.sql`, but later migrations add or redefine some, e.g. `get_user_saved_products`, `toggle_saved_product`, `is_product_saved`; check all migration files for the current definition), rather than querying tables directly via the JS client. This is deliberate: RLS policies on joined tables (e.g. products + profiles + product_images) will silently return no rows for a direct multi-table `select`, so joins/aggregations are done server-side in a `SECURITY DEFINER` function instead. Key functions include `get_public_products`, `get_public_product_detail`, `get_profile_names`, `get_user_conversations`, `get_conversation_details`, `get_conversation_messages_with_read_status`, `get_user_saved_products`, `create_conversation`, `mark_message_read`/`mark_conversation_read`.

When adding a feature that needs data across related tables: check the migration file for an existing RPC first, and if none fits, add a new `CREATE FUNCTION` (in a new migration) rather than composing joins client-side.

### Schema shape (see the consolidated migration for full detail)

- `profiles` — one row per auth user (created via `handle_new_user` trigger on signup)
- `products` / `product_images` — listings and their photos (Storage bucket policies scoped to `auth.uid()`'s folder)
- `conversations` / `participants` / `messages` / `message_status` — per-product buyer/seller chat, with triggers (`bump_conversation_on_message`, `add_participants_on_conversation`) and read-receipt RPCs
- `saved_products` — user bookmarks

### Frontend structure

- `src/pages/*` — route-level screens, wired up in `src/App.tsx` (React Router). New routes must be added above the catch-all `*` route. `/` renders `Categories`; `Index.tsx` is unused.
- Storage bucket `product-images` is public. Upload paths must start with `<auth.uid()>/` or the storage policies reject the upload.
- `src/hooks/*` — data-fetching hooks that wrap Supabase RPC calls (e.g. `usePublicProducts`, `useUserProducts`, `useSavedProducts`, `useUnreadMessagesCount`) and `useAuth` (wraps `supabase.auth` session/user state).
- `src/contexts/PresenceProvider.tsx` — app-wide online/offline presence via a single Supabase Realtime channel (`global-presence`), tracked per authenticated user.
- `src/components/ui/*` — shadcn/ui primitives (generated; treat as vendor code, edit sparingly).
- `src/components/auth/*` — sign in/up/reset forms.
- `@/*` path alias resolves to `src/*` (configured in `vite.config.ts` and `tsconfig.json`).
- `src/lib/imageUtils.ts` — client-side image resizing (Canvas API) before upload, used by listing creation.

### Conventions specific to this repo

- Name component files in PascalCase matching the export (e.g. `CreateListingForm.tsx`).
- Never commit PII (e.g. user emails) into code, seed data, or migrations.
- Prefer reusing an existing RPC function over writing a new Supabase query; add a new RPC (via migration) when joined/RLS-protected data is needed and no existing function covers it.

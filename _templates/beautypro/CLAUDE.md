# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

**BeautyPro** — beauty studio management app. Turborepo monorepo targeting iOS + Android (Expo) and Web (Next.js).

## Commands

```bash
# Install (root)
npm install

# Run mobile app
npx expo start

# Run web app
cd apps/web && npm run dev

# Type check all packages
npx turbo run typecheck

# Lint all packages
npx turbo run lint

# Build all packages
npx turbo run build

# Run single package task
npx turbo run typecheck --filter=@beautypro/shared

# Supabase: generate TypeScript types after schema changes
npx supabase gen types typescript --project-id <SUPABASE_PROJECT_ID> > packages/supabase/src/types.ts
```

## Monorepo Structure

```
apps/
  mobile/   → Expo (React Native) app — see apps/mobile/CLAUDE.md
  web/      → Next.js + React.js app (Phase 8)
packages/
  shared/   → Business logic: hooks, Zustand store, utils — see packages/shared/CLAUDE.md
  ui/       → Shared UI components (cross-platform where possible)
  supabase/ → Supabase client + generated types — see packages/supabase/CLAUDE.md
  i18n/     → i18next setup + locale files — see packages/i18n/CLAUDE.md
```

**~65-70% of code lives in `packages/`** — shared between mobile and web. Only navigation and platform-specific UI lives in `apps/`.

## Key Conventions

- All user-facing strings go through `t('namespace.key')` — no hardcoded Serbian or English strings in components
- Currency: stored in `profiles.currency`, always formatted via `Intl.NumberFormat(locale, { currency })`
- Profession config drives feature flags (`showCycleTracker`, `showLashProfile`, `services`) — check `PROFESSION_CONFIG` before adding profession-specific UI
- Supabase RLS: every query is scoped to `profile_id = auth.uid()` — never bypass RLS in queries
- `packages/supabase/src/types.ts` is auto-generated — never edit by hand

## Git Flow

- `main` — production
- `develop` — active development
- Feature branches from `develop`, PR back to `develop`

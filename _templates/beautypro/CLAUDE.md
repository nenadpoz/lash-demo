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

## Coding Standards

### Stilizovanje

**Mobile** (`apps/mobile`): `styled-components/native`
```typescript
import styled from 'styled-components/native'
import type { DefaultTheme } from 'styled-components/native'

const Container = styled.View`
  background-color: ${({ theme }) => theme.colors.background};
  padding: ${({ theme }) => theme.spacing.md}px;
`
```

**Web** (`apps/web`): `styled-components`
```typescript
import styled from 'styled-components'

const Container = styled.div`
  background-color: ${({ theme }) => theme.colors.background};
`
```

**Tema:** design tokeni žive u `packages/ui/src/theme.ts`, importuju se u oba app-a. Nikad hardcodovane boje ili spacing vrednosti direktno u komponentama.

---

### TanStack Query

Koristimo `useSuspenseQuery` + centralizovani `queryKeys` objekat.

```typescript
// packages/shared/src/queryKeys.ts
export const queryKeys = {
  clients: {
    all: () => ['clients'] as const,
    detail: (id: string) => ['clients', id] as const,
  },
  appointments: {
    all: () => ['appointments'] as const,
    byDay: (date: string) => ['appointments', 'day', date] as const,
  },
}

// packages/shared/src/hooks/useClients.ts
export function useClients() {
  return useSuspenseQuery({
    queryKey: queryKeys.clients.all(),
    queryFn: () => clientQueries.getAll(),
  })
}
```

Svaki ekran koji koristi `useSuspenseQuery` mora biti omotan `<Suspense fallback={<LoadingScreen />}>` u parent layoutu.

Mutacije uvek invalidiraju relevantne queryKeys:
```typescript
onSuccess: () => queryClient.invalidateQueries({ queryKey: queryKeys.clients.all() })
```

---

### Zustand Store

Jedan store po feature-u. `persist` samo za `authStore` i `settingsStore`.

```typescript
// packages/shared/src/store/clientStore.ts
interface ClientStore {
  searchQuery: string
  filterBlocked: boolean
  setSearchQuery: (q: string) => void
  setFilterBlocked: (v: boolean) => void
}

export const useClientStore = create<ClientStore>((set) => ({
  searchQuery: '',
  filterBlocked: false,
  setSearchQuery: (q) => set({ searchQuery: q }),
  setFilterBlocked: (v) => set({ filterBlocked: v }),
}))
```

**Pravilo:** Zustand čuva UI state (search, filter, active tab, modal state). Server data (clients, appointments) živi isključivo u TanStack Query cache-u — nikad se ne kopira u Zustand.

---

### Error Handling

`throw error` u queryFn — TanStack hvata i eksponira kroz `error` property:

```typescript
// packages/supabase/src/queries/clientQueries.ts
getAll: async () => {
  const { data, error } = await supabase.from('clients').select('*')
  if (error) throw error   // ← uvek throw, nikad return { data, error }
  return data
}
```

**Nivoi:**
1. **Globalni error boundary** (`apps/mobile/app/_layout.tsx`, `apps/web/app/layout.tsx`) — hvata neočekivane greške, prikazuje fallback ekran
2. **Per-screen** za kritične flow-ove: auth, payment, onboarding — eksplicitan `try/catch` i korisnička poruka
3. **Inline** za nekritične operacije (npr. promjena teme): `mutation.isError && <ErrorMessage />`

---

### TypeScript

- `strict: true` u svim `tsconfig.json` fajlovima
- Props interface uvek eksplicitan (ne `React.FC` bez tipova):
  ```typescript
  interface ButtonProps {
    label: string
    variant: 'primary' | 'outline' | 'ghost'
    onPress: () => void
    disabled?: boolean
  }
  ```
- Nema `any` — koristiti `unknown` sa type guard-om ako tip nije poznat

---

## Git Flow

- `main` — production
- `develop` — aktivni razvoj
- Feature grane od `develop`, PR nazad u `develop`

# CLAUDE.md — packages/shared

Shared business logic used by both `apps/mobile` and `apps/web`. No UI, no platform APIs.

## Structure

```
src/
  hooks/        → TanStack Query hooks (useClients, useAppointments, useFinance...)
  store/        → Zustand stores (authStore, professionStore, settingsStore)
  utils/        → Pure functions (formatPrice, formatDate, calcCycleStep, validateOverlap)
  constants/    → PROFESSION_CONFIG, SERVICE_KEYS, CYCLE_STEPS, SVC_LABELS
  types/        → Shared TypeScript types (not Supabase — those live in packages/supabase)
```

## Hook Pattern

All data hooks wrap Supabase queries via TanStack Query:

```typescript
export function useClients() {
  return useQuery({
    queryKey: ['clients'],
    queryFn: () => clientQueries.getAll(),  // from packages/supabase/queries/
  })
}
```

Mutation hooks return `useMutation` with `onSuccess: () => queryClient.invalidateQueries(...)`.

## Zustand Store Pattern

```typescript
// store/professionStore.ts
interface ProfessionStore {
  activeProfession: ProfessionKey
  setActiveProfession: (key: ProfessionKey) => void
}
export const useProfessionStore = create<ProfessionStore>()(
  persist((set) => ({ ... }), { name: 'profession-store' })
)
```

## PROFESSION_CONFIG

```typescript
// constants/professions.ts
export const PROFESSION_CONFIG: Record<ProfessionKey, ProfessionConfig> = {
  lash: { showCycleTracker: true, showLashProfile: true, services: [...], ... },
  nails: { showCycleTracker: false, showLashProfile: false, services: [...], ... },
  // ... 9 professions total
}
```

Always gate profession-specific features through `PROFESSION_CONFIG[key].showCycleTracker` etc. — never hardcode profession keys in components.

## Cycle Logic

`calcCycleStep(lastAppointmentDate): CycleStep` — returns `'new-set' | 'c1' | 'c2' | 'c3' | 'done-all'` based on days elapsed. Lives in `utils/cycle.ts`.

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

Koristimo `useSuspenseQuery` + `queryKeys` iz `src/queryKeys.ts`:

```typescript
// src/hooks/useClients.ts
export function useClients() {
  return useSuspenseQuery({
    queryKey: queryKeys.clients.all(),
    queryFn: () => clientQueries.getAll(),
  })
}
```

Mutacije uvek invalidiraju queryKey:
```typescript
export function useCreateClient() {
  const queryClient = useQueryClient()
  return useMutation({
    mutationFn: clientQueries.create,
    onSuccess: () => queryClient.invalidateQueries({ queryKey: queryKeys.clients.all() }),
  })
}
```

Svi `queryKey`-evi centralizovani u `src/queryKeys.ts` — nikad inline stringovi.

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

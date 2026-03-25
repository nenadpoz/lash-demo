# CLAUDE.md — apps/mobile

Expo (Managed Workflow) React Native app. Primary platform for BeautyPro.

## Navigation (Expo Router)

File-based routing — screen = file in `app/` directory:

```
app/
  (auth)/
    login.tsx         → /login
    signup.tsx        → /signup
    profession.tsx    → /profession (onboarding, multi-select)
  (app)/
    dashboard.tsx     → main calendar view
    clients/
      index.tsx       → client list
      [id].tsx        → client profile
    finance/
      index.tsx       → finance tabs
    settings.tsx
  (shop)/             → Phase 9 only
```

Layouts: `_layout.tsx` in each group defines the navigator type (Stack, Tabs, etc.).

## Screen Patterns

- Auth guard: in `app/(app)/_layout.tsx` — redirect to `/(auth)/login` if no session
- Tab navigation: `app/(app)/_layout.tsx` defines bottom tabs (4 tabs: Dashboard, Clients, Finance, Settings)
- Modals: use `router.push()` with a modal screen, or bottom sheet via `@gorhom/bottom-sheet`

## State

- **Zustand** (`packages/shared/store/`) — auth state, active profession, app settings
- **TanStack Query** — all server data (clients, appointments, finance). Never fetch Supabase directly in components — use hooks from `packages/shared/hooks/`
- Optimistic updates for appointment status changes (complete/cancel)

## Platform Notes

- All dimensions in `dp` (density-independent pixels), not `px`
- Safe area: always wrap screens with `<SafeAreaView>` or use `useSafeAreaInsets()`
- Keyboard: use `KeyboardAvoidingView` on forms, behavior `'padding'` on iOS, `'height'` on Android

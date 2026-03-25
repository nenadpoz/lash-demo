# CLAUDE.md — packages/supabase

Supabase client, auto-generated types, and reusable query functions.

## Critical Rule

**`src/types.ts` is auto-generated. Never edit it manually.**

After any schema migration, regenerate:
```bash
npx supabase gen types typescript --project-id <SUPABASE_PROJECT_ID> > packages/supabase/src/types.ts
```

## Structure

```
src/
  client.ts       → createClient() — singleton, handles both browser and server (SSR)
  types.ts        → AUTO-GENERATED from Supabase schema — do not edit
  queries/
    clientQueries.ts
    appointmentQueries.ts
    financeQueries.ts
    profileQueries.ts
```

## Query Pattern

All queries are scoped to the authenticated user via RLS — no `profile_id` filter needed in queries because Supabase RLS policy handles it automatically:

```typescript
// queries/clientQueries.ts
export const clientQueries = {
  getAll: async () => {
    const { data, error } = await supabase
      .from('clients')
      .select('*')
      .order('full_name')
    if (error) throw error
    return data
  },
}
```

## RLS Policy Rule

Every table has: `USING (profile_id = auth.uid())`. Never add `profile_id` filters in queries — it's redundant and creates a false sense of security. The RLS policy is the source of truth.

## Type Usage

```typescript
import type { Tables, TablesInsert, TablesUpdate } from '@beautypro/supabase'

type Client = Tables<'clients'>
type NewClient = TablesInsert<'clients'>
type ClientUpdate = TablesUpdate<'clients'>
```

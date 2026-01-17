description: Data provider pattern for server-side data fetching and context initialization
globs:
  - "**/data-providers/**"
alwaysApply: false
---

# Data Provider Pattern

Data providers are Server Components that fetch initial data and wrap client components with context providers.

## Purpose

Data providers enable:
1. Server-side data fetching (SSR/RSC)
2. Initial data for React Query
3. Context provider initialization
4. Type-safe data passing

## File Structure

```
/data-providers/
├── entity-details.tsx      # For detail/edit pages
├── entity-list.tsx          # For list pages
└── fields-data-provider.tsx # Simple pass-through
```

## Basic Pattern

Reference: /features/entity-details/data-providers/event-details.tsx

```typescript
'use server';

import { resolveOrThrow } from '@/lib/utils/server-action-helper';
import { ConnectedEntityDetails } from '../connected/connected-entity-details';
import { EntityContextProvider } from '../contexts/entity-context';
import { fetchEntityByIdAction } from '../server-actions/entity';
import { notFound } from 'next/navigation';

export async function EntityDetailsDataProvider({ entityId }: { entityId?: string }) {
  // 1. Fetch initial data (only if ID provided)
  const initialData = entityId
    ? await resolveOrThrow(fetchEntityByIdAction({ id: entityId }))
    : null;

  // 2. Handle not found case
  if (entityId && !initialData?.entity) {
    notFound();
  }

  // 3. Wrap with context provider and render connected component
  return (
    <EntityContextProvider initialData={initialData}>
      <ConnectedEntityDetails isNewEntity={!entityId} />
    </EntityContextProvider>
  );
}

// 4. Export type for initial data
export type InitialEntityData = Awaited<ReturnType<typeof fetchEntityByIdAction>>['data'];
```

## Naming Convention

- File: `[entity]-details.tsx` or `[entity]-data-provider.tsx`
- Export: `[Entity]DetailsDataProvider` or `[Entity]DataProvider`

## Pattern Variations

### Detail/Edit Page Data Provider

For pages with entity ID (create or edit):

```typescript
'use server';

export async function EntityDetailsDataProvider({ 
  entityId 
}: { 
  entityId?: string 
}) {
  // Fetch only if editing (ID provided)
  const initialData = entityId
    ? await resolveOrThrow(fetchEntityByIdAction({ id: entityId }))
    : null;

  // 404 if ID provided but entity not found
  if (entityId && !initialData?.entity) {
    notFound();
  }

  return (
    <EntityContextProvider initialData={initialData}>
      <ConnectedEntityDetails isNewEntity={!entityId} />
    </EntityContextProvider>
  );
}

export type InitialEntityData = Awaited<
  ReturnType<typeof fetchEntityByIdAction>
>['data'];
```

### List Page Data Provider

For list pages (usually simpler):

Reference: /features/entity-list/data-providers/fields-data-provider.tsx

```typescript
'use server';

import { ConnectedFeature } from '../connected/connected-feature';

export async function FeatureDataProvider() {
  // No initial data needed - client will fetch based on query params
  return <ConnectedFeature />;
}
```

**Why no data fetch?**
- List data depends on query params (filters, sorting, pagination)
- Query params are read client-side from URL
- Better to let client fetch with React Query

### Data Provider with Multiple Queries

For pages that need multiple data sources:

```typescript
'use server';

export async function EntityDetailsDataProvider({ entityId }: { entityId: string }) {
  // Fetch multiple data sources in parallel
  const [initialEntityData, optionsData] = await Promise.all([
    resolveOrThrow(fetchEntityByIdAction({ id: entityId })),
    resolveOrThrow(fetchOptionsAction()),
  ]);

  if (!initialEntityData?.entity) {
    notFound();
  }

  return (
    <EntityContextProvider 
      initialData={initialEntityData}
      options={optionsData.options}
    >
      <ConnectedEntityDetails />
    </EntityContextProvider>
  );
}

export type InitialEntityData = Awaited<
  ReturnType<typeof fetchEntityByIdAction>
>['data'];
```

## Using Data Providers in Pages

### In App Router Pages

```typescript
// app/entities/[id]/page.tsx
import { EntityDetailsDataProvider } from '@/features/entity-details';

export default async function EntityPage({
  params,
}: {
  params: Promise<{ id: string }>;
}) {
  const { id } = await params;
  return <EntityDetailsDataProvider entityId={id} />;
}
```

### For Create Pages

```typescript
// app/entities/new/page.tsx
import { EntityDetailsDataProvider } from '@/features/entity-details';

export default async function NewEntityPage() {
  // No ID = create mode
  return <EntityDetailsDataProvider />;
}
```

## Type Export Pattern

Always export the initial data type for use in contexts:

```typescript
export type InitialEntityData = Awaited<
  ReturnType<typeof fetchEntityByIdAction>
>['data'];
```

This type is used in the context provider:

```typescript
// contexts/entity-context.tsx
import { InitialEntityData } from '../data-providers/entity-details';

export const EntityContextProvider = ({
  children,
  initialData,
}: {
  children: React.ReactNode;
  initialData: InitialEntityData | null;
}) => {
  // Use initialData
};
```

## Error Handling

### Not Found (404)

```typescript
import { notFound } from 'next/navigation';

export async function EntityDetailsDataProvider({ entityId }: { entityId: string }) {
  const initialData = await resolveOrThrow(fetchEntityByIdAction({ id: entityId }));

  if (!initialData?.entity) {
    notFound(); // Returns Next.js 404 page
  }

  return (
    <EntityContextProvider initialData={initialData}>
      <ConnectedEntityDetails />
    </EntityContextProvider>
  );
}
```

### Unauthorized (403)

```typescript
import { redirect } from 'next/navigation';

export async function EntityDetailsDataProvider({ entityId }: { entityId: string }) {
  try {
    const initialData = await resolveOrThrow(fetchEntityByIdAction({ id: entityId }));
    
    return (
      <EntityContextProvider initialData={initialData}>
        <ConnectedEntityDetails />
      </EntityContextProvider>
    );
  } catch (error) {
    if (error.message === 'Unauthorized') {
      redirect('/login');
    }
    throw error; // Let Next.js error boundary handle
  }
}
```

## Data Flow

```
1. Page Component (Server)
   ↓
2. Data Provider (Server) - Fetches initial data
   ↓
3. Context Provider (Client) - Receives initialData
   ↓
4. React Query - Uses initialData, can refetch
   ↓
5. Connected Component (Client) - Consumes context
   ↓
6. Presentational Components - Receive props
```

## Performance Considerations

### Parallel Data Fetching

```typescript
// ✅ GOOD: Fetch in parallel
const [entityData, optionsData] = await Promise.all([
  resolveOrThrow(fetchEntityAction()),
  resolveOrThrow(fetchOptionsAction()),
]);

// ❌ BAD: Sequential fetching
const entityData = await resolveOrThrow(fetchEntityAction());
const optionsData = await resolveOrThrow(fetchOptionsAction()); // Waits for first!
```

### Conditional Fetching

```typescript
// Only fetch if needed
const initialData = entityId
  ? await resolveOrThrow(fetchEntityByIdAction({ id: entityId }))
  : null;
```

## Anti-Patterns

### ❌ Don't: Fetch List Data in Data Provider
```typescript
// ❌ BAD: List data depends on client-side query params
export async function EntitiesDataProvider() {
  const entities = await resolveOrThrow(fetchEntitiesAction({}));
  return <ConnectedEntities initialData={entities} />;
}

// ✅ GOOD: Let client fetch based on query params
export async function EntitiesDataProvider() {
  return <ConnectedEntities />;
}
```

### ❌ Don't: Use 'use client' Directive
```typescript
// ❌ BAD: Data provider should be server component
'use client';

export async function EntityDetailsDataProvider() {
  // This won't work as expected!
}

// ✅ GOOD: Use 'use server' or no directive (defaults to server)
'use server';

export async function EntityDetailsDataProvider() {
  // Runs on server
}
```

### ❌ Don't: Forget to Export Type
```typescript
// ❌ BAD: Context can't type initialData properly
export async function EntityDetailsDataProvider({ entityId }) {
  const initialData = await resolveOrThrow(fetchEntityByIdAction({ id: entityId }));
  return <EntityContextProvider initialData={initialData}>...</EntityContextProvider>;
}

// ✅ GOOD: Export type for context to use
export async function EntityDetailsDataProvider({ entityId }) {
  const initialData = await resolveOrThrow(fetchEntityByIdAction({ id: entityId }));
  return <EntityContextProvider initialData={initialData}>...</EntityContextProvider>;
}

export type InitialEntityData = Awaited<ReturnType<typeof fetchEntityByIdAction>>['data'];
```

### ❌ Don't: Put Business Logic in Data Provider
```typescript
// ❌ BAD: Business logic belongs in context or hooks
export async function EntityDetailsDataProvider({ entityId }) {
  const initialData = await resolveOrThrow(fetchEntityByIdAction({ id: entityId }));
  
  // ❌ Don't transform or process data here
  const processedData = {
    ...initialData,
    computed: computeSomething(initialData),
  };
  
  return <EntityContextProvider initialData={processedData}>...</EntityContextProvider>;
}

// ✅ GOOD: Pass raw data, let context handle processing
export async function EntityDetailsDataProvider({ entityId }) {
  const initialData = await resolveOrThrow(fetchEntityByIdAction({ id: entityId }));
  return <EntityContextProvider initialData={initialData}>...</EntityContextProvider>;
}
```

## Quick Reference

```typescript
// Template for detail page data provider
'use server';

import { resolveOrThrow } from '@/lib/utils/server-action-helper';
import { notFound } from 'next/navigation';

export async function EntityDetailsDataProvider({ 
  entityId 
}: { 
  entityId?: string 
}) {
  const initialData = entityId
    ? await resolveOrThrow(fetchEntityByIdAction({ id: entityId }))
    : null;

  if (entityId && !initialData?.entity) {
    notFound();
  }

  return (
    <EntityContextProvider initialData={initialData}>
      <ConnectedEntityDetails isNewEntity={!entityId} />
    </EntityContextProvider>
  );
}

export type InitialEntityData = Awaited<
  ReturnType<typeof fetchEntityByIdAction>
>['data'];
```

**Complete reference**: /features/entity-details/data-providers/event-details.tsx

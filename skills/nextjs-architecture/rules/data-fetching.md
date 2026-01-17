description: Data fetching patterns with React Query, makeDataFetcher, and resolveOrThrow
alwaysApply: false
---

# Data Fetching Pattern

Consistent patterns for data fetching with React Query, the `makeDataFetcher` factory, and server actions.

## React Query Setup

### In Custom Hooks

Reference: /features/entity-list/hooks/use-fields.tsx

```typescript
'use client';

import { useQuery, skipToken } from '@tanstack/react-query';
import { resolveOrThrow } from '@/lib/utils/server-action-helper';
import { fetchItemsAction } from '../server-actions/fetch-items-action';

export function useFeature() {
  const { queryParams } = useQueryParams({ schema: QueryParamsSchema });

  const itemsQuery = useQuery({
    queryKey: ['items', JSON.stringify(queryParams)],
    queryFn: queryParams !== undefined
      ? () => resolveOrThrow(fetchItemsAction({ queryParams }))
      : skipToken, // Don't fetch if params not initialized
  });

  const items = itemsQuery.data?.items;
  const metadata = itemsQuery.data?.metadata;
  const isLoading = itemsQuery.isLoading || itemsQuery.isFetching || queryParams === undefined;

  return { items, metadata, loading: isLoading };
}
```

**Key patterns**:
- Use `resolveOrThrow` to unwrap server action responses
- Use `skipToken` for conditional queries
- Serialize complex dependencies in `queryKey`
- Destructure data carefully to avoid undefined access

### In Contexts

Reference: /features/entity-details/contexts/event-context.tsx

```typescript
import { useQuery, useMutation, skipToken } from '@tanstack/react-query';
import { resolveOrThrow } from '@/lib/utils/server-action-helper';

export const EntityContextProvider = ({ children, initialData }) => {
  const entityId = initialData?.entity?.id;

  // Query with initialData
  const {
    data: queryResult,
    isFetching,
    refetch,
  } = useQuery({
    queryKey: ['entity', entityId],
    queryFn: entityId 
      ? () => resolveOrThrow(fetchEntityByIdAction({ id: entityId }))
      : skipToken,
    initialData: entityId ? initialData : null,
  });

  const serverData = queryResult?.entity ?? null;

  return (
    <EntityContext.Provider value={{ serverData, isFetching, refetch }}>
      {children}
    </EntityContext.Provider>
  );
};
```

## Mutations Pattern

### Basic Mutation

```typescript
import { useMutation } from '@tanstack/react-query';
import { useNotifications } from '@/features/notifications';

export function useUpdateEntity() {
  const { sendCud } = useNotifications();

  const updateMutation = useMutation({
    mutationFn: (data) => 
      resolveOrThrow(updateEntityAction({ id: entityId, data })),
    onSuccess: () => {
      sendCud({
        input: {
          success: true,
          title: 'Updated',
          description: 'Changes saved successfully',
        },
      });
    },
    onError: (error) => {
      sendCud({
        input: {
          success: false,
          title: 'Update Failed',
          description: error.message || 'Failed to save changes',
        },
      });
    },
  });

  return updateMutation;
}
```

### Mutation with Refetch

```typescript
const updateMutation = useMutation({
  mutationFn: ({ id, data }) => 
    resolveOrThrow(updateEntityAction({ id, data })),
  onSuccess: () => {
    // Refetch related query
    queryClient.invalidateQueries({ queryKey: ['entity'] });
    // OR use refetch from useQuery
    refetch();
  },
});
```

### Create Mutation with Navigation

```typescript
import { useRouter } from 'next/navigation';

const createMutation = useMutation({
  mutationFn: (data) => 
    resolveOrThrow(createEntityAction({ data })),
  onSuccess: (result) => {
    sendCud({
      input: {
        success: true,
        title: 'Created',
        description: 'Entity created successfully',
      },
    });
    if (result.entity?.id) {
      router.push(`/entities/${result.entity.id}`);
    }
  },
});
```

### Delete Mutation

```typescript
const deleteMutation = useMutation({
  mutationFn: (id) => 
    resolveOrThrow(deleteEntityAction({ id })),
  onMutate: () => {
    setLoading(true);
  },
  onSettled: () => {
    setLoading(false);
  },
  onSuccess: () => {
    router.push('/entities');
    sendCud({
      input: {
        success: true,
        title: 'Deleted',
        description: 'Entity deleted successfully',
      },
    });
  },
});
```

## makeDataFetcher Pattern

Reference: /features/entity-list/utils/make-data-fetcher.ts

The `makeDataFetcher` factory creates data fetching pipelines with three stages:

1. **Query**: Fetch raw data
2. **Normalize**: Transform to expected shape
3. **Refine**: Apply filters, sorting, pagination

### Basic Usage

```typescript
import { makeDataFetcher } from '../utils/make-data-fetcher';
import { z } from 'zod';

const outputSchema = z.object({
  items: z.array(itemSchema),
  refinements: refinementsSchema,
});

const fetchData = makeDataFetcher({
  output: outputSchema,
  query: async (context) => {
    // Fetch raw data from database
    const items = await prisma.item.findMany({
      where: { organization_id: context.organizationId },
    });
    return { items };
  },
  normalize: (data, context) => {
    // Transform to expected format
    return data.items.map(item => ({
      id: item.id,
      name: item.name,
      // ... normalize fields
    }));
  },
  refine: (data, context) => {
    // Apply filters, sorting
    const filtered = applyFilters(data, context.queryParams?.filter);
    const sorted = sortItems(filtered, context.queryParams?.sortBy);
    
    return {
      items: sorted,
      refinements: {
        count: { total: data.length, filtered: sorted.length },
      },
    };
  },
});
```

### In Server Actions

Reference: /features/entity-list/server-actions/fetch-fields-action.ts

```typescript
'use server';

import { makeDataFetcher, QueryParamsSchema } from '../utils/make-data-fetcher';
import { unstable_cache } from 'next/cache';

const fetchFields = makeDataFetcher({
  output: fieldsOutputSchema,
  query: async (context) => {
    const getCachedAttributes = unstable_cache(
      async (organizationId: string) => {
        return prisma.attribute.findMany({
          where: { organization_id: organizationId },
        });
      },
      ['fields', context.organizationId],
      {
        tags: [`fields-${context.organizationId}`, 'fields'],
        revalidate: 3600,
      }
    );

    const attributes = await getCachedAttributes(context.organizationId);
    return { attributes };
  },
  normalize: ({ attributes }, context) => {
    return attributes.map(attr => ({
      id: attr.id,
      fieldName: attr.name,
      fieldType: attr.definitions[0]?.type || 'unknown',
      // ... transform fields
    }));
  },
  refine: (data, context) => {
    const filters = context.queryParams?.filter || [];
    const filteredData = applyFilters(data, filters);
    const sortedData = sortItems(filteredData, context.queryParams?.sortBy);

    return {
      items: sortedData,
      refinements: {
        count: { total: data.length, filtered: sortedData.length },
        filterOptions: buildFilterOptions(data),
      },
    };
  },
});

export const fetchFieldsAction = makeAuthorizedServerAction({
  input: z.object({
    queryParams: QueryParamsSchema.partial().nullish(),
  }),
  handler: async (args, context) => {
    return await fetchFields({
      queryParams: args.queryParams,
      organizationId: context.organization.id,
    });
  },
});
```

## Query Key Strategies

### Simple Query Keys
```typescript
queryKey: ['entity', entityId]
```

### With Parameters
```typescript
queryKey: ['entities', { status: 'active', page: 1 }]
```

### With Serialized State
```typescript
queryKey: ['entities', JSON.stringify(queryParams)]
```

### Hierarchical Keys
```typescript
queryKey: ['organization', orgId, 'entities', entityId]
```

## Conditional Queries

### Using skipToken
```typescript
const { data } = useQuery({
  queryKey: ['entity', entityId],
  queryFn: entityId 
    ? () => fetchEntity(entityId)
    : skipToken, // Don't execute query
});
```

### Using enabled
```typescript
const { data } = useQuery({
  queryKey: ['entity', entityId],
  queryFn: () => fetchEntity(entityId),
  enabled: !!entityId, // Only run if entityId exists
});
```

## Loading States

### Combined Loading State
```typescript
const { data, isLoading, isFetching, isError } = useQuery(...);

// Show loading when: initial load OR refetching OR params not ready
const loading = isLoading || isFetching || queryParams === undefined;

return { data, loading, isError };
```

### Separate States
```typescript
const { data, isLoading, isFetching } = useQuery(...);

// Initial load
if (isLoading) return <Skeleton />;

// Background refetch
if (isFetching) return <div>Updating... {data}</div>;

return <div>{data}</div>;
```

## Error Handling

### With resolveOrThrow
```typescript
// In hook
const query = useQuery({
  queryKey: ['entity'],
  queryFn: () => resolveOrThrow(fetchEntityAction()), // Throws on error
});

// React Query handles the error
if (query.isError) {
  return <div>Error: {query.error.message}</div>;
}
```

### Manual Error Handling
```typescript
const query = useQuery({
  queryKey: ['entity'],
  queryFn: async () => {
    const response = await fetchEntityAction();
    if (!response.success) {
      throw new Error(response.error);
    }
    return response.data;
  },
});
```

### In Mutations
```typescript
const mutation = useMutation({
  mutationFn: (data) => resolveOrThrow(updateAction(data)),
  onError: (error) => {
    console.error('Update failed:', error);
    sendCud({
      input: {
        success: false,
        title: 'Error',
        description: error.message,
      },
    });
  },
});
```

## Cache Management

### Invalidate Queries
```typescript
import { useQueryClient } from '@tanstack/react-query';

const queryClient = useQueryClient();

// Invalidate specific query
queryClient.invalidateQueries({ queryKey: ['entity', entityId] });

// Invalidate all queries with prefix
queryClient.invalidateQueries({ queryKey: ['entities'] });
```

### Manual Refetch
```typescript
const { refetch } = useQuery({
  queryKey: ['entity'],
  queryFn: fetchEntity,
});

// Later
await refetch();
```

### Set Query Data
```typescript
const queryClient = useQueryClient();

// Update cache directly
queryClient.setQueryData(['entity', entityId], newData);
```

## Anti-Patterns

### ❌ Don't: Missing Error Handling
```typescript
// ❌ BAD: No error handling
const { data } = useQuery({
  queryKey: ['entity'],
  queryFn: () => fetchEntityAction(), // Returns {success, data, error}
});
// data might be undefined or error response!

// ✅ GOOD: Use resolveOrThrow
const { data } = useQuery({
  queryKey: ['entity'],
  queryFn: () => resolveOrThrow(fetchEntityAction()),
});
```

### ❌ Don't: Static Query Keys with Dynamic Data
```typescript
// ❌ BAD: Query key doesn't include filter
const { data } = useQuery({
  queryKey: ['entities'], // Same key regardless of filter!
  queryFn: () => fetchEntities(filter),
});

// ✅ GOOD: Include dependencies in key
const { data } = useQuery({
  queryKey: ['entities', filter],
  queryFn: () => fetchEntities(filter),
});
```

### ❌ Don't: Forget skipToken for Conditional Queries
```typescript
// ❌ BAD: Query runs with undefined entityId
const { data } = useQuery({
  queryKey: ['entity', entityId],
  queryFn: () => fetchEntity(entityId), // Error if entityId is undefined!
});

// ✅ GOOD: Use skipToken
const { data } = useQuery({
  queryKey: ['entity', entityId],
  queryFn: entityId ? () => fetchEntity(entityId) : skipToken,
});
```

## Quick Reference

```typescript
// Query pattern
const { data, isLoading, isFetching, refetch } = useQuery({
  queryKey: ['resource', id, JSON.stringify(params)],
  queryFn: params ? () => resolveOrThrow(fetchAction(params)) : skipToken,
  initialData: initialServerData,
});

// Mutation pattern
const mutation = useMutation({
  mutationFn: (data) => resolveOrThrow(updateAction(data)),
  onSuccess: () => {
    refetch();
    sendNotification({ success: true });
  },
  onError: (error) => {
    sendNotification({ success: false, message: error.message });
  },
});
```

**Key utilities**:
- `resolveOrThrow` - @apps/web/src/lib/utils/server-action-helper.ts
- `makeDataFetcher` - /features/entity-list/utils/make-data-fetcher.ts
- `skipToken` - from `@tanstack/react-query`

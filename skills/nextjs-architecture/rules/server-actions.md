description: Server action implementation pattern using makeAuthorizedServerAction
globs:
  - "**/server-actions/**"
alwaysApply: false
---

# Server Actions Pattern

Use `makeAuthorizedServerAction` for all server actions to ensure consistent authorization, validation, and error handling.

## Basic Structure

Reference: @apps/web/src/lib/utils/make-authorized-action.ts

```typescript
'use server';

import { makeAuthorizedServerAction } from '@/lib/utils/make-authorized-action';
import { z } from 'zod';

export const myAction = makeAuthorizedServerAction({
  input: z.object({
    // Input validation with Zod
    id: z.string(),
    data: myValidationSchema,
  }),
  handler: async (args, context) => {
    // args: validated input
    // context: { organization }
    
    // Your business logic here
    const result = await doSomething(args.id, context.organization.id);
    
    return result;
  },
});
```

## Response Shape

All server actions return a consistent shape:

```typescript
type ServerActionResponse<RESULT> = 
  | {
      success: true;
      data: RESULT;
      error: null;
      message: string;
    }
  | {
      success: false;
      error: string;
      data: null;
      message: string;
    };
```

Reference: @apps/web/src/lib/utils/server-action-helper.ts

## File Organization

### Naming Convention
- File names: `[entity].ts` or `[action-name]-action.ts`
- Action names: `[verb][Entity]Action`

```
/server-actions/
├── entity.ts                   # CRUD for entities
├── fetch-items-action.ts       # Specific fetch action
├── shared.ts                   # Shared operations
└── related-entity.ts           # Related entity operations
```

### Single File Pattern
Reference: /features/entity-details/server-actions/event.ts

```typescript
'use server';

import { makeAuthorizedServerAction } from '@/lib/utils/make-authorized-action';
import { entityValidationSchema } from '../schemas/entity';
import { z } from 'zod';
import { prisma } from '@adtribute/database';

export const createEntityAction = makeAuthorizedServerAction({
  input: z.object({
    data: entityValidationSchema,
  }),
  handler: async ({ data }, { organization }) => {
    const entity = await prisma.entity.create({
      data: {
        ...data,
        organization_id: organization.id,
      },
    });
    return { entity };
  },
});

export const updateEntityAction = makeAuthorizedServerAction({
  input: z.object({
    id: z.string(),
    data: entityValidationSchema,
  }),
  handler: async ({ id, data }, { organization }) => {
    const entity = await prisma.entity.update({
      where: { id, organization_id: organization.id },
      data: {
        name: data.name,
        description: data.description,
        configuration: data.configuration,
      },
    });
    return { entity };
  },
});

export const fetchEntityByIdAction = makeAuthorizedServerAction({
  input: z.object({
    id: z.string(),
  }),
  handler: async ({ id }, { organization }) => {
    const entity = await prisma.entity.findUnique({
      where: { id, organization_id: organization.id },
    });
    return { entity };
  },
});
```

## Common Patterns

### CRUD Operations

```typescript
// CREATE
export const createEntityAction = makeAuthorizedServerAction({
  input: z.object({ data: entityValidationSchema }),
  handler: async ({ data }, { organization }) => {
    const entity = await prisma.entity.create({
      data: { ...data, organization_id: organization.id },
    });
    return { entity };
  },
});

// READ
export const fetchEntityByIdAction = makeAuthorizedServerAction({
  input: z.object({ id: z.string() }),
  handler: async ({ id }, { organization }) => {
    const entity = await prisma.entity.findUnique({
      where: { id, organization_id: organization.id },
    });
    return { entity };
  },
});

// UPDATE
export const updateEntityAction = makeAuthorizedServerAction({
  input: z.object({
    id: z.string(),
    data: entityValidationSchema,
  }),
  handler: async ({ id, data }, { organization }) => {
    const entity = await prisma.entity.update({
      where: { id, organization_id: organization.id },
      data,
    });
    return { entity };
  },
});

// DELETE
export const deleteEntityAction = makeAuthorizedServerAction({
  input: z.object({ id: z.string() }),
  handler: async ({ id }, { organization }) => {
    await prisma.entity.delete({
      where: { id, organization_id: organization.id },
    });
    return { success: true };
  },
});
```

### List/Query Operations

Reference: /features/entity-list/server-actions/fetch-fields-action.ts

```typescript
import { QueryParamsSchema } from '../utils/make-data-fetcher';

export const fetchEntitiesAction = makeAuthorizedServerAction({
  input: z.object({
    queryParams: QueryParamsSchema.partial().nullish(),
  }),
  handler: async ({ queryParams }, { organization }) => {
    const data = await fetchEntities({
      queryParams,
      organizationId: organization.id,
    });
    return data;
  },
});
```

### Actions with Optional Input

```typescript
export const fetchOptionsAction = makeAuthorizedServerAction({
  input: z.void(), // No input required
  handler: async (_, { organization }) => {
    const options = await prisma.entity.findMany({
      where: { organization_id: organization.id },
    });
    return { options };
  },
});
```

### Actions with Complex Validation

```typescript
export const fetchAvailableAttributesAction = makeAuthorizedServerAction({
  input: z.object({
    triggerDefinitionKey: z.string().optional(),
  }),
  handler: async ({ triggerDefinitionKey }, { organization }) => {
    if (!triggerDefinitionKey) {
      return { attributes: [] };
    }

    const attributes = await computeAvailableAttributes('configuration', {
      triggerDefinitionKey,
      organizationId: organization.id,
    });

    return { attributes };
  },
});
```

## Using Server Actions in Client Components

### With React Query

```typescript
'use client';

import { useQuery, useMutation } from '@tanstack/react-query';
import { resolveOrThrow } from '@/lib/utils/server-action-helper';
import { fetchEventByIdAction, updateEventAction } from '../server-actions/event';

export function useEvent(eventId: string) {
  // Query
  const eventQuery = useQuery({
    queryKey: ['event', eventId],
    queryFn: () => resolveOrThrow(fetchEventByIdAction({ id: eventId })),
  });

  // Mutation
  const updateMutation = useMutation({
    mutationFn: (data) => 
      resolveOrThrow(updateEventAction({ id: eventId, data })),
    onSuccess: () => {
      eventQuery.refetch();
    },
  });

  return { event: eventQuery.data, update: updateMutation.mutate };
}
```

### With resolveOrThrow

The `resolveOrThrow` helper unwraps the response or throws an error:

```typescript
import { resolveOrThrow } from '@/lib/utils/server-action-helper';

// Without resolveOrThrow
const response = await fetchEventByIdAction({ id: '123' });
if (!response.success) {
  throw new Error(response.error);
}
const event = response.data;

// With resolveOrThrow (cleaner)
const event = await resolveOrThrow(fetchEventByIdAction({ id: '123' }));
```

### In Mutations with Notifications

```typescript
import { useNotifications } from '@/features/notifications';

const { sendCud } = useNotifications();

const createMutation = useMutation({
  mutationFn: (data) => resolveOrThrow(createEntityAction({ data })),
  onSuccess: (result) => {
    sendCud({
      input: {
        success: true,
        title: 'Entity Created',
        description: 'The entity has been created successfully',
      },
    });
    router.push(`/entities/${result.entity.id}`);
  },
  onError: (error) => {
    sendCud({
      input: {
        success: false,
        title: 'Save Failed',
        description: error.message || 'Failed to create entity',
      },
    });
  },
});
```

## Cache Invalidation

Invalidate Next.js cache after mutations:

```typescript
import { revalidateTag } from 'next/cache';

export const updateEntityAction = makeAuthorizedServerAction({
  input: z.object({ id: z.string(), data: entityValidationSchema }),
  handler: async ({ id, data }, { organization }) => {
    const entity = await prisma.entity.update({
      where: { id },
      data,
    });
    
    // Invalidate cache
    revalidateTag(`entities-${organization.id}`);
    revalidateTag('entities');
    
    return { entity };
  },
});
```

## Data Fetching with Caching

Reference: /features/entity-list/server-actions/fetch-fields-action.ts

```typescript
import { unstable_cache } from 'next/cache';

export const fetchItemsAction = makeAuthorizedServerAction({
  input: z.object({
    queryParams: QueryParamsSchema.partial().nullish(),
  }),
  handler: async ({ queryParams }, { organization }) => {
    const getCachedItems = unstable_cache(
      async (orgId: string) => {
        return prisma.item.findMany({
          where: { organization_id: orgId },
        });
      },
      ['items', organization.id],
      {
        tags: [`items-${organization.id}`, 'items'],
        revalidate: 3600, // 1 hour
      }
    );

    const items = await getCachedItems(organization.id);
    return { items };
  },
});
```

## Error Handling

Server actions automatically catch errors and return them in the response:

```typescript
export const riskyAction = makeAuthorizedServerAction({
  input: z.object({ id: z.string() }),
  handler: async ({ id }, { organization }) => {
    // Any thrown error is caught and returned as { success: false, error: message }
    const entity = await prisma.entity.findUniqueOrThrow({
      where: { id, organization_id: organization.id },
    });
    return { entity };
  },
});
```

## Anti-Patterns

### ❌ Don't: Skip Authorization Wrapper
```typescript
// ❌ BAD: No authorization check
export async function fetchEntity(id: string) {
  'use server';
  return prisma.entity.findUnique({ where: { id } });
}

// ✅ GOOD: Use makeAuthorizedServerAction
export const fetchEntityAction = makeAuthorizedServerAction({
  input: z.object({ id: z.string() }),
  handler: async ({ id }, { organization }) => {
    const entity = await prisma.entity.findUnique({
      where: { id, organization_id: organization.id },
    });
    return { entity };
  },
});
```

### ❌ Don't: Skip Input Validation
```typescript
// ❌ BAD: No validation
export const updateEntity = makeAuthorizedServerAction({
  input: z.any(), // ❌ Too permissive!
  handler: async (data) => { ... },
});

// ✅ GOOD: Validate with Zod schema
export const updateEntity = makeAuthorizedServerAction({
  input: z.object({
    id: z.string(),
    data: entityValidationSchema,
  }),
  handler: async ({ id, data }) => { ... },
});
```

### ❌ Don't: Forget Organization Filter
```typescript
// ❌ BAD: No organization filter (security issue!)
export const fetchEntityAction = makeAuthorizedServerAction({
  input: z.object({ id: z.string() }),
  handler: async ({ id }) => {
    return prisma.entity.findUnique({ where: { id } }); // ❌ Any org can access!
  },
});

// ✅ GOOD: Always filter by organization
export const fetchEntityAction = makeAuthorizedServerAction({
  input: z.object({ id: z.string() }),
  handler: async ({ id }, { organization }) => {
    return prisma.entity.findUnique({
      where: { id, organization_id: organization.id },
    });
  },
});
```

## Quick Reference

```typescript
// Template for new server action
'use server';

import { makeAuthorizedServerAction } from '@/lib/utils/make-authorized-action';
import { z } from 'zod';

export const myAction = makeAuthorizedServerAction({
  input: z.object({
    // Define input schema
  }),
  handler: async (args, { organization }) => {
    // Implement logic
    // Always filter by organization.id
    return { result };
  },
});
```

**Key utilities**:
- `makeAuthorizedServerAction` - @apps/web/src/lib/utils/make-authorized-action.ts
- `resolveOrThrow` - @apps/web/src/lib/utils/server-action-helper.ts

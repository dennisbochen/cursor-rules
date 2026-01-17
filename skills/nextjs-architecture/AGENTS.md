# Next.js Architecture Patterns

**Version 1.0.0**  
Generic Patterns for Next.js App Router Projects  
January 2026

---

## When to Apply This Skill

**✅ Use this skill when:**
- Building Next.js projects with App Router
- Structuring features and folders
- Creating server actions with authorization
- Building React context providers
- Setting up data providers for SSR
- Implementing useReducer for complex state
- Defining Zod schemas for validation
- Organizing components (presentational vs connected)

**❌ Do NOT use this skill when:**
- Working with Next.js Pages Router (`pages/` directory)
- Not using Next.js (React-only, Vue, Angular, etc.)
- Building backend-only APIs without frontend

---

> **Note:**  
> This document contains comprehensive architectural patterns for Next.js App Router projects.  
> These are generic patterns adaptable to any Next.js project, not specific to any codebase.

---

## Table of Contents

1. [Feature Architecture](#1-feature-architecture) - Folder structure and organization
2. [Code Style](#2-code-style) - TypeScript and React conventions
3. [Component Naming](#3-component-naming) - Presentational vs Connected components
4. [Context Pattern](#4-context-pattern) - React Context for state management
5. [Data Fetching](#5-data-fetching) - React Query and server action patterns
6. [Data Providers](#6-data-providers) - Server-side data fetching for SSR
7. [Server Actions](#7-server-actions) - Authorized server action implementation
8. [useReducer Pattern](#8-usereducer-pattern) - Complex state management
9. [Zod Schemas](#9-zod-schemas) - Dual schema pattern for validation

---

## 1. Feature Architecture

All features should follow this consistent folder structure for maintainability and scalability.

### Standard Feature Structure

```
/features/[feature-name]/
├── components/        # Presentational (dumb) components
├── connected/         # Connected (smart) components
├── contexts/          # React Context providers (optional)
├── data-providers/    # Server Components for SSR (optional)
├── hooks/             # Custom React hooks
├── schemas/           # Zod schemas for validation
├── server-actions/    # Next.js server actions (optional)
├── utils/             # Helper functions and utilities
├── constants/         # Constants and type definitions
└── index.ts           # Public API - re-exports
```

### Folder Guidelines

**`/components/`** - Presentational (dumb) components
- Pure UI components that receive data via props
- No direct data fetching or business logic
- Highly reusable

**`/connected/`** - Connected (smart) components
- Wire data from hooks/contexts to presentational components
- File naming: `connected-[name].tsx` exports `Connected[Name]`

**`/contexts/`** - React Context providers
- Feature-level state management
- Includes context definition, provider, and custom hook

**`/data-providers/`** - Server Components for SSR
- Fetch initial data for SSR
- Wrap client components with context providers
- Export type for initial data shape

**`/hooks/`** - Custom React hooks
- State management and side effects
- Complex hooks use useReducer pattern

**`/schemas/`** - Zod schemas
- Validation and type generation
- Follow dual schema pattern (type + validation)

**`/server-actions/`** - Next.js server actions
- Use 'use server' directive
- Use `makeAuthorizedServerAction` helper

**`/utils/`** - Pure utility functions
- No React dependencies

**`/constants/`** - Constants and type definitions

**`index.ts`** - Public API
- Re-export only public-facing items
- Don't expose internal implementation

### Creating a New Feature

1. Create the feature folder: `features/[feature-name]/`
2. Add `index.ts` first - define public API
3. Create folders as needed (not all required)
4. Start with `schemas/` to define data types
5. Build `components/` for UI
6. Add `connected/` to wire data to UI
7. Add other folders as complexity requires

### Anti-Patterns to Avoid

- ❌ Don't put business logic in presentational components
- ❌ Don't import from other feature's internals (only use `index.ts`)
- ❌ Don't create circular dependencies between features
- ❌ Don't mix server and client code in the same file
- ❌ Don't skip the `index.ts` - always provide clean public API

---

## 2. Code Style

### TypeScript Conventions

**Always Use Functional React:**

```typescript
// ✅ GOOD
export function MyComponent({ name }: { name: string }) {
  return <div>{name}</div>;
}

// ❌ BAD
export class MyComponent extends React.Component {
  render() { return <div>{this.props.name}</div>; }
}
```

**Type Definitions:**

```typescript
// Simple props: inline type
export function Button({ label, onClick }: { 
  label: string; 
  onClick: () => void;
}) {
  return <button onClick={onClick}>{label}</button>;
}

// Complex props: separate type
type EntityListProps = {
  entities: Entity[];
  loading: boolean;
  onEntityClick: (id: string) => void;
  filterOptions?: FilterOptions;
};
```

**Type Inference:**

```typescript
// Let TypeScript infer when obvious
const [count, setCount] = useState(0); // Infers number

// Explicit type when needed
const [user, setUser] = useState<User | null>(null);
```

### React Patterns

**Component Structure:**

```typescript
'use client'; // Only for client components

import { useState, useEffect, useCallback } from 'react';

type ComponentProps = {
  // Props definition
};

export function Component({ prop1, prop2 }: ComponentProps) {
  // 1. Hooks
  const [state, setState] = useState();
  
  // 2. Derived values
  const derivedValue = useMemo(() => compute(state), [state]);
  
  // 3. Callbacks
  const handleClick = useCallback(() => {}, []);
  
  // 4. Effects
  useEffect(() => {}, []);
  
  // 5. Early returns
  if (loading) return <LoadingSpinner />;
  
  // 6. Render
  return <div>{/* JSX */}</div>;
}
```

**Hooks Usage:**

```typescript
// ✅ useCallback for event handlers
const handleClick = useCallback((id: string) => {
  dispatch({ type: 'update', payload: { id } });
}, []);

// ✅ useMemo for expensive computations
const filteredItems = useMemo(() => {
  return items.filter(item => item.active);
}, [items]);
```

### Naming Conventions

- **Variables/Functions**: camelCase (`userName`, `calculateTotal`)
- **Constants**: UPPER_SNAKE_CASE (`MAX_RETRY_COUNT`)
- **Components**: PascalCase (`EntityList`, `ConnectedFields`)
- **Types/Interfaces**: PascalCase (`UserData`, `ComponentState`)
- **Files**: kebab-case (`entity-list.tsx`, `use-form.tsx`)

### Import Organization

```typescript
// 1. React imports
import { useState, useEffect } from 'react';

// 2. External libraries
import { useQuery } from '@tanstack/react-query';

// 3. Internal packages (monorepo)
import { Button } from '@company/ui';

// 4. Absolute imports from app
import { resolveOrThrow } from '@/lib/utils';

// 5. Relative imports (feature-local)
import { EntityList } from '../components/entity-list';
```

---

## 3. Component Naming

### File Naming

**Presentational Components:**
- Location: `/features/[feature-name]/components/`
- File: `[name].tsx` (kebab-case)
- Export: PascalCase function name

```typescript
// components/entity-list.tsx
export function EntityList({ children }: { children: React.ReactNode }) {
  return <ul>{children}</ul>;
}
```

**Connected Components:**
- Location: `/features/[feature-name]/connected/`
- File: `connected-[name].tsx` (with prefix)
- Export: `Connected[Name]` (with prefix)

```typescript
// connected/connected-feature.tsx
export function ConnectedFeature() {
  const { items, loading } = useFeature();
  return <FeatureList>{/* render */}</FeatureList>;
}
```

### Component Separation

**Presentational Components (Dumb):**
- Receive ALL data via props
- No hooks except UI-only hooks (useState for local UI)
- No data fetching (no useQuery, no server actions)
- No business logic
- Highly reusable

**Connected Components (Smart):**
- Use custom hooks or contexts to get data
- Wire data to presentational components
- Handle user interactions
- May use React Query, server actions
- Feature-specific (less reusable)

### Anti-Patterns

```typescript
// ❌ Data fetching in presentational components
export function FieldsList() {
  const { data } = useQuery(['fields'], fetchFields);
  return <ul>{data?.map(...)}</ul>;
}

// ❌ Business logic in presentational components
export function EntityList({ items }) {
  const filteredItems = items.filter(item => item.active);
  return <ul>{filteredItems.map(...)}</ul>;
}
```

---

## 4. Context Pattern

React Context pattern for managing entity state with draft changes, server data, and validation.

### Basic Structure

```typescript
'use client';

import { createContext, useContext, useState, useMemo, useCallback } from 'react';

// 1. Define context type
export type EntityContextType = {
  currentData: EntityState;
  serverData: Entity | null;
  isFetching: boolean;
  hasChanges: boolean;
  setName: (name: string) => void;
  saveEntity: () => Promise<void>;
  isSaving: boolean;
  errors: string[] | null;
};

// 2. Create context with defaults
export const EntityContext = createContext<EntityContextType>({
  currentData: {},
  serverData: null,
  isFetching: false,
  hasChanges: false,
  setName: () => {},
  saveEntity: async () => {},
  isSaving: false,
  errors: null,
});

// 3. Provider component
export const EntityContextProvider = ({ children, initialData }) => {
  // Implementation
  return <EntityContext.Provider value={{ ... }}>{children}</EntityContext.Provider>;
};

// 4. Custom hook
export const useEntityContext = () => {
  const context = useContext(EntityContext);
  if (!context) {
    throw new Error('useEntityContext must be used within provider');
  }
  return context;
};
```

### Draft State Pattern

Keep draft changes separate from server data:

```typescript
export const EntityContextProvider = ({ children, initialData }) => {
  // Server state (via React Query)
  const { data: queryResult, isFetching } = useQuery({
    queryKey: ['entity', entityId],
    queryFn: () => resolveOrThrow(fetchEntityAction({ id: entityId })),
    initialData,
  });

  const serverData = queryResult?.entity ?? null;

  // Draft state (only stores user changes)
  const [draftChanges, setDraftChanges] = useState<Partial<EntityState>>({});

  // Merge server + draft
  const currentData: EntityState = useMemo(() => ({
    id: serverData?.id,
    name: draftChanges.name ?? serverData?.name ?? 'New Entity',
    configuration: {
      ...serverData?.configuration,
      ...draftChanges.configuration,
    },
  }), [serverData, draftChanges]);

  // State updaters
  const setName = useCallback((name: string) => {
    setDraftChanges((prev) => ({ ...prev, name }));
  }, []);

  return (
    <EntityContext.Provider value={{ currentData, serverData, setName, ... }}>
      {children}
    </EntityContext.Provider>
  );
};
```

### Validation Pattern

```typescript
import { formatZodErrors } from '@/lib/utils/zod/format-errors';

const [errors, setErrors] = useState<string[] | null>(null);

useEffect(() => {
  const result = entityValidationSchema.safeParse(currentData);
  if (!result.success) {
    setErrors(formatZodErrors(result.error));
  } else {
    setErrors(null);
  }
}, [currentData]);
```

---

## 5. Data Fetching

### React Query Pattern

```typescript
'use client';

import { useQuery, skipToken } from '@tanstack/react-query';
import { resolveOrThrow } from '@/lib/utils/server-action-helper';

export function useFeature() {
  const { queryParams } = useQueryParams();

  const itemsQuery = useQuery({
    queryKey: ['items', JSON.stringify(queryParams)],
    queryFn: queryParams !== undefined
      ? () => resolveOrThrow(fetchItemsAction({ queryParams }))
      : skipToken,
  });

  return { 
    items: itemsQuery.data?.items,
    loading: itemsQuery.isLoading || itemsQuery.isFetching
  };
}
```

### Mutations Pattern

```typescript
import { useMutation } from '@tanstack/react-query';

const updateMutation = useMutation({
  mutationFn: (data) => resolveOrThrow(updateEntityAction({ id, data })),
  onSuccess: () => {
    refetch();
    sendNotification({ success: true });
  },
  onError: (error) => {
    sendNotification({ success: false, message: error.message });
  },
});
```

---

## 6. Data Providers

Data providers are Server Components that fetch initial data and wrap client components with context providers.

### Basic Pattern

```typescript
'use server';

import { resolveOrThrow } from '@/lib/utils/server-action-helper';
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

  // 3. Wrap with context and render connected component
  return (
    <EntityContextProvider initialData={initialData}>
      <ConnectedEntityDetails isNewEntity={!entityId} />
    </EntityContextProvider>
  );
}

// 4. Export type for initial data
export type InitialEntityData = Awaited<
  ReturnType<typeof fetchEntityByIdAction>
>['data'];
```

---

## 7. Server Actions

Use `makeAuthorizedServerAction` for all server actions.

### Basic Structure

```typescript
'use server';

import { makeAuthorizedServerAction } from '@/lib/utils/make-authorized-action';
import { z } from 'zod';

export const myAction = makeAuthorizedServerAction({
  input: z.object({
    id: z.string(),
    data: myValidationSchema,
  }),
  handler: async (args, context) => {
    // args: validated input
    // context: { organization }
    
    const result = await doSomething(args.id, context.organization.id);
    return result;
  },
});
```

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
  input: z.object({ id: z.string(), data: entityValidationSchema }),
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

---

## 8. useReducer Pattern

Follow this structure for all complex state management hooks.

### File Structure

```
/hooks/use-[feature-name]/
├── index.ts           # Main hook implementation
├── actions.ts         # Action creators and handlers
├── reducer.ts         # Reducer function
└── schema/
    ├── schema.ts      # Type schema (permissive)
    └── validationSchema.ts  # Validation schema (strict)
```

### Schema Definition

```typescript
// schema/schema.ts - Permissive
export const ruleSchema = z.object({
  id: z.string().default(() => generateId()),
  left: z.object({
    type: z.literal('attribute'),
    attribute: z.string(),
  }).nullish(),
  operator: ruleOperatorSchema.nullish(),
});

export type Rule = z.infer<typeof ruleSchema>;
```

### Actions Definition

```typescript
// actions.ts
export type UpdateConditionAction = {
  type: 'update-condition';
  payload: { conditionId: string; rule: Rule };
};

export type Actions = UpdateConditionAction | /* other actions */;

export function updateCondition(
  state: FilterState, 
  action: UpdateConditionAction
): FilterState {
  return {
    ...state,
    conditions: state.conditions.map((condition) => {
      if (condition.id !== action.payload.conditionId) return condition;
      return { ...condition, rule: { ...condition.rule, ...action.payload.rule } };
    }),
  };
}
```

### Reducer Implementation

```typescript
// reducer.ts
export function reducer(state: FilterState, action: Actions): FilterState {
  switch (action.type) {
    case 'update-condition':
      return updateCondition(state, action);
    default:
      return state;
  }
}
```

### Main Hook

```typescript
// index.ts
'use client';

import { useCallback, useReducer } from 'react';
import { reducer } from './reducer';

export function useFilter({ initialState }: UseFilterProps) {
  const [state, dispatch] = useReducer(reducer, initialState);

  const updateCondition = useCallback(
    (payload: { conditionId: string; rule: Rule }) => {
      dispatch({ type: 'update-condition', payload });
    }, 
    []
  );

  return { state, updateCondition };
}
```

---

## 9. Zod Schemas

Use dual schema approach: one permissive schema for UI state, one strict schema for validation.

### Dual Schema Pattern

```typescript
// Permissive TYPE SCHEMA - for UI state
export const entityConfigTypeSchema = z.object({
  name: z.string().optional(),
  status: z.string().optional(),
  settings: settingsTypeSchema.optional(),
});

// Strict VALIDATION SCHEMA - for saving
export const entityConfigValidationSchema = z.object({
  name: z
    .string({ required_error: 'Please enter a name' })
    .min(1, 'Name is required'),
  status: z
    .string({ required_error: 'Please select a status' })
    .min(1, 'Status is required'),
  settings: settingsValidationSchema,
});

// Export types
export type EntityConfig = z.infer<typeof entityConfigTypeSchema>;
export type ValidatedEntityData = z.infer<typeof entityConfigValidationSchema>;
```

### Usage in Contexts

```typescript
import { entityStateSchema, entityValidationSchema } from '../schemas/entity';

export const EntityContextProvider = ({ children, initialData }) => {
  // currentData follows type schema shape
  const currentData: EntityState = useMemo(() => ({
    id: serverData?.id,
    name: draftChanges.name ?? serverData?.name ?? 'New Entity',
  }), [serverData, draftChanges]);

  // Validation runs against validation schema
  useEffect(() => {
    const result = entityValidationSchema.safeParse(currentData);
    if (!result.success) {
      setErrors(formatZodErrors(result.error));
    } else {
      setErrors(null);
    }
  }, [currentData]);

  return <EntityContext.Provider value={{ currentData, errors, ... }}>
    {children}
  </EntityContext.Provider>;
};
```

### Usage in Server Actions

```typescript
export const createEntityAction = makeAuthorizedServerAction({
  input: z.object({
    data: entityValidationSchema, // Use validation schema
  }),
  handler: async ({ data }, { organization }) => {
    // data is guaranteed to be valid
    const entity = await prisma.entity.create({ data: { ...data } });
    return { entity };
  },
});
```

---

## Quick Reference Summary

| Pattern | When to Use |
|---------|-------------|
| Feature Architecture | Structuring any new feature |
| Code Style | All TypeScript/React code |
| Component Naming | Creating components |
| Context Pattern | Complex entity state management |
| Data Fetching | Client-side data needs |
| Data Providers | SSR with initial data |
| Server Actions | Backend operations |
| useReducer Pattern | Complex state with many actions |
| Zod Schemas | Type definitions + validation |

---

## Additional Resources

For detailed examples and edge cases, refer to individual rule files in the `rules/` directory.

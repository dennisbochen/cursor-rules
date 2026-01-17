description: React Context provider pattern for entity state management with draft/server state separation
globs:
  - "**/contexts/**"
alwaysApply: false
---

# Context Pattern

React Context pattern for managing entity state with draft changes, server data, and validation.

## File Structure

Each context file should include:
1. Context type definition
2. Context creation with default values
3. Provider component
4. Custom hook for consuming context

Reference: /features/entity-details/contexts/event-context.tsx

## Basic Structure

```typescript
'use client';

import { createContext, useContext, useState, useMemo, useCallback, useEffect } from 'react';
import { useQuery, useMutation } from '@tanstack/react-query';

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

// 2. Create context with default values
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
  // Implementation (see below)
  return (
    <EntityContext.Provider value={{ ... }}>
      {children}
    </EntityContext.Provider>
  );
};

// 4. Custom hook
export const useEntityContext = () => {
  const context = useContext(EntityContext);
  if (!context) {
    throw new Error('useEntityContext must be used within EntityContextProvider');
  }
  return context;
};
```

## Draft State Pattern

Key principle: Keep draft changes separate from server data.

```typescript
export const EntityContextProvider = ({ children, initialData }) => {
  const entityId = initialData?.entity?.id;

  // Server state (via React Query)
  const { data: queryResult, isFetching, refetch } = useQuery({
    queryKey: ['entity', entityId],
    queryFn: entityId 
      ? () => resolveOrThrow(fetchEntityByIdAction({ id: entityId }))
      : skipToken,
    initialData: entityId ? initialData : null,
  });

  const serverData = queryResult?.entity ?? null;

  // Draft state (only stores user changes)
  const [draftChanges, setDraftChanges] = useState<Partial<EntityState>>({});

  // Merge server + draft to get current state
  const currentData: EntityState = useMemo(() => ({
    id: serverData?.id,
    name: draftChanges.name ?? serverData?.name ?? 'New Entity',
    description: draftChanges.description ?? serverData?.description ?? '',
    configuration: {
      ...serverData?.configuration,
      ...draftChanges.configuration,
    },
  }), [serverData, draftChanges]);

  // Track changes
  const hasChanges = useMemo(() => {
    if (serverData) {
      const serverState: EntityState = {
        id: serverData.id,
        name: serverData.name ?? 'New Entity',
        description: serverData.description ?? '',
        configuration: serverData.configuration,
      };
      return !isEqual(currentData, serverState);
    } else {
      const defaultState: EntityState = {
        id: undefined,
        name: 'New Entity',
        description: '',
        configuration: {},
      };
      return !isEqual(currentData, defaultState);
    }
  }, [currentData, serverData]);

  return (
    <EntityContext.Provider value={{ currentData, serverData, hasChanges, ... }}>
      {children}
    </EntityContext.Provider>
  );
};
```

## State Update Pattern

Use `useCallback` with functional updates to avoid stale closures:

```typescript
// Simple field update
const setName = useCallback((name: string) => {
  setDraftChanges((prev) => ({ ...prev, name }));
}, []);

const setDescription = useCallback((description: string) => {
  setDraftChanges((prev) => ({ ...prev, description }));
}, []);

// Partial nested update
const setConfiguration = useCallback((updates: Partial<EntityConfig>) => {
  setDraftChanges((prev) => ({
    ...prev,
    configuration: {
      ...prev.configuration,
      ...updates,
    },
  }));
}, []);

// Reset to server state
const resetChanges = useCallback(() => {
  setDraftChanges({});
}, []);
```

## Validation Pattern

Validate on state changes using `useEffect`:

```typescript
import { formatZodErrors } from '~/lib/utils/zod/format-errors';

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

## Mutation Pattern

### Create and Update Mutations

```typescript
import { useMutation } from '@tanstack/react-query';
import { useRouter } from 'next/navigation';
import { useNotifications } from '@/features/notifications';

const router = useRouter();
const { sendCud } = useNotifications();

// Create mutation
const createMutation = useMutation({
  mutationFn: (validatedData: ValidatedEntityData) =>
    resolveOrThrow(createEntityAction({ data: validatedData })),
  onSuccess: (result) => {
    sendCud({
      input: {
        success: true,
        title: 'Entity Created',
        description: 'The entity has been created successfully',
      },
    });
    if (result.entity?.id) {
      router.push(`/entities/${result.entity.id}`);
    }
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

// Update mutation
const updateMutation = useMutation({
  mutationFn: ({ id, validatedData }) =>
    resolveOrThrow(updateEntityAction({ id, data: validatedData })),
  onSuccess: () => {
    refetch(); // Refetch server data
    setDraftChanges({}); // Clear draft changes
    sendCud({
      input: {
        success: true,
        title: 'Entity Updated',
        description: 'Your changes have been saved',
      },
    });
  },
  onError: (error) => {
    sendCud({
      input: {
        success: false,
        title: 'Save Failed',
        description: error.message || 'Failed to save entity',
      },
    });
  },
});
```

### Save Handler

```typescript
const saveEntity = useCallback(async () => {
  // Validate before saving
  const validationResult = entityValidationSchema.safeParse(currentData);
  if (!validationResult.success) {
    return; // Errors already shown via validation effect
  }

  if (currentData.id) {
    // Update existing
    updateMutation.mutate({
      id: currentData.id,
      validatedData: validationResult.data,
    });
  } else {
    // Create new
    createMutation.mutate(validationResult.data);
  }
}, [currentData, updateMutation, createMutation]);

const isSaving = createMutation.isPending || updateMutation.isPending;
```

### Delete Mutation

```typescript
const [contextMenuOpen, setContextMenuOpen] = useState(false);
const [contextMenuLoading, setContextMenuLoading] = useState(false);

const deleteMutation = useMutation({
  mutationFn: () => {
    if (!currentData.id) throw new Error('Entity ID is required');
    return resolveOrThrow(deleteEntityAction({ id: currentData.id }));
  },
  onMutate: () => {
    setContextMenuLoading(true);
  },
  onSettled: () => {
    setContextMenuLoading(false);
  },
  onSuccess: () => {
    router.push('/entities');
    sendCud({
      input: {
        success: true,
        title: 'Entity Deleted',
        description: 'The entity has been deleted successfully',
      },
    });
  },
  onError: (error) => {
    sendCud({
      input: {
        success: false,
        title: 'Delete Failed',
        description: error.message || 'Failed to delete entity',
      },
    });
  },
});

const handleDelete = () => {
  deleteMutation.mutate();
};
```

## Validation on Load (Optional)

For edit views, validate that the entity exists and is the correct type:

```typescript
import { notFound } from 'next/navigation';

useEffect(() => {
  if (entityId && !isFetching && queryResult) {
    if (!serverData) {
      notFound(); // 404 if entity not found
    }
    if (serverData.type !== 'expected_type') {
      console.error('Wrong entity type', serverData);
      notFound();
    }
  }
}, [entityId, isFetching, queryResult, serverData]);
```

## Complete Example

Reference: /features/entity-details/contexts/event-context.tsx

```typescript
'use client';

import { createContext, useContext, useState, useMemo, useCallback, useEffect } from 'react';
import { useQuery, useMutation, skipToken } from '@tanstack/react-query';
import { resolveOrThrow } from '@/lib/utils/server-action-helper';
import { formatZodErrors } from '~/lib/utils/zod/format-errors';
import { useNotifications } from '@/features/notifications';
import { useRouter } from 'next/navigation';
import { isEqual } from 'lodash';

export type EntityContextType = {
  currentData: EntityState;
  serverData: Entity | null;
  isFetching: boolean;
  hasChanges: boolean;
  setName: (name: string) => void;
  setDescription: (description: string) => void;
  setConfiguration: (configuration: Partial<EntityConfig>) => void;
  resetChanges: () => void;
  errors: string[] | null;
  saveEntity: () => Promise<void>;
  isSaving: boolean;
  handleDelete: () => void;
};

export const EntityContext = createContext<EntityContextType>({ /* defaults */ });

export const EntityContextProvider = ({ children, initialData }) => {
  const entity = initialData?.entity ?? null;
  const router = useRouter();
  const { sendCud } = useNotifications();

  // Server state
  const { data: queryResult, isFetching, refetch } = useQuery({
    queryKey: ['entity', entity?.id],
    queryFn: entity ? () => resolveOrThrow(fetchEntityByIdAction({ id: entity.id })) : skipToken,
    initialData: entity ? initialData : null,
  });

  const serverData = queryResult?.entity ?? null;

  // Draft state
  const [draftChanges, setDraftChanges] = useState<Partial<EntityState>>({});
  const [errors, setErrors] = useState<string[] | null>(null);

  // Current state (merged)
  const currentData: EntityState = useMemo(() => ({
    id: serverData?.id,
    name: draftChanges.name ?? serverData?.name ?? 'New Entity',
    description: draftChanges.description ?? serverData?.description ?? '',
    configuration: {
      ...serverData?.configuration,
      ...draftChanges.configuration,
    },
  }), [serverData, draftChanges]);

  // Has changes
  const hasChanges = useMemo(() => {
    if (serverData) {
      const serverState: EntityState = { /* ... */ };
      return !isEqual(currentData, serverState);
    } else {
      const defaultState: EntityState = { /* ... */ };
      return !isEqual(currentData, defaultState);
    }
  }, [currentData, serverData]);

  // State updaters
  const setName = useCallback((name: string) => {
    setDraftChanges((prev) => ({ ...prev, name }));
  }, []);

  const setDescription = useCallback((description: string) => {
    setDraftChanges((prev) => ({ ...prev, description }));
  }, []);

  const setConfiguration = useCallback((updates: Partial<EntityConfig>) => {
    setDraftChanges((prev) => ({
      ...prev,
      configuration: { ...prev.configuration, ...updates },
    }));
  }, []);

  const resetChanges = useCallback(() => {
    setDraftChanges({});
  }, []);

  // Validation
  useEffect(() => {
    const result = entityValidationSchema.safeParse(currentData);
    setErrors(result.success ? null : formatZodErrors(result.error));
  }, [currentData]);

  // Mutations
  const createMutation = useMutation({ /* ... */ });
  const updateMutation = useMutation({ /* ... */ });
  const deleteMutation = useMutation({ /* ... */ });

  const saveEntity = useCallback(async () => {
    const result = entityValidationSchema.safeParse(currentData);
    if (!result.success) return;

    if (currentData.id) {
      updateMutation.mutate({ id: currentData.id, validatedData: result.data });
    } else {
      createMutation.mutate(result.data);
    }
  }, [currentData, updateMutation, createMutation]);

  const isSaving = createMutation.isPending || updateMutation.isPending;

  return (
    <EntityContext.Provider
      value={{
        currentData,
        serverData,
        isFetching,
        hasChanges,
        setName,
        setDescription,
        setConfiguration,
        resetChanges,
        errors,
        saveEntity,
        isSaving,
        handleDelete: () => deleteMutation.mutate(),
      }}
    >
      {children}
    </EntityContext.Provider>
  );
};

export const useEntityContext = () => {
  const context = useContext(EntityContext);
  if (!context) {
    throw new Error('useEntityContext must be used within EntityContextProvider');
  }
  return context;
};
```

## Anti-Patterns

### ❌ Don't: Store Full Copy of Server Data in State
```typescript
// ❌ BAD: Duplicates server data
const [localData, setLocalData] = useState(serverData);

// ✅ GOOD: Only store changes
const [draftChanges, setDraftChanges] = useState({});
const currentData = useMemo(() => ({ ...serverData, ...draftChanges }), [serverData, draftChanges]);
```

### ❌ Don't: Missing Error Boundary in useContext Hook
```typescript
// ❌ BAD: No error handling
export const useEntityContext = () => {
  return useContext(EntityContext); // Could be undefined!
};

// ✅ GOOD: Throw error if used outside provider
export const useEntityContext = () => {
  const context = useContext(EntityContext);
  if (!context) {
    throw new Error('useEntityContext must be used within EntityContextProvider');
  }
  return context;
};
```

### ❌ Don't: Forget Functional Updates
```typescript
// ❌ BAD: Stale closure
const setName = useCallback((name: string) => {
  setDraftChanges({ ...draftChanges, name }); // ❌ Captures stale draftChanges
}, []); // Empty deps but uses draftChanges!

// ✅ GOOD: Functional update
const setName = useCallback((name: string) => {
  setDraftChanges((prev) => ({ ...prev, name }));
}, []);
```

## Quick Reference

```typescript
// Context structure
export const EntityContext = createContext<EntityContextType>({ /* defaults */ });

export const EntityContextProvider = ({ children, initialData }) => {
  // 1. Query for server data
  // 2. Draft changes state
  // 3. Merge to current data
  // 4. Track hasChanges
  // 5. State updaters with useCallback
  // 6. Validation with useEffect
  // 7. Mutations for save/delete
  return <EntityContext.Provider value={{ ... }}>{children}</EntityContext.Provider>;
};

export const useEntityContext = () => {
  const context = useContext(EntityContext);
  if (!context) throw new Error('Must be used within provider');
  return context;
};
```

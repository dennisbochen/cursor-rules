description: TypeScript, React, and code style conventions for the Next.js projects
alwaysApply: true
---

# Code Style Conventions

Comprehensive code style guide for TypeScript and React in the Next.js projects.

## TypeScript

### Always Use Functional React

```typescript
// ✅ GOOD: Functional component
export function MyComponent({ name }: { name: string }) {
  return <div>{name}</div>;
}

// ❌ BAD: Class component
export class MyComponent extends React.Component {
  render() { return <div>{this.props.name}</div>; }
}
```

### Type Definitions

```typescript
// ✅ GOOD: Inline type for simple props
export function Button({ label, onClick }: { 
  label: string; 
  onClick: () => void;
}) {
  return <button onClick={onClick}>{label}</button>;
}

// ✅ GOOD: Separate type for complex props
type EntityListProps = {
  entities: Entity[];
  loading: boolean;
  onEntityClick: (id: string) => void;
  filterOptions?: FilterOptions;
};

export function EntityList({ 
  entities, 
  loading, 
  onEntityClick,
  filterOptions,
}: EntityListProps) {
  // ...
}
```

### Type Inference

```typescript
// ✅ GOOD: Let TypeScript infer when obvious
const [count, setCount] = useState(0); // Infers number
const [name, setName] = useState(''); // Infers string

// ✅ GOOD: Explicit type when needed
const [user, setUser] = useState<User | null>(null);
const [errors, setErrors] = useState<string[] | undefined>(undefined);
```

### Type Exports

```typescript
// Export types from schemas
export type EntityState = z.infer<typeof entityStateSchema>;
export type ValidatedEntityData = z.infer<typeof entityValidationSchema>;

// Export types from server actions
export type InitialEntityData = Awaited<
  ReturnType<typeof fetchEntityByIdAction>
>['data'];
```

## React Patterns

### Component Structure

```typescript
'use client'; // Only for client components

import { useState, useEffect, useCallback } from 'react';
import { ExternalLib } from 'external-lib';
import { InternalUtil } from '@/lib/utils';
import { LocalComponent } from './local-component';

type ComponentProps = {
  // Props type definition
};

export function Component({ prop1, prop2 }: ComponentProps) {
  // 1. Hooks
  const [state, setState] = useState();
  const customHook = useCustomHook();
  
  // 2. Derived values
  const derivedValue = useMemo(() => compute(state), [state]);
  
  // 3. Callbacks
  const handleClick = useCallback(() => {
    // Handler logic
  }, []);
  
  // 4. Effects
  useEffect(() => {
    // Effect logic
  }, []);
  
  // 5. Early returns (loading, error states)
  if (loading) return <LoadingSpinner />;
  if (error) return <ErrorMessage />;
  
  // 6. Render
  return (
    <div>
      {/* JSX */}
    </div>
  );
}
```

### Hooks Usage

```typescript
// ✅ GOOD: useCallback for event handlers
const handleClick = useCallback((id: string) => {
  dispatch({ type: 'update', payload: { id } });
}, []);

// ✅ GOOD: useMemo for expensive computations
const filteredItems = useMemo(() => {
  return items.filter(item => item.active);
}, [items]);

// ✅ GOOD: useRef for values that don't trigger re-renders
const onChangeRef = useRef(onChange);
onChangeRef.current = onChange;

// ❌ BAD: Missing useCallback for passed callbacks
const handleClick = (id: string) => { // Will create new function on every render
  dispatch({ type: 'update', payload: { id } });
};
```

### Conditional Rendering

```typescript
// ✅ GOOD: Early returns for loading/error states
if (loading) return <LoadingSpinner />;
if (error) return <ErrorMessage error={error} />;

// ✅ GOOD: Short-circuit for optional elements
{isAdmin && <AdminPanel />}

// ✅ GOOD: Ternary for either/or
{isLoggedIn ? <Dashboard /> : <LoginForm />}

// ❌ BAD: Nested ternaries (hard to read)
{isLoggedIn ? isAdmin ? <AdminDashboard /> : <UserDashboard /> : <LoginForm />}
```

## Naming Conventions

### Variables and Functions

```typescript
// camelCase for variables and functions
const userName = 'John';
const itemCount = 10;
function calculateTotal() { }
const handleSubmit = () => { };
```

### Constants

```typescript
// UPPER_SNAKE_CASE for true constants
const MAX_RETRY_COUNT = 3;
const API_BASE_URL = 'https://api.example.com';

// camelCase for derived constants or configuration objects
const defaultState = { count: 0 };
const buttonStyles = { padding: '10px' };
```

### Components

```typescript
// PascalCase for components
export function EntityList() { }
export function ConnectedFields() { }
```

### Types and Interfaces

```typescript
// PascalCase for types and interfaces
type UserData = { };
type EntityListProps = { };
interface ComponentState { }
```

### Files

```typescript
// kebab-case for files
entity-list.tsx
connected-fields.tsx
use-entity-form.tsx
fetch-data-action.ts
```

## Import Organization

```typescript
// 1. React imports
import { useState, useEffect, useCallback } from 'react';

// 2. External libraries
import { useQuery } from '@tanstack/react-query';
import { z } from 'zod';

// 3. Internal packages (monorepo)
import { Button } from '@adtribute/ui/new';
import { prisma } from '@adtribute/database';

// 4. Absolute imports from app
import { resolveOrThrow } from '@/lib/utils/server-action-helper';
import { useNotifications } from '@/features/notifications';

// 5. Relative imports (feature-local)
import { EntityList } from '../components/entity-list';
import { useEntityContext } from '../contexts/entity-context';
import { type EntityState } from '../schemas/entity';
```

## Code Comments

### When to Comment

```typescript
// ✅ GOOD: Explain WHY, not WHAT
// Skip initial render to avoid calling onChange with initialValue
if (isFirstRender.current) {
  isFirstRender.current = false;
  return;
}

// ✅ GOOD: Explain business rules
// Identity attribute is always implicitly included in deduplication,
// so once_per can be empty (identity alone is valid)
const deduplicationSchema = z.object({ once_per: z.array(z.string()).optional() });

// ✅ GOOD: Document complex logic
// Warning: this is legacy naming. In the frontend, we use the definition type 'conversion' for events.
if (serverData.type !== 'conversion') {
  notFound();
}
```

### Avoid Obvious Comments

```typescript
// ❌ BAD: Comment states the obvious
// Set the name
setName(newName);

// ❌ BAD: Comment duplicates code
// Loop through items
items.forEach(item => { });

// ✅ GOOD: No comment needed - code is self-explanatory
setName(newName);
items.forEach(item => { });
```

### Don't Remove Code Comments

Per user rules: **Don't remove code comments** - preserve existing comments when making changes.

## JSX Conventions

### Prop Formatting

```typescript
// ✅ GOOD: One prop per line when many props
<EntityListItem
  key={entity.id}
  href={`/entities/${entity.id}`}
  className={cn(gridClasses)}
  onClick={handleClick}
>
  <EntityName name={entity.name} />
</EntityListItem>

// ✅ GOOD: Single line for few props
<Button variant="primary" onClick={handleClick}>Save</Button>
```

### Boolean Props

```typescript
// ✅ GOOD: Omit true value
<Button disabled />

// ❌ BAD: Explicit true (unnecessary)
<Button disabled={true} />

// ✅ GOOD: Explicit for false
<Button disabled={false} />
```

### Conditional Props

```typescript
// ✅ GOOD: Spread conditional props
<EntityList
  {...(isAdmin && { showAdminActions: true })}
  {...(error && { error })}
/>

// ✅ GOOD: Inline conditional for single prop
<Button disabled={loading || !hasChanges} />
```

## Styling with Tailwind

### Use cn() Utility

```typescript
import { cn } from '@adtribute/ui';

// ✅ GOOD: Use cn() for conditional classes
<div className={cn(
  'flex items-center gap-2',
  isActive && 'bg-primary text-white',
  disabled && 'opacity-50 pointer-events-none'
)}>
```

### Class Organization

```typescript
// ✅ GOOD: Group related classes
<div className={cn(
  // Layout
  'flex flex-col gap-4',
  // Sizing
  'w-full h-full',
  // Styling
  'bg-background border border-border rounded-lg',
  // States
  'hover:bg-accent focus:ring-2',
  // Conditional
  isActive && 'bg-primary'
)}>
```

## Async/Await

### Always Use async/await (Not Promises)

```typescript
// ✅ GOOD: async/await
const fetchData = async () => {
  const result = await resolveOrThrow(fetchAction());
  return result;
};

// ❌ BAD: Promise chains (harder to read)
const fetchData = () => {
  return fetchAction()
    .then(result => resolveOrThrow(result))
    .then(data => data);
};
```

### Error Handling

```typescript
// ✅ GOOD: Let React Query handle errors
const query = useQuery({
  queryKey: ['entity'],
  queryFn: () => resolveOrThrow(fetchEntityAction()), // Throws on error
});

// ✅ GOOD: Try/catch when needed
const saveData = async () => {
  try {
    await resolveOrThrow(saveAction(data));
    showSuccessMessage();
  } catch (error) {
    console.error('Save failed:', error);
    showErrorMessage(error.message);
  }
};
```

## Destructuring

```typescript
// ✅ GOOD: Destructure props
export function Component({ name, age, address }: ComponentProps) {
  return <div>{name}, {age}, {address}</div>;
}

// ✅ GOOD: Destructure in function body when conditional
export function Component(props: ComponentProps) {
  if (!props.user) return null;
  const { name, age } = props.user;
  return <div>{name}, {age}</div>;
}

// ✅ GOOD: Destructure hook returns
const { data, isLoading, error } = useQuery({ ... });
```

## Array Operations

```typescript
// ✅ GOOD: Use array methods
const activeItems = items.filter(item => item.active);
const itemNames = items.map(item => item.name);
const hasActive = items.some(item => item.active);

// ❌ BAD: Manual loops for simple operations
const activeItems = [];
for (let i = 0; i < items.length; i++) {
  if (items[i].active) activeItems.push(items[i]);
}
```

## Object Spread

```typescript
// ✅ GOOD: Use spread for immutable updates
const updated = { ...state, name: newName };

// ✅ GOOD: Merge nested objects
const updated = {
  ...state,
  configuration: {
    ...state.configuration,
    newField: value,
  },
};

// ❌ BAD: Mutation
state.name = newName; // Mutates state!
```

## Console Logging

```typescript
// ✅ GOOD: Keep useful logs
console.error('Server action error:', error);
console.time('fetchFields');
console.timeEnd('fetchFields');

// ❌ BAD: Debug logs in production code
console.log('user clicked button'); // Remove before commit

// ✅ GOOD: Conditional dev logging
if (process.env.NODE_ENV === 'development') {
  console.log('Debug info:', data);
}
```

## Quick Reference

```typescript
// Template for a typical component
'use client';

import { useState, useCallback } from 'react';
import { Button } from '@adtribute/ui/new';
import { cn } from '@adtribute/ui';

type MyComponentProps = {
  title: string;
  onSave: (data: FormData) => void;
};

export function MyComponent({ title, onSave }: MyComponentProps) {
  const [data, setData] = useState<FormData>({ name: '' });
  
  const handleSave = useCallback(() => {
    onSave(data);
  }, [data, onSave]);
  
  return (
    <div className={cn('flex flex-col gap-4 p-4')}>
      <h1 className="text-2xl font-bold">{title}</h1>
      <Button onClick={handleSave}>Save</Button>
    </div>
  );
}
```

**Key Principles**:
- Always use functional React and TypeScript
- Don't remove code comments
- Use `cn()` for className management
- Use `useCallback` for event handlers
- Use `async/await` over promises
- Destructure props and hook returns
- Keep imports organized

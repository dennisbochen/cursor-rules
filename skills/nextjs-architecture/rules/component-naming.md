description: Component naming conventions and separation of concerns for presentational vs connected components
globs:
  - "**/components/**"
  - "**/connected/**"
alwaysApply: false
---

# Component Naming Conventions

Strict naming conventions for React components to maintain consistency and clarity.

## File Naming

### Presentational Components
- **Location**: `/features/[feature-name]/components/`
- **File**: `[name].tsx` (kebab-case)
- **Export**: PascalCase function name

```typescript
// components/entity-list.tsx
export function EntityList({ children }: { children: React.ReactNode }) {
  return <ul>{children}</ul>;
}

// components/filter.tsx
export function FilterGroup({ children }: { children: React.ReactNode }) {
  return <div>{children}</div>;
}
```

### Connected Components
- **Location**: `/features/[feature-name]/connected/`
- **File**: `connected-[name].tsx` (kebab-case with `connected-` prefix)
- **Export**: `Connected[Name]` (PascalCase with `Connected` prefix)

```typescript
// connected/connected-feature.tsx
export function ConnectedFeature() {
  const { items, loading } = useFeature(); // Hook provides data
  return <FeatureList>{/* render */}</FeatureList>; // Presentational component
}

// connected/connected-entity-details.tsx
export function ConnectedEntityDetails({ isNew }: { isNew: boolean }) {
  const context = useEntityContext(); // Context provides data
  return <EntityDetailsLayout>{/* render */}</EntityDetailsLayout>;
}
```

## Component Separation Principles

### Presentational Components (Dumb)
**Purpose**: Pure UI rendering

**Characteristics**:
- Receive ALL data via props
- No hooks except UI-only hooks (useState for local UI state)
- No data fetching (no useQuery, no server actions)
- No business logic
- Highly reusable

**Reference**: /features/entity-list/components/entity-list.tsx

```typescript
// ✅ Good: Presentational component
export function EntityList({ children }: { children: React.ReactNode }) {
  return <ul className="flex flex-col">{children}</ul>;
}

export function EntityName({ name }: { name: string }) {
  return <span className="font-semibold">{name}</span>;
}
```

### Connected Components (Smart)
**Purpose**: Data wiring and orchestration

**Characteristics**:
- Use custom hooks or contexts to get data
- Wire data to presentational components
- Handle user interactions (callbacks)
- May use React Query, server actions
- Feature-specific (less reusable)

**Reference**: /features/entity-list/connected/connected-fields.tsx

```typescript
// ✅ Good: Connected component
export function ConnectedFields() {
  // 1. Get data from hooks/contexts
  const {
    fields,
    refinements,
    loading,
    handleToggleFieldNameSortDirection,
    queryParams,
  } = useFields();

  // 2. Render presentational components with data
  return (
    <ListPage>
      <PageHeader title="Attributes" />
      <EntityList>
        {fields?.map((field) => (
          <EntityListItem href={`/definitions/fields/${field.id}`}>
            <EntityName name={field.fieldName} />
          </EntityListItem>
        ))}
      </EntityList>
    </ListPage>
  );
}
```

## Common Component Patterns

### Layout Components
Layout components are presentational but compose other components:

```typescript
// components/layout.tsx
export function Layout({
  operatorComponent,
  filtersComponent,
  buttonsComponent,
  layoutClassName,
}: LayoutProps) {
  return (
    <div className={layoutClassName}>
      {operatorComponent}
      {filtersComponent}
      {buttonsComponent}
    </div>
  );
}
```

### Input Components
Input components may have local state but receive/send data via props:

```typescript
// components/string-input.tsx
export function StringInput({
  onChange,
  onError,
  initialValue,
}: StringInputProps) {
  const [value, setValue] = useState(initialValue || '');
  
  useEffect(() => {
    const timer = setTimeout(() => onChange?.(value), 200);
    return () => clearTimeout(timer);
  }, [value, onChange]);

  return <input value={value} onChange={(e) => setValue(e.target.value)} />;
}
```

## Naming Sub-Components

For components that are only used within a specific parent:

### Option 1: Same File (Preferred for simple sub-components)
```typescript
// components/entity-list.tsx
const BadgePill = ({ children }: { children: ReactNode }) => {
  return <div className="badge">{children}</div>;
};

export function EntityList({ children }: { children: ReactNode }) {
  return <ul>{children}</ul>;
}

export function EntityListItem({ name }: { name: string }) {
  return (
    <li>
      <BadgePill>{name}</BadgePill>
    </li>
  );
}
```

### Option 2: Separate File (For complex sub-components)
```typescript
// components/entity-list/badge-pill.tsx
export function BadgePill({ children }: { children: ReactNode }) {
  return <div className="badge">{children}</div>;
}

// components/entity-list/index.tsx
export { EntityList } from './entity-list';
export { EntityListItem } from './entity-list-item';
```

## Exporting Components

### From Feature's index.ts
Only export components that are meant to be used outside the feature:

```typescript
// features/feature-name/index.ts
export { ConnectedFeature } from './connected/connected-feature';
export { FeatureDataProvider } from './data-providers/feature-data-provider';

// DON'T export internal components:
// ❌ export { FeatureList } from './components/feature-list';
```

### From components/index.ts
Only create if you have many components to organize:

```typescript
// features/entity-list/components/index.ts
export { EntityList } from './entity-list';
export { FilterGroup } from './filter';
export { ListPage } from './list-page';
```

## Anti-Patterns

### ❌ Don't: Data Fetching in Presentational Components
```typescript
// ❌ BAD: components/fields-list.tsx
export function FieldsList() {
  const { data } = useQuery(['fields'], fetchFields); // ❌ No data fetching!
  return <ul>{data?.map(...)}</ul>;
}
```

### ❌ Don't: Business Logic in Presentational Components
```typescript
// ❌ BAD: components/entity-list.tsx
export function EntityList({ items }) {
  // ❌ No business logic!
  const filteredItems = items.filter(item => item.active);
  const sortedItems = filteredItems.sort((a, b) => a.name.localeCompare(b.name));
  return <ul>{sortedItems.map(...)}</ul>;
}
```

### ❌ Don't: Wrong Naming Convention
```typescript
// ❌ BAD: connected/fields.tsx (missing 'connected-' prefix)
export function Fields() { ... }

// ✅ GOOD: connected/connected-fields.tsx
export function ConnectedFields() { ... }
```

## Quick Reference

| Component Type | Location | File Name | Export Name |
|----------------|----------|-----------|-------------|
| Presentational | `/components/` | `item-list.tsx` | `ItemList` |
| Connected | `/connected/` | `connected-feature.tsx` | `ConnectedFeature` |
| Layout | `/components/` | `layout.tsx` | `Layout` |
| Input | `/components/` | `string-input.tsx` | `StringInput` |

**Remember**: The `Connected` prefix tells developers "this component fetches/manages data" at a glance.

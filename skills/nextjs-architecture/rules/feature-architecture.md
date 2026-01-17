description: Feature folder structure and organization standards for the Next.js projects
alwaysApply: true
---

# Feature Architecture

All features in `apps/web/src/features/` must follow this consistent folder structure.

## Standard Feature Structure

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

## Folder Guidelines

### `/components/`
- Contains **presentational** (dumb) components
- Should be pure UI components that receive data via props
- No direct data fetching or business logic
- Example: `list-page.tsx`, `filter.tsx`, `entity-list.tsx`

### `/connected/`
- Contains **connected** (smart) components
- Wire data from hooks/contexts to presentational components
- File naming: `connected-[name].tsx` exports `Connected[Name]`
- Example: `connected-fields.tsx` exports `ConnectedFields`

### `/contexts/`
- React Context providers for feature-level state
- Includes context definition, provider, and custom hook
- Example: `event-context.tsx` exports `EventContext`, `EventContextProvider`, `useEventContext`

### `/data-providers/`
- Server Components that fetch initial data for SSR
- Wrap client components with context providers
- Export type for initial data shape
- Example: `event-details.tsx` exports `EventDetailsDataProvider` and `InitialEventData` type

### `/hooks/`
- Custom React hooks for state management and side effects
- Complex hooks should follow the useReducer pattern (see use-reducer-pattern rule)
- Simple hooks can be single files
- Example: `use-fields.tsx`, `use-query-params.tsx`

### `/schemas/`
- Zod schemas for validation and type generation
- Follow dual schema pattern (see zod-schemas rule)
- One schema per entity/concept
- Example: `event.ts`, `field.ts`

### `/server-actions/`
- Next.js server actions with 'use server' directive
- Use `makeAuthorizedServerAction` helper
- File per entity or logical grouping
- Example: `fetch-fields-action.ts`, `event.ts`

### `/utils/`
- Pure utility functions
- No React dependencies
- Example: `defaults.ts`, `filter-helpers.ts`, `url-helpers.ts`

### `/constants/`
- Constant values, enums, and static configurations
- TypeScript types and interfaces
- Example: `filter-operators.ts`

### `index.ts`
- Public API for the feature
- Re-export only what should be consumed outside the feature
- Do not export internal implementation details

## Example: Reference Implementation

See /features/entity-list for a complete example

```typescript
// index.ts - Clean public API
export { ConnectedFeature } from './connected/connected-feature';
export { FeatureDataProvider } from './data-providers/feature-data-provider';
export { fetchFeatureAction } from './server-actions/fetch-feature-action';
export { type QueryParams } from './utils/make-data-fetcher';
```

## Creating a New Feature

When creating a new feature:

1. Create the feature folder: `features/[feature-name]/`
2. Add `index.ts` first - think about your public API
3. Create folders as needed (not all folders are required)
4. Start with `schemas/` to define your data types
5. Build `components/` for UI
6. Add `connected/` to wire data to UI
7. Add other folders (contexts, server-actions, etc.) as complexity requires

## Anti-Patterns to Avoid

- ❌ Don't put business logic in presentational components
- ❌ Don't import from other feature's internals (only use their `index.ts` exports)
- ❌ Don't create circular dependencies between features
- ❌ Don't mix server and client code in the same file
- ❌ Don't skip the `index.ts` - always provide a clean public API

## Cross-Feature Dependencies

- Import from other features via their `index.ts` only
- Keep features as independent as possible
- Share common utilities via `/lib` or `/components` at app level
- Consider extracting shared logic to packages if used across multiple apps

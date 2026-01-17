---
name: nextjs-architecture
description: Next.js App Router architecture patterns including feature structure, component organization, server actions, contexts, data providers, and state management with useReducer. Generic patterns applicable to any Next.js project.
license: MIT
metadata:
  version: "1.0.0"
applies_to:
  - "**/app/**/*.tsx"
  - "**/app/**/*.ts"
  - "**/features/**/*.tsx"
  - "**/features/**/*.ts"
  - "package.json contains next"
  - "next.config.js or next.config.mjs present"
triggers:
  - "feature structure"
  - "feature architecture"
  - "server actions"
  - "context provider"
  - "data provider"
  - "component naming"
  - "useReducer pattern"
  - "Zod schemas"
  - "data fetching"
  - "Next.js App Router"
  - "server component"
  - "client component"
skip_if:
  - "Next.js Pages Router (pages/ directory)"
  - "not using Next.js"
  - "vue in package.json"
  - "angular in package.json"
  - "plain React without Next.js"
---

# Next.js Architecture Patterns

Comprehensive architectural patterns for Next.js App Router projects, covering feature organization, component structure, server actions, contexts, data providers, and state management.

## When to Apply

Reference these patterns when:
- Structuring features and folders in Next.js App Router
- Creating server actions with authorization
- Building React context providers for state management
- Setting up data providers for SSR
- Implementing the useReducer pattern for complex state
- Defining Zod schemas for validation
- Organizing components (presentational vs connected)
- Implementing data fetching patterns

## Pattern Categories

1. **Feature Architecture** - Folder structure and organization standards
2. **Component Naming** - Naming conventions for presentational vs connected components
3. **Context Pattern** - React Context for entity state management
4. **Data Fetching** - React Query, makeDataFetcher, and resolveOrThrow patterns
5. **Data Providers** - Server-side data fetching and context initialization
6. **Server Actions** - makeAuthorizedServerAction implementation
7. **useReducer Pattern** - Complex state management with reducers
8. **Zod Schemas** - Dual schema pattern for type definitions and validation
9. **Code Style** - TypeScript, React, and general code conventions

## Full Patterns Document

For complete patterns with examples: `AGENTS.md`

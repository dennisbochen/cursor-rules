description: Dual Zod schema pattern for type definitions and validation
globs:
  - "**/schemas/**"
  - "**/schema/**"
alwaysApply: false
---

# Zod Schema Pattern

Use the dual schema approach for all features: one permissive schema for UI state, one strict schema for validation.

## Why Dual Schemas?

- **Type Schema** (permissive): Allows incomplete/partial data during editing
- **Validation Schema** (strict): Enforces complete/valid data before saving

This pattern enables better UX by allowing users to work with incomplete forms without seeing validation errors until save.

## Pattern Overview

Reference: /features/entity-details/schemas/event.ts

```typescript
import { z } from 'zod';

// 1. TYPE SCHEMA - Permissive for UI state
export const entityConfigTypeSchema = z.object({
  name: z.string().optional(),
  status: z.string().optional(),
  category: z.string().optional(),
  settings: settingsTypeSchema.optional(),
  metadata: metadataTypeSchema.optional(),
});

// 2. VALIDATION SCHEMA - Strict for saving
export const entityConfigValidationSchema = z.object({
  name: z
    .string({ required_error: 'Please enter a name' })
    .min(1, 'Name is required'),
  status: z
    .string({ required_error: 'Please select a status' })
    .min(1, 'Status is required'),
  category: z
    .string({ required_error: 'Please select a category' })
    .min(1, 'Category is required'),
  settings: settingsValidationSchema,
  metadata: metadataValidationSchema,
});

// 3. EXPORT TYPES
export type EntityConfig = z.infer<typeof entityConfigTypeSchema>;
export type ValidatedEntityData = z.infer<typeof entityConfigValidationSchema>;
```

## Schema Naming Conventions

### For Nested Schemas
```typescript
// Type schema (permissive)
const deduplicationTypeSchema = z.discriminatedUnion('type', [
  z.object({
    type: z.literal('every'),
  }),
  z.object({
    type: z.literal('once_per'),
    once_per: z.array(z.string()).optional(), // Optional during editing
  }),
]);

// Validation schema (strict)
const deduplicationValidationSchema = z.discriminatedUnion('type', [
  z.object({
    type: z.literal('every'),
  }),
  z.object({
    type: z.literal('once_per'),
    once_per: z.preprocess(
      (val) => val ?? [], 
      z.array(z.string())
    ), // Required, with default
  }),
]);
```

### For Top-Level Entity Schemas
```typescript
// Type schema
export const entityStateSchema = z.object({
  id: z.string().optional(),
  key: z.string().optional(),
  name: z.string().optional(),
  description: z.string().optional(),
  configuration: entityConfigTypeSchema.optional(),
});

// Validation schema
export const entityValidationSchema = z.object({
  name: z
    .string({ required_error: 'Please enter a name' })
    .min(1, 'Please enter a name'),
  description: z.string(),
  key: z.string().optional(),
  configuration: entityConfigValidationSchema,
});

// Type exports
export type EntityState = z.infer<typeof entityStateSchema>;
export type ValidatedEntityData = z.infer<typeof entityValidationSchema>;
```

## Schema File Organization

### Single Schema File (Simple Features)
```
/schemas/
└── event.ts  # Contains both type and validation schemas
```

### Multiple Schema Files (Complex Features)
```
/hooks/use-complex-filter/
└── schema/
    ├── schema.ts           # Type schemas
    ├── validationSchema.ts # Validation schemas
    └── shared.ts           # Shared enums/primitives
```

Reference: /features/complex-filter/hooks/use-complex-filter/schema/

## Permissive Type Schema Rules

Type schemas should be **permissive** to allow incomplete state:

```typescript
// ✅ GOOD: Permissive type schema
export const ruleSchema = z.object({
  id: z.string().default(() => generateId()),
  left: z.object({
    type: z.literal('attribute'),
    attribute: z.string(),
  }).nullish(),  // nullish = null | undefined | value
  operator: ruleOperatorSchema.nullish(),
  right: z.discriminatedUnion('type', [...]).nullish(),
});
```

**Use `.nullish()` or `.optional()` for fields that**:
- Can be empty during editing
- Are set incrementally by the user
- Have default values

## Strict Validation Schema Rules

Validation schemas should **enforce completeness**:

```typescript
// ✅ GOOD: Strict validation schema
export const ruleSchema = z.object({
  left: z.object({
    type: z.literal('attribute'),
    attribute: z.string().min(1, 'An attribute is required for a rule'),
  }),
  operator: ruleOperatorSchema,
  right: z
    .discriminatedUnion('type', [
      attributeVariableSchema,
      constantVariableSchema,
    ])
    .nullish(),
}).superRefine((data, ctx) => {
  // Custom validation with business rules
  if (!['Is defined', 'Is not defined'].includes(data.operator) && !data.right) {
    ctx.addIssue({
      code: z.ZodIssueCode.custom,
      message: 'A value is required',
    });
  }
});
```

**Validation schemas should**:
- Require all necessary fields
- Include helpful error messages
- Use `.superRefine()` for complex validation
- Enforce business rules

## Using Schemas in Components

### In Contexts (State Management)
Reference: /features/entity-details/contexts/event-context.tsx

```typescript
import { entityStateSchema, entityValidationSchema } from '../schemas/entity';

export const EntityContextProvider = ({ children, initialData }) => {
  // currentData follows type schema shape
  const currentData: EntityState = useMemo(() => ({
    id: serverData?.id,
    name: draftChanges.name ?? serverData?.name ?? 'New Entity',
    configuration: {
      ...serverData?.configuration,
      ...draftChanges.configuration,
    },
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

### In Server Actions
```typescript
import { entityValidationSchema } from '../schemas/entity';

export const createEntityAction = makeAuthorizedServerAction({
  input: z.object({
    data: entityValidationSchema, // Use validation schema for server
  }),
  handler: async ({ data }, { organization }) => {
    // data is guaranteed to be valid
    const entity = await prisma.entity.create({ data: { ...data } });
    return { entity };
  },
});
```

## Advanced Patterns

### Discriminated Unions
```typescript
// Type schema
export const conditionSchema = z.discriminatedUnion('type', [
  z.object({
    id: z.string().default(() => generateId()),
    type: z.literal('rule'),
    rule: ruleSchema,
  }),
  z.object({
    id: z.string().default(() => generateId()),
    type: z.literal('group'),
    group: z.object({
      operator: logicalOperatorSchema,
      rules: z.array(ruleSchema),
    }),
  }),
]);

// Validation schema
export const conditionSchema = z.discriminatedUnion('type', [
  z.object({
    type: z.literal('rule'),
    rule: ruleSchema, // Strict rule schema
  }),
  z.object({
    type: z.literal('group'),
    group: z.object({
      operator: logicalOperatorSchema,
      rules: z.array(ruleSchema).min(1, 'At least one rule is required'),
    }),
  }),
]);
```

### Shared Primitives
```typescript
// schema/shared.ts
export const logicalOperatorSchema = z.enum(['AND', 'OR']);
export const ruleOperatorSchema = z.enum([
  'Equals',
  'Does not equal',
  'Is defined',
  'Is not defined',
  // ... more operators
]);

// Use in both type and validation schemas
import { logicalOperatorSchema } from './shared';
```

### Preprocessing
```typescript
export const deduplicationValidationSchema = z.object({
  type: z.literal('once_per'),
  once_per: z.preprocess(
    (val) => val ?? [], // Convert null/undefined to empty array
    z.array(z.string())
  ),
});
```

## Type Exports

Always export types from schemas:

```typescript
// Export type from type schema
export type EventState = z.infer<typeof eventStateSchema>;
export type EventConfig = z.infer<typeof eventConfigTypeSchema>;

// Export type from validation schema
export type ValidatedEventData = z.infer<typeof eventValidationSchema>;

// Export component types
export type Rule = z.infer<typeof ruleSchema>;
export type Condition = z.infer<typeof conditionSchema>;
```

## Error Formatting

Use the `formatZodErrors` utility for user-friendly error messages:

Reference: @apps/web/src/lib/utils/zod/format-errors.ts

```typescript
import { formatZodErrors } from '@/lib/utils/zod/format-errors';

const result = validationSchema.safeParse(data);
if (!result.success) {
  const errors = formatZodErrors(result.error);
  // errors is string[] of formatted messages
  setValidationErrors(errors);
}
```

## Anti-Patterns

### ❌ Don't: Use Validation Schema for State
```typescript
// ❌ BAD: Using validation schema for state type
const [state, setState] = useState<ValidatedEntityData>({
  name: '', // ❌ This will fail validation immediately!
  configuration: { ... },
});
```

### ❌ Don't: Use Type Schema for Server Actions
```typescript
// ❌ BAD: Using type schema for server validation
export const createEntityAction = makeAuthorizedServerAction({
  input: z.object({
    data: entityStateSchema, // ❌ Too permissive!
  }),
  handler: async ({ data }) => {
    // data might be incomplete!
  },
});
```

### ❌ Don't: Single Schema for Everything
```typescript
// ❌ BAD: One schema trying to do both
export const entitySchema = z.object({
  name: z.string().optional(), // Confusing - is this required or not?
  configuration: z.object({ ... }),
});
```

## Quick Reference

| Use Case | Schema Type | Characteristics |
|----------|-------------|-----------------|
| UI State | Type Schema | `.optional()`, `.nullish()`, permissive |
| Form Validation | Validation Schema | `.min()`, error messages, strict |
| Server Actions | Validation Schema | Complete data required |
| API Responses | Type Schema | May have nullable fields |

**Remember**: Type schema = "what can exist in memory", Validation schema = "what is ready to save"

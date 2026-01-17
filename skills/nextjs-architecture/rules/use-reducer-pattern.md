description: useReducer hook pattern implementation following the established best practices
globs:
  - "**/hooks/use-*/**"
alwaysApply: false
---

# useReducer Pattern

Follow the established useReducer pattern from @reducer-hooks-best-practices.md for all complex state management hooks.

## File Structure (Required)

```
/hooks/use-[feature-name]/
├── index.ts           # Main hook implementation
├── actions.ts         # Action creators and handlers
├── reducer.ts         # Reducer function (switch statement only)
└── schema/            # Zod schemas (or schema.ts)
    ├── schema.ts      # Type schema (permissive)
    └── validationSchema.ts  # Validation schema (strict)
```

## Implementation Checklist

- [ ] Dual schema approach (type + validation)
- [ ] Discriminated union types for actions
- [ ] Separated action handler functions (not inline)
- [ ] Immutable state updates
- [ ] Hook returns state + action creators
- [ ] Action creators use `useCallback`
- [ ] Validation with `useEffect`

## 1. Schema Definition

Reference: /features/complex-filter/hooks/use-complex-filter/schema/schema.ts

```typescript
// schema/schema.ts - Permissive type schema
import { z } from 'zod';

export const ruleSchema = z.object({
  id: z.string().default(() => generateId()),
  left: z.object({
    type: z.literal('attribute'),
    attribute: z.string(),
  }).nullish(),  // nullish allows incomplete state
  operator: ruleOperatorSchema.nullish(),
  right: z.discriminatedUnion('type', [...]).nullish(),
});

export const filterSchema = z.object({
  operator: logicalOperatorSchema,
  conditions: z.array(conditionSchema),
});

export type FilterState = z.infer<typeof filterSchema>;
export type Rule = z.infer<typeof ruleSchema>;
```

```typescript
// schema/validationSchema.ts - Strict validation schema
import { z } from 'zod';

export const ruleSchema = z.object({
  left: z.object({
    type: z.literal('attribute'),
    attribute: z.string().min(1, 'Required'),  // Strict validation
  }),
  operator: ruleOperatorSchema,
  right: z.discriminatedUnion('type', [...]),
}).superRefine((data, ctx) => {
  // Custom validation logic
  if (!data.right && !['Is defined', 'Is not defined'].includes(data.operator)) {
    ctx.addIssue({
      code: z.ZodIssueCode.custom,
      message: 'A value is required',
    });
  }
});

export const filterSchema = z.object({
  operator: logicalOperatorSchema,
  conditions: z.array(conditionSchema).min(1, 'At least one condition required'),
});
```

## 2. Actions Definition

Reference: /features/complex-filter/hooks/use-complex-filter/actions.ts

**Key principles**:
- One action type per action
- Discriminated union of all actions
- Separate handler function for each action
- Handler functions are pure (no side effects)
- Always return new state (immutable)

```typescript
// actions.ts
import { FilterState, Rule } from './schema/schema';

// 1. Define individual action types
export type AddSingleConditionAction = {
  type: 'add-single-condition';
  condition: Condition;
};

export type UpdateConditionAction = {
  type: 'update-condition';
  payload: { conditionId: string; rule: Rule };
};

export type RemoveConditionAction = {
  type: 'remove-condition';
  payload: { conditionId: string };
};

// 2. Create discriminated union
export type Actions = 
  | AddSingleConditionAction 
  | UpdateConditionAction 
  | RemoveConditionAction;

// 3. Separate handler functions
export function addSingleCondition(
  state: FilterState, 
  action: AddSingleConditionAction
): FilterState {
  return { 
    ...state, 
    conditions: [...state.conditions, action.condition] 
  };
}

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

export function removeCondition(
  state: FilterState, 
  action: RemoveConditionAction
): FilterState {
  return {
    ...state,
    conditions: state.conditions.filter(c => c.id !== action.payload.conditionId),
  };
}
```

**Naming conventions**:
- Action types: kebab-case (`'add-single-condition'`)
- Action interfaces: PascalCase + "Action" (`AddSingleConditionAction`)
- Handler functions: camelCase (`addSingleCondition`)
- Always use `payload` property for action data

## 3. Reducer Implementation

Reference: /features/complex-filter/hooks/use-complex-filter/reducer.ts

**Key principle**: Reducer contains ONLY switch statement, no logic.

```typescript
// reducer.ts
import { FilterState } from './schema/schema';
import {
  Actions,
  addSingleCondition,
  updateCondition,
  removeCondition,
} from './actions';

export function reducer(state: FilterState, action: Actions): FilterState {
  switch (action.type) {
    case 'add-single-condition':
      return addSingleCondition(state, action);
    case 'update-condition':
      return updateCondition(state, action);
    case 'remove-condition':
      return removeCondition(state, action);
    default:
      return state;
  }
}
```

**Rules**:
- Import all action handlers
- One case per action type
- Always include default case
- No inline logic

## 4. Main Hook Implementation

Reference: /features/complex-filter/hooks/use-complex-filter/index.ts

```typescript
// index.ts
'use client';

import { useCallback, useEffect, useReducer, useRef } from 'react';
import { reducer } from './reducer';
import { FilterState } from './schema/schema';

type UseFilterProps = {
  initialState?: FilterState;
  onChange?: (state: FilterState) => void;
};

export function useFilter({ initialState, onChange }: UseFilterProps) {
  // 1. Define default state
  const defaultState: FilterState = {
    operator: 'AND',
    conditions: [],
  };

  // 2. Process initial state (add IDs, parse, etc.)
  function processInitialState(state?: FilterState): FilterState | null {
    if (!state) return null;
    return parseState(state); // Add IDs, validate structure
  }

  // 3. Initialize reducer
  const [state, dispatch] = useReducer(
    reducer,
    processInitialState(initialState) || defaultState
  );

  // 4. Use ref for onChange to avoid re-renders
  const onChangeRef = useRef(onChange);
  onChangeRef.current = onChange;

  // 5. Call onChange on state changes
  useEffect(() => {
    onChangeRef.current?.(state);
  }, [state]);

  // 6. Action creators with useCallback
  const addSingleCondition = useCallback(() => {
    dispatch({ type: 'add-single-condition', condition: makeDefaultCondition() });
  }, []);

  const updateCondition = useCallback(
    (payload: { conditionId: string; rule: Rule }) => {
      dispatch({ type: 'update-condition', payload });
    }, 
    []
  );

  const removeCondition = useCallback(
    (payload: { conditionId: string }) => {
      dispatch({ type: 'remove-condition', payload });
    }, 
    []
  );

  // 7. Return state + action creators
  return {
    state,
    addSingleCondition,
    updateCondition,
    removeCondition,
  };
}
```

**Key patterns**:
- `'use client'` directive at top
- Default state constant
- Initial state processing function
- `useRef` for onChange to avoid re-renders
- `useCallback` for all action creators
- Return state + actions (no validation here)

## 5. Validation Pattern (Optional)

If validation is needed in the hook (not handled by parent):

```typescript
export function useFilter({ initialState, onChange }: UseFilterProps) {
  const [state, dispatch] = useReducer(reducer, defaultState);
  const [isStateValid, setIsStateValid] = useState<boolean | undefined>(undefined);
  const [validationErrors, setValidationErrors] = useState<string[] | undefined>(undefined);

  // Validate on state changes
  useEffect(() => {
    const result = validationSchema.safeParse(state);
    if (result.error) {
      setValidationErrors(formatZodErrors(result.error));
    } else {
      setValidationErrors(undefined);
    }
    setIsStateValid(result.success);
  }, [state]);

  return {
    state,
    isStateValid,
    validationErrors,
    // ... action creators
  };
}
```

## Common Patterns

### Nested State Updates
```typescript
export function updateGroupCondition(
  state: FilterState,
  action: UpdateGroupConditionAction
): FilterState {
  return {
    ...state,
    conditions: state.conditions.map((condition) => {
      if (condition.type !== 'group') return condition;
      if (condition.id !== action.payload.conditionId) return condition;
      return {
        ...condition,
        group: {
          ...condition.group,
          rules: condition.group.rules?.map((r) => {
            if (r.id !== action.payload.rule.id) return r;
            return { ...r, ...action.payload.rule };
          }),
        },
      };
    }),
  };
}
```

### Adding Items to Arrays
```typescript
export function addRuleToGroup(
  state: FilterState,
  action: AddRuleToGroupAction
): FilterState {
  return {
    ...state,
    conditions: state.conditions.map((condition) => {
      if (condition.id !== action.payload.groupId) return condition;
      return {
        ...condition,
        group: {
          ...condition.group,
          rules: [...(condition.group.rules || []), action.payload.rule],
        },
      };
    }),
  };
}
```

## Anti-Patterns

### ❌ Don't: Logic in Reducer
```typescript
// ❌ BAD
export function reducer(state: FilterState, action: Actions): FilterState {
  switch (action.type) {
    case 'add-condition':
      // ❌ No logic in reducer!
      return { ...state, conditions: [...state.conditions, action.condition] };
    default:
      return state;
  }
}
```

### ❌ Don't: Mutate State
```typescript
// ❌ BAD
export function addCondition(state: FilterState, action: AddConditionAction): FilterState {
  state.conditions.push(action.condition); // ❌ Mutation!
  return state;
}

// ✅ GOOD
export function addCondition(state: FilterState, action: AddConditionAction): FilterState {
  return { ...state, conditions: [...state.conditions, action.condition] };
}
```

### ❌ Don't: Missing useCallback
```typescript
// ❌ BAD
const addCondition = () => {
  dispatch({ type: 'add-condition', ... });
};

// ✅ GOOD
const addCondition = useCallback(() => {
  dispatch({ type: 'add-condition', ... });
}, []);
```

## Quick Reference

Complete reference implementation: /features/complex-filter/hooks/use-complex-filter/

Full best practices document: @reducer-hooks-best-practices.md

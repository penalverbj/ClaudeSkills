---
name: typescript-best-practices
description: >
  Write idiomatic, type-safe TypeScript following official handbook best practices.
  Use this skill whenever writing, reviewing, or refactoring any TypeScript code —
  including functions, interfaces, classes, generics, modules, enums, error handling,
  async code, or tsconfig setup. Trigger even when the user doesn't explicitly ask
  for "best practices" — any .ts or .tsx file, any TypeScript question, or any
  JavaScript code that should be written in TypeScript should use this skill.
  Apply strict types, avoid `any`, prefer `unknown` for unsafe values, use
  discriminated unions over optional fields, leverage utility types, and always
  produce code that compiles cleanly under `"strict": true`.
---

# TypeScript Best Practices Skill

Grounded in the official [TypeScript Handbook](https://www.typescriptlang.org/docs/handbook/intro.html),
the [Declaration Files Do's and Don'ts](https://www.typescriptlang.org/docs/handbook/declaration-files/do-s-and-don-ts.html),
and the [TSConfig Reference](https://www.typescriptlang.org/tsconfig/).

**Non-negotiable baseline:** All code must compile cleanly under `"strict": true`.

---

## Core Principles

1. **Make illegal states unrepresentable** — use the type system to prevent bad data, not runtime checks alone
2. **Prefer inference over annotation** — let TypeScript infer where it can; annotate where it adds clarity
3. **Never use `any`** — use `unknown` for genuinely unknown values and narrow before use
4. **Fail loudly at compile time, not silently at runtime**
5. **Types are documentation** — a well-typed function signature is its own spec

---

## The Essential Checklist

Apply these on every TypeScript file written or reviewed:

### Strict Mode & Config
- [ ] `"strict": true` in `tsconfig.json` — non-negotiable
- [ ] `"noUncheckedIndexedAccess": true` — array/object index access returns `T | undefined`
- [ ] `"noImplicitOverride": true` — class overrides must use `override` keyword
- [ ] `"noFallthroughCasesInSwitch": true` — explicit `break`/`return`/`throw` per case
- [ ] `"exactOptionalPropertyTypes": true` — optional props can't be set to `undefined` unless typed that way

### Types & Declarations
- [ ] No `any` — replace with `unknown`, a union, or a generic
- [ ] No type assertions (`as X`) unless unavoidable; never `as any`
- [ ] No non-null assertions (`!`) unless the value is provably non-null at that point
- [ ] Prefer `interface` for object shapes (extendable, better error messages)
- [ ] Use `type` for unions, intersections, mapped types, conditional types, utility types
- [ ] Use `readonly` on properties that shouldn't mutate; `ReadonlyArray<T>` for immutable arrays
- [ ] Use `as const` to preserve literal types in arrays and objects
- [ ] Annotate function return types explicitly when the inference is non-obvious

### Narrowing & Control Flow
- [ ] Use `typeof`, `instanceof`, `in`, and discriminated union checks — not type assertions
- [ ] Add an exhaustiveness check (`satisfies never`) in switch/if chains over unions
- [ ] Handle `null` and `undefined` explicitly — never assume non-null
- [ ] Use optional chaining (`?.`) and nullish coalescing (`??`) over truthiness checks

### Functions
- [ ] Prefer function overloads over a single signature with `any` parameters
- [ ] Use rest parameters (`...args: T[]`) instead of `arguments`
- [ ] Avoid overloads that differ only in trailing optional params — use a single optional param instead
- [ ] Prefer `(x: T) => U` callback types over `Function`
- [ ] Use generic constraints (`extends`) to restrict type params meaningfully

### Generics
- [ ] Use generics when a function/class is genuinely reusable across types
- [ ] Don't use generics where a concrete union type is clearer
- [ ] Name type params: `T` for simple cases; descriptive names (`TItem`, `TKey`) for complex ones
- [ ] Use `extends` to constrain — `<T extends { id: string }>` not `<T>` + runtime check
- [ ] Prefer built-in utility types (`Partial`, `Required`, `Pick`, `Omit`, `Record`, `ReturnType`, `Awaited`) over re-implementing them

### Enums
- [ ] Prefer `const enum` for performance when values don't need to be iterable at runtime
- [ ] Prefer `as const` string union over numeric enums for readability and JSON safety:
  ```ts
  // Prefer this
  const Direction = { Up: 'up', Down: 'down' } as const;
  type Direction = typeof Direction[keyof typeof Direction];
  // Over numeric enum
  enum Direction { Up, Down }
  ```
- [ ] Never mix string and numeric values in the same enum

### Modules & Imports
- [ ] Use `import type` for type-only imports (`import type { Foo } from './foo'`)
- [ ] Prefer named exports over default exports for better refactoring tooling
- [ ] One concept per file; file name in `kebab-case` matching the primary export
- [ ] Never use triple-slash references (`/// <reference>`) in module code — use `import`

### Error Handling
- [ ] `catch` clauses receive `unknown` under `"useUnknownInCatchVariables": true` — narrow before use
- [ ] Never swallow errors silently (`catch (e) {}`)
- [ ] Use discriminated union result types (`{ ok: true; value: T } | { ok: false; error: E }`) for expected failures
- [ ] Reserve `throw` for truly exceptional, unrecoverable states

---

## Patterns: Do This, Not That

### Replacing `any`
```ts
// ❌ Loses all type safety
function process(data: any) {
  return data.map((item: any) => item.name);
}

// ✅ Generic — preserves type relationship
function process<T extends { name: string }>(items: T[]): string[] {
  return items.map(item => item.name);
}

// ✅ unknown — forces caller to narrow before use
function parseJson(raw: string): unknown {
  return JSON.parse(raw);
}
```

### Discriminated Unions over Optional Fields
```ts
// ❌ Optional fields make impossible states possible
interface Response {
  status: 'loading' | 'success' | 'error';
  data?: User;
  error?: string;
}

// ✅ Each state carries only the fields that exist in that state
type Response =
  | { status: 'loading' }
  | { status: 'success'; data: User }
  | { status: 'error'; error: string };
```

### Exhaustiveness Checking
```ts
type Shape = { kind: 'circle'; radius: number } | { kind: 'square'; side: number };

function area(shape: Shape): number {
  switch (shape.kind) {
    case 'circle': return Math.PI * shape.radius ** 2;
    case 'square': return shape.side ** 2;
    default:
      // ✅ Compile error if a new Shape variant is added without handling it
      shape satisfies never;
      throw new Error(`Unhandled shape: ${(shape as Shape).kind}`);
  }
}
```

### `as const` for Literal Types
```ts
// ❌ Widened to string[]
const roles = ['admin', 'editor', 'viewer'];

// ✅ Narrow literal tuple; derive union type from it
const ROLES = ['admin', 'editor', 'viewer'] as const;
type Role = typeof ROLES[number]; // 'admin' | 'editor' | 'viewer'
```

### Narrowing over Assertions
```ts
// ❌ Type assertion bypasses the type system
function getUser(id: string) {
  const result = cache.get(id) as User;
  return result.name; // crashes if undefined
}

// ✅ Narrow explicitly
function getUser(id: string): User | undefined {
  const result = cache.get(id);
  if (!result) return undefined;
  return result;
}
```

### Readonly Immutability
```ts
// ✅ Communicate intent; prevents accidental mutation
interface Config {
  readonly apiUrl: string;
  readonly timeout: number;
}

function processItems(items: ReadonlyArray<string>): void {
  // items.push('x'); // ❌ compile error — good!
}
```

### Generic Fetch Pattern
```ts
// ✅ Type-safe API call
async function apiFetch<T>(url: string): Promise<T> {
  const res = await fetch(url);
  if (!res.ok) throw new Error(`HTTP ${res.status}`);
  return res.json() as Promise<T>;
}

const user = await apiFetch<User>('/api/user/1');
```

### Result Type for Expected Failures
```ts
type Result<T, E = string> =
  | { ok: true; value: T }
  | { ok: false; error: E };

function divide(a: number, b: number): Result<number> {
  if (b === 0) return { ok: false, error: 'Division by zero' };
  return { ok: true, value: a / b };
}

const result = divide(10, 0);
if (result.ok) {
  console.log(result.value);
} else {
  console.error(result.error);
}
```

---

## Utility Types — Use These, Don't Re-implement

| Utility | What it does |
|---|---|
| `Partial<T>` | All properties optional |
| `Required<T>` | All properties required |
| `Readonly<T>` | All properties readonly |
| `Pick<T, K>` | Keep only keys K from T |
| `Omit<T, K>` | Remove keys K from T |
| `Record<K, V>` | Object with keys K and values V |
| `Exclude<T, U>` | Remove U from union T |
| `Extract<T, U>` | Keep only U from union T |
| `NonNullable<T>` | Remove null and undefined |
| `ReturnType<F>` | Return type of function F |
| `Parameters<F>` | Parameter tuple of function F |
| `Awaited<T>` | Unwrap Promise type |
| `InstanceType<C>` | Instance type of constructor C |

---

## tsconfig.json Templates

### App (with bundler, no tsc emit)
```json
{
  "compilerOptions": {
    "target": "es2022",
    "lib": ["es2022", "dom", "dom.iterable"],
    "module": "preserve",
    "moduleDetection": "force",
    "noEmit": true,
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "noFallthroughCasesInSwitch": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "isolatedModules": true,
    "verbatimModuleSyntax": true,
    "resolveJsonModule": true
  }
}
```

### Node.js / Server (tsc emits JS)
```json
{
  "compilerOptions": {
    "target": "es2022",
    "lib": ["es2022"],
    "module": "NodeNext",
    "outDir": "dist",
    "rootDir": "src",
    "sourceMap": true,
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "noFallthroughCasesInSwitch": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "isolatedModules": true,
    "verbatimModuleSyntax": true,
    "resolveJsonModule": true
  }
}
```

### Library
```json
{
  "compilerOptions": {
    "target": "es2022",
    "module": "NodeNext",
    "outDir": "dist",
    "rootDir": "src",
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "composite": true,
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "verbatimModuleSyntax": true
  }
}
```

---

## What to Append to Every TypeScript Response

After writing any TypeScript, add a **Type Safety Notes** section when relevant:

```
## Type Safety Notes

✅ Applied: [key type decisions, e.g. "discriminated union for API state", "generic constraint on fetch"]
⚠️ Watch out for: [e.g. "index access returns T | undefined under noUncheckedIndexedAccess"]
💡 Consider: [optional improvement, e.g. "branded type for IDs", "Result<T> instead of throw"]
```

---

## Reference Files

- `references/type-patterns.md` — Advanced patterns: branded types, template literal types,
  mapped types, conditional types, module augmentation, declaration merging. Read when working
  on complex type utilities, library typings, or advanced generic patterns.
- `references/dos-and-donts.md` — Official TypeScript Declaration File Do's and Don'ts,
  plus common mistakes catalog. Read when writing `.d.ts` files or reviewing type declarations.

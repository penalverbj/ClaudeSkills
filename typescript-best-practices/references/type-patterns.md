# TypeScript Advanced Type Patterns

Reference for the `typescript-best-practices` skill. Read this when working on
complex type utilities, library code, or advanced generic patterns.

---

## Branded / Nominal Types

TypeScript's structural type system means `type UserId = string` and `string` are
interchangeable. Branded types prevent mixing up semantically different string/number values.

```ts
// Define a brand
type Brand<T, B> = T & { readonly __brand: B };

type UserId   = Brand<string, 'UserId'>;
type OrderId  = Brand<string, 'OrderId'>;

// Constructor functions that "cast" safely
const toUserId  = (s: string): UserId  => s as UserId;
const toOrderId = (s: string): OrderId => s as OrderId;

function getUser(id: UserId) { /* ... */ }

const uid  = toUserId('u-123');
const oid  = toOrderId('o-456');

getUser(uid);  // ✅
getUser(oid);  // ❌ Argument of type 'OrderId' is not assignable to 'UserId'
```

Use branded types for: IDs, validated strings (email, URL), physical units (meters vs feet).

---

## Template Literal Types

Compose string types from other string types.

```ts
type HttpMethod = 'GET' | 'POST' | 'PUT' | 'DELETE';
type ApiPath    = '/users' | '/orders' | '/products';

// Cartesian product
type ApiEndpoint = `${HttpMethod} ${ApiPath}`;
// "GET /users" | "GET /orders" | "POST /users" | ...

// CSS utility pattern
type Side      = 'top' | 'right' | 'bottom' | 'left';
type CssProp   = `margin-${Side}` | `padding-${Side}`;

// Event map pattern
type EventName<T extends string> = `on${Capitalize<T>}`;
type ButtonEvent = EventName<'click' | 'focus'>; // 'onClick' | 'onFocus'
```

---

## Mapped Types

Transform every property in a type.

```ts
// Make all properties into getters
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};

interface User { name: string; age: number; }
type UserGetters = Getters<User>;
// { getName: () => string; getAge: () => number }

// Filter properties by value type
type PickByValue<T, V> = {
  [K in keyof T as T[K] extends V ? K : never]: T[K];
};

type OnlyStrings = PickByValue<{ id: number; name: string; tag: string }, string>;
// { name: string; tag: string }
```

---

## Conditional Types

Types that branch based on a type relationship.

```ts
// Unwrap a Promise (equivalent to built-in Awaited<T>)
type Unwrap<T> = T extends Promise<infer U> ? Unwrap<U> : T;

// Extract the element type of an array
type ElementOf<T> = T extends (infer E)[] ? E : never;
type N = ElementOf<number[]>; // number

// Distribute over unions
type IsString<T> = T extends string ? true : false;
type R = IsString<string | number>; // true | false (distributed)

// Non-distributive version (wrap in tuple)
type IsExactlyString<T> = [T] extends [string] ? true : false;
type R2 = IsExactlyString<string | number>; // false
```

---

## Recursive Types

Types that reference themselves — useful for tree structures and deep utilities.

```ts
// JSON-safe value
type JsonValue =
  | string
  | number
  | boolean
  | null
  | JsonValue[]
  | { [key: string]: JsonValue };

// Deep partial
type DeepPartial<T> = T extends object
  ? { [K in keyof T]?: DeepPartial<T[K]> }
  : T;

// Deep readonly
type DeepReadonly<T> = T extends (infer U)[]
  ? ReadonlyArray<DeepReadonly<U>>
  : T extends object
    ? { readonly [K in keyof T]: DeepReadonly<T[K]> }
    : T;
```

---

## `infer` Keyword

Extract type information from a type in a conditional.

```ts
// Extract return type of any function (like built-in ReturnType<F>)
type MyReturnType<F> = F extends (...args: any[]) => infer R ? R : never;

// Extract the first parameter
type FirstParam<F> = F extends (first: infer P, ...rest: any[]) => any ? P : never;

// Extract the resolved type of a Promise
type PromiseValue<T> = T extends Promise<infer V> ? V : T;
```

---

## `satisfies` Operator (TS 4.9+)

Validates a value against a type without widening the inferred type.

```ts
type Colors = 'red' | 'green' | 'blue';
type Palette = Record<Colors, string | [number, number, number]>;

// ✅ satisfies checks the type but keeps the narrow inferred type
const palette = {
  red:   [255, 0, 0],
  green: '#00ff00',
  blue:  [0, 0, 255],
} satisfies Palette;

// palette.red is still inferred as [number, number, number], not string | [number, number, number]
palette.red.map(v => v * 2); // ✅ works — map is available on the tuple
```

---

## Module Augmentation & Declaration Merging

Add properties to existing types from third-party libraries.

```ts
// Augment Express's Request type
import 'express';

declare module 'express' {
  interface Request {
    user?: { id: string; role: string };
  }
}

// Augment global Window
declare global {
  interface Window {
    analytics: AnalyticsInstance;
  }
}
```

---

## Variance Annotations (TS 4.7+)

For performance in very large/complex types. **Use only when profiling shows it's needed.**

```ts
// Covariant (out) — T only appears in output positions
type Producer<out T> = () => T;

// Contravariant (in) — T only appears in input positions
type Consumer<in T> = (value: T) => void;

// Invariant (default) — T appears in both
type Transform<in out T> = (value: T) => T;
```

---

## Narrowing Patterns

### Custom Type Guards
```ts
// Type predicate — user-defined narrowing
function isError(value: unknown): value is Error {
  return value instanceof Error;
}

// Assertion function — throws if assertion fails
function assertIsString(value: unknown): asserts value is string {
  if (typeof value !== 'string') {
    throw new TypeError(`Expected string, got ${typeof value}`);
  }
}
```

### `in` Operator Narrowing
```ts
type Cat = { meow(): void };
type Dog = { bark(): void };

function speak(animal: Cat | Dog) {
  if ('meow' in animal) {
    animal.meow(); // narrowed to Cat
  } else {
    animal.bark(); // narrowed to Dog
  }
}
```

### Discriminated Union Exhaustiveness
```ts
type Result<T> =
  | { tag: 'ok';    value: T }
  | { tag: 'err';   error: Error }
  | { tag: 'empty' };

function handle<T>(result: Result<T>): T | null {
  switch (result.tag) {
    case 'ok':    return result.value;
    case 'err':   throw result.error;
    case 'empty': return null;
    default:
      result satisfies never; // compile error if a new tag is added
      throw new Error('Impossible');
  }
}
```

---

## Index Signatures vs. Record

```ts
// Index signature — any string key
interface StringMap {
  [key: string]: string;
}

// Record — preferred, more explicit
type StringMap = Record<string, string>;

// With noUncheckedIndexedAccess enabled, index access returns T | undefined:
const map: Record<string, string> = {};
const val = map['key']; // string | undefined — handle both cases!
```

---

## `unknown` vs `any` vs `never`

| Type | Meaning | When to use |
|---|---|---|
| `unknown` | Could be anything; must narrow before use | External data, catch clauses, `JSON.parse` results |
| `any` | Anything goes; no type checking | **Avoid.** Only for gradual migration with a comment |
| `never` | Impossible / unreachable | Exhaustiveness checks, return type of functions that always throw |

```ts
// Correct catch clause handling
try {
  await riskyOp();
} catch (e: unknown) {
  if (e instanceof Error) {
    console.error(e.message);
  } else {
    console.error('Unknown error', e);
  }
}
```

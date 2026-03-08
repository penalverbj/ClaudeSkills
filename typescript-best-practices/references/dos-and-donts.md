# TypeScript Do's and Don'ts

Based on the official TypeScript Declaration Files
[Do's and Don'ts](https://www.typescriptlang.org/docs/handbook/declaration-files/do-s-and-don-ts.html)
and common mistakes catalog.

---

## General Types

### ❌ Don't use `Number`, `String`, `Boolean`, `Symbol`, `Object`
These are the boxed object wrappers, not the primitives. Almost never what you want.

```ts
// ❌
function greet(name: String): String { return 'Hello, ' + name; }

// ✅
function greet(name: string): string { return 'Hello, ' + name; }
```

### ❌ Don't use `Function` as a type
It accepts anything callable, with any arguments and return type.

```ts
// ❌
function run(fn: Function) { fn(); }

// ✅ — specific signature
function run(fn: () => void) { fn(); }

// ✅ — generic callback
function transform<T, U>(value: T, fn: (input: T) => U): U {
  return fn(value);
}
```

### ❌ Don't use `{}` as a "non-null/undefined" type
`{}` accepts everything except `null` and `undefined`, which is almost never the intent.
Use `object` for "any non-primitive", or a specific interface.

```ts
// ❌ Confusing
function merge(a: {}, b: {}): {} { return Object.assign({}, a, b); }

// ✅ Specific
function merge<T extends object, U extends object>(a: T, b: U): T & U {
  return Object.assign({}, a, b);
}
```

---

## Callbacks

### ❌ Don't use optional parameters in callbacks unnecessarily
If a callback doesn't need a parameter, it simply won't use it — TypeScript already allows that.

```ts
// ❌ Overly permissive
interface Fetcher {
  fetch(url: string, callback: (data: string, err?: Error) => void): void;
}

// ✅ Separate the success/error cases clearly
interface Fetcher {
  fetch(url: string, callback: (data: string) => void, onError: (err: Error) => void): void;
}
```

### ❌ Don't write separate overloads that differ only in callback arity
```ts
// ❌
declare function beforeAll(action: () => void, timeout?: number): void;
declare function beforeAll(action: (done: DoneFn) => void, timeout?: number): void;

// ✅ Use the widest callback type TypeScript can safely check
declare function beforeAll(action: (done?: DoneFn) => void, timeout?: number): void;
```

### ✅ Do use `void` return type for callbacks you don't use the return value of
```ts
// ✅ Signals callers they don't need to return anything meaningful
type EventHandler<T> = (event: T) => void;
```

---

## Overloads

### ❌ Don't write overloads that differ only by union types in one position
Collapse them into a single signature with a union.

```ts
// ❌ Redundant overloads
function format(x: string): string;
function format(x: number): string;
function format(x: string | number): string {
  return String(x);
}

// ✅
function format(x: string | number): string {
  return String(x);
}
```

### ✅ Do put more specific overloads before less specific ones
TypeScript resolves overloads top-to-bottom; broader signatures should come last.

```ts
// ✅
function createElement(tag: 'canvas'): HTMLCanvasElement;
function createElement(tag: 'div'): HTMLDivElement;
function createElement(tag: string): HTMLElement; // fallback last
```

---

## Classes

### ❌ Don't use `public` modifier — it's the default
```ts
// ❌ Noisy
class Greeter {
  public name: string;
  public greet(): void { /* ... */ }
}

// ✅
class Greeter {
  name: string;
  greet(): void { /* ... */ }
}
```

### ✅ Do use `readonly` on constructor-initialized, never-mutated properties
```ts
class Config {
  constructor(
    readonly apiUrl: string,
    readonly timeout: number,
  ) {}
}
```

### ✅ Do use `override` when re-implementing a base class method
Requires `"noImplicitOverride": true` in tsconfig — prevents silent typos.
```ts
class Animal {
  speak(): string { return 'Generic sound'; }
}

class Dog extends Animal {
  override speak(): string { return 'Woof'; } // ✅ explicit intent
}
```

---

## Enums

### ❌ Don't use `const enum` in library `.d.ts` files
`const enum` values are inlined at compile time; external consumers with different
tsconfigs (e.g., Babel, esbuild) won't see the values correctly.

```ts
// ❌ In a library's .d.ts
export const enum Direction { Up, Down }

// ✅ In library code — use a regular enum or a string union
export enum Direction { Up = 'up', Down = 'down' }
// or better:
export type Direction = 'up' | 'down';
export const Direction = { Up: 'up', Down: 'down' } as const;
```

### ❌ Don't mix numeric and string members in an enum
```ts
// ❌ Confusing — numeric members have reverse mappings, string members don't
enum Mixed {
  A = 1,
  B = 'b',
}
```

---

## Modules & Namespaces

### ❌ Don't use namespaces in module files
Namespaces are a legacy pattern from before ES modules. Use ES module `import`/`export`.

```ts
// ❌ Old-style
namespace Validation {
  export function isEmail(s: string): boolean { /* ... */ }
}

// ✅
export function isEmail(s: string): boolean { /* ... */ }
```

### ✅ Do use `import type` for type-only imports
Avoids importing runtime values just for type information; required with `verbatimModuleSyntax`.

```ts
import type { User, UserRole } from './user.types';
import { createUser } from './user.service';
```

### ✅ Do prefer named exports over default exports
Default exports are harder to refactor, lose their name in imports, and don't appear in auto-import suggestions as reliably.

```ts
// ❌
export default function parseDate(s: string): Date { /* ... */ }

// ✅
export function parseDate(s: string): Date { /* ... */ }
```

---

## Common Mistakes

### Using `||` instead of `??` for default values
`||` treats `0`, `''`, and `false` as falsy; `??` only triggers on `null`/`undefined`.

```ts
// ❌ Returns 'default' if count is 0
const count = userCount || 'default';

// ✅ Returns 0 correctly
const count = userCount ?? 'default';
```

### Using `==` instead of `===`
`==` performs implicit coercion; always use `===` in TypeScript.

```ts
// ❌
if (value == null) { /* matches both null and undefined */ }

// ✅ explicit
if (value === null || value === undefined) { /* ... */ }
// or using nullish:
if (value == null) { /* acceptable shorthand for both */ }
```

### Ignoring `noUncheckedIndexedAccess` implications
With this flag on, array/record access returns `T | undefined`. Handle it.

```ts
const items = ['a', 'b', 'c'];
const first = items[0]; // string | undefined

// ❌
console.log(first.toUpperCase()); // potential runtime crash

// ✅
console.log(first?.toUpperCase() ?? '');
```

### Type assertions instead of narrowing
```ts
// ❌ Asserts without checking; crashes if wrong
const el = document.getElementById('app') as HTMLDivElement;

// ✅ Check first, then use
const el = document.getElementById('app');
if (!(el instanceof HTMLDivElement)) throw new Error('Missing #app element');
el.classList.add('active');
```

### Forgetting to handle `undefined` in optional chaining results
```ts
interface Config { db?: { host: string } }

// ❌ host is string | undefined but used as string
const host: string = config.db?.host; // Type error under strict

// ✅
const host = config.db?.host ?? 'localhost';
```

### Using `any` in a catch clause
Under `"useUnknownInCatchVariables": true` (included in `strict`), catch params are `unknown`.

```ts
// ❌
try { /* ... */ } catch (e: any) { console.error(e.message); }

// ✅
try { /* ... */ } catch (e: unknown) {
  const msg = e instanceof Error ? e.message : String(e);
  console.error(msg);
}
```

# type-smith — TypeScript / JavaScript Type System Expert

## Frontmatter

```yaml
id: type-smith
name: type-smith
description: >
  TypeScript and JavaScript type system expert. Full coverage of conditional
  types, infer keyword, mapped types, template literal types, discriminated
  unions, recursive types, branded types, builder pattern types, and the
  complete utility type catalog. Use when debugging "type error", writing
  complex generics, understanding "infer", "conditional types", "mapped types",
  or designing a type-safe API surface. Triggers on: TypeScript generics,
  complex types, type error, infer, conditional types, mapped types,
  discriminated unions, branded types, utility types.
triggers:
  - TypeScript generics
  - complex types
  - type error
  - infer keyword
  - conditional types
  - mapped types
  - discriminated unions
  - branded types
  - template literal types
  - utility types
  - ".ts files"
  - ".tsx files"
```

---

## 1. Investigate Before Typing

Before writing a type fix, read the actual type error message in full — do not guess. TypeScript errors contain the exact shape mismatch. Common misread:

- `Type 'X' is not assignable to type 'Y'` — read both sides carefully; the problem is usually at the deepest mismatch level, not the top level.
- `Type instantiation is excessively deep` — you have a recursive type without a depth exit; add a depth counter.
- `Argument of type 'X' is not assignable to parameter of type 'never'` — a conditional branch resolved to `never`; trace which branch collapsed it.

---

## 2. Utility Type Catalog (built-in)

| Utility | Effect | When to use |
|---|---|---|
| `Partial<T>` | all keys optional | update payloads, patches |
| `Required<T>` | all keys required | after validating an optional input |
| `Readonly<T>` | all keys readonly | immutable config, frozen state |
| `Pick<T, K>` | keep only K keys | DTOs, projection |
| `Omit<T, K>` | drop K keys | strip internal fields before sending |
| `Extract<T, U>` | keep union members assignable to U | narrow a union |
| `Exclude<T, U>` | remove union members assignable to U | complement of Extract |
| `NonNullable<T>` | remove null and undefined | after a null check |
| `ReturnType<F>` | extract return type of function F | mirror a function's output shape |
| `Parameters<F>` | extract params tuple of function F | wrap or proxy a function |
| `ConstructorParameters<C>` | extract constructor params | factory helpers |
| `InstanceType<C>` | extract instance type from class | generic class factory |
| `Awaited<T>` | unwrap Promise recursively | async return types |
| `Record<K, V>` | object with keys K and values V | dictionaries, lookup tables |

---

## 3. Conditional Types

```ts
// Basic form
type IsString<T> = T extends string ? true : false;

// With infer — extract a piece of the type
type UnwrapPromise<T> = T extends Promise<infer R> ? R : T;
type ElementType<T> = T extends (infer E)[] ? E : never;
type FirstArg<F> = F extends (first: infer A, ...rest: any[]) => any ? A : never;

// Distributive conditional type (distributes over union members)
type ToArray<T> = T extends any ? T[] : never;
// ToArray<string | number> => string[] | number[]

// Prevent distribution with tuple wrapping
type ToArrayNonDist<T> = [T] extends [any] ? T[] : never;
// ToArrayNonDist<string | number> => (string | number)[]
```

**Key rule:** bare `T extends ...` distributes; `[T] extends [...]` does not. Pick intentionally.

---

## 4. Mapped Types

```ts
// Remap all keys
type Stringify<T> = { [K in keyof T]: string };

// Key remapping with as (TS 4.1+)
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};

// Filter keys by value type
type OnlyStrings<T> = {
  [K in keyof T as T[K] extends string ? K : never]: T[K];
};

// Mutable version of Readonly
type Mutable<T> = { -readonly [K in keyof T]: T[K] };

// Required version (remove ?)
type Concrete<T> = { [K in keyof T]-?: T[K] };
```

---

## 5. Template Literal Types

```ts
type EventName<T extends string> = `on${Capitalize<T>}`;
// EventName<'click'> => 'onClick'

type CSSProperty = `${string}-${string}`;

// Path autocomplete pattern
type Paths<T, Prefix extends string = ""> = {
  [K in keyof T & string]: T[K] extends object
    ? Paths<T[K], `${Prefix}${K}.`>
    : `${Prefix}${K}`;
}[keyof T & string];
```

---

## 6. Discriminated Unions

Always add a **literal type discriminant** (`kind`, `type`, `tag`) as the first property. TypeScript narrows automatically in switch/if.

```ts
type Shape =
  | { kind: "circle";   radius: number }
  | { kind: "square";   side: number   }
  | { kind: "rect";     w: number; h: number };

function area(s: Shape): number {
  switch (s.kind) {
    case "circle": return Math.PI * s.radius ** 2;
    case "square": return s.side ** 2;
    case "rect":   return s.w * s.h;
    // exhaustiveness check:
    default: return s satisfies never;
  }
}
```

**Exhaustiveness pattern:** `s satisfies never` (TS 4.9+) — compile error if a branch is unhandled.

---

## 7. Branded / Nominal Types

TypeScript is structurally typed. Branding makes two `string` types incompatible at compile time.

```ts
type Brand<T, B> = T & { readonly __brand: B };

type UserId    = Brand<string, "UserId">;
type ProductId = Brand<string, "ProductId">;

function getUser(id: UserId): User { ... }

const uid = "abc" as UserId;
const pid = "abc" as ProductId;
getUser(uid); // ✅
getUser(pid); // ❌ type error — correct behavior
```

Use for IDs, validated strings (email, URL), currency amounts — anywhere structural equivalence is dangerous.

---

## 8. Recursive Types

```ts
// JSON value type
type Json =
  | string | number | boolean | null
  | Json[]
  | { [key: string]: Json };

// Deep partial
type DeepPartial<T> = T extends object
  ? { [K in keyof T]?: DeepPartial<T[K]> }
  : T;

// Depth-limited to avoid "excessively deep" error
type DeepReadonly<T, D extends number = 5> = D extends 0
  ? T
  : T extends object
  ? { readonly [K in keyof T]: DeepReadonly<T[K], [-1,0,1,2,3,4][D]> }
  : T;
```

If you hit `Type instantiation is excessively deep`, add a depth counter tuple as shown.

---

## 9. Builder Pattern Types

```ts
type Builder<T, Required extends keyof T = never> = {
  set<K extends keyof T>(key: K, value: T[K]): Builder<T, Required | K>;
  build(): Required extends keyof T ? T : never;
};
```

Forces callers to set all required fields before `build()` compiles; missing fields produce `never`.

---

## 10. Advanced Patterns Cheatsheet

```ts
// Awaited deep unwrap
type DeepAwaited<T> = T extends Promise<infer R> ? DeepAwaited<R> : T;

// Function overload selector
type Overloads<T> = T extends {
  (...args: infer A1): infer R1;
  (...args: infer A2): infer R2;
} ? [(...args: A1) => R1, (...args: A2) => R2] : never;

// Strict object (no excess properties in callbacks)
type StrictRecord<T> = { [K in keyof T]: T[K] } & Record<string, never>;

// XOR type (exactly one of A or B)
type XOR<A, B> =
  | (A & { [K in Exclude<keyof B, keyof A>]?: never })
  | (B & { [K in Exclude<keyof A, keyof B>]?: never });
```

---

## 11. Common Type Errors and Fixes

| Error | Cause | Fix |
|---|---|---|
| `Type 'X' is not assignable to 'never'` | Union branch collapsed | Trace the conditional; likely a missing case or wrong extends |
| `Argument of type 'string' is not assignable to 'never'` | Generic constrained to `never` | Check the generic bound; you probably narrowed too hard |
| `Property 'X' does not exist on type 'Y \| Z'` | Accessing property before narrowing | Add discriminant check or use optional chaining |
| `Excessive stack depth` | Recursive type with no exit | Add depth counter or use `interface` (interfaces are lazier) |
| `This expression is not callable` | Type is `never` or union with non-callable | Narrow first; check conditional type branches |
| `Object is possibly 'undefined'` | Optional chain not handled | Add `!` (only if certain) or default with `?? fallback` |
| `Type 'X' does not satisfy the constraint 'Y'` | Generic arg violates extends bound | Read the bound; restructure the arg or widen the bound |

---

## 12. tsconfig Flags That Affect Type Safety

Always-on in new projects:

```json
{
  "strict": true,           // enables all strict flags below
  "noUncheckedIndexedAccess": true,  // arr[0] is T | undefined
  "exactOptionalPropertyTypes": true, // { x?: T } forbids { x: undefined }
  "noImplicitOverride": true,
  "useUnknownInCatchVariables": true
}
```

`noUncheckedIndexedAccess` is the single highest-value flag not in `strict`. Turn it on early; retrofitting is painful.

---

## 13. Verification

After writing a type, run:

```sh
tsc --noEmit          # zero type errors
tsc --noEmit --strict # confirm strict mode clean
```

Never ship a `@ts-ignore` or `any` without a dated comment explaining why and a tracking ticket.

---

gates — godmode:activation: fired | /honestscore: skipped: skill file, not code | /ideas: skipped: skill authoring, no defect | brain: skipped: subagent seat, no brain access

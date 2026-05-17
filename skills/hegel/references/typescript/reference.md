# Hegel TypeScript Reference

## Table of Contents

- [Setup](#setup)
- [Test Structure](#test-structure) — `hegel.test`, `hegel.testAsync`, `Settings`, `HealthCheck`, database
- [TestCase Methods](#testcase-methods) — `draw`, `assume`, `note`
- [Generator Reference](#generator-reference) — Numeric, boolean, text, characters, binary, collections, tuples, optional, format, regex
- [Combinator Methods](#combinator-methods) — `.map()`, `.filter()`, `.flatMap()`
- [Composite Generators](#composite-generators) — `composite`, `record`
- [TypeScript-Specific Examples](#typescript-specific-examples) — Async, dependent generation, BigInt
- [Gotchas](#gotchas)

## Setup

```bash
npm install --save-dev @hegeldev/hegel
```

Hegel requires **Node 16+**. Bun and Deno are not currently supported.

Hegel is **test-runner agnostic** — `hegel.test(...)` is a regular function that runs the property, shrinks on failure, and throws on the minimal counterexample. Use whatever runner the project already uses (Vitest, Jest, Mocha, node:test).

Run your tests with the existing runner (e.g. `npx vitest run`). The first invocation auto-installs the `hegel-core` Python server via `uv`. If something goes wrong with that, see https://hegel.dev/reference/installation.

## Test Structure

### `hegel.test` (sync) and `hegel.testAsync` (async)

`hegel.test` runs immediately when called and returns `void`. To use it with a test runner that expects a callback, wrap the call in `() => hegel.test(...)`:

```typescript
import { test } from "vitest";
import * as hegel from "@hegeldev/hegel";
import * as gs from "@hegeldev/hegel/generators";

test("addition commutes", () =>
  hegel.test((tc) => {
    const a = tc.draw(gs.integers());
    const b = tc.draw(gs.integers());
    if (a + b !== b + a) {
      throw new Error(`not commutative: ${a} + ${b}`);
    }
  }));
```

For async test bodies, use `hegel.testAsync`. It returns `Promise<void>` that resolves when the test completes:

```typescript
test("fetch round-trip", () =>
  hegel.testAsync(async (tc) => {
    const id = tc.draw(gs.integers({ minValue: 1, maxValue: 1000 }));
    const user = await fetchUser(id);
    if (user.id !== id) {
      throw new Error(`expected id=${id}, got ${user.id}`);
    }
  }));
```

`hegel.test` throws `TypeError` if you pass it an async function — use `hegel.testAsync` for those.

### Settings

Both `hegel.test` and `hegel.testAsync` accept an optional second argument: a `Partial<Settings>` object overriding defaults.

```typescript
import { Verbosity, HealthCheck, Database } from "@hegeldev/hegel";

hegel.test(
  (tc) => { /* ... */ },
  {
    testCases: 500,
    verbosity: Verbosity.Verbose,
    seed: 42,
  },
);
```

| Field | Type | Default | Purpose |
|-------|------|---------|---------|
| `testCases` | `number` | `100` | Number of test cases to run |
| `seed` | `number \| null` | `null` | Fixed RNG seed for reproducibility |
| `verbosity` | `Verbosity` | `Normal` | `Quiet`, `Normal`, `Verbose`, `Debug` |
| `derandomize` | `boolean` | `true` in CI | Use a deterministic seed derived from the test |
| `database` | `Database` | `unset` (`disabled` in CI) | Failing-example persistence |
| `suppressHealthCheck` | `HealthCheck[]` | `[]` | Suppress specific health checks |

### HealthCheck

`HealthCheck` is a string enum:

- `HealthCheck.FilterTooMuch` — Too many test cases rejected via `assume()` or `.filter()`
- `HealthCheck.TooSlow` — Test execution is too slow
- `HealthCheck.TestCasesTooLarge` — Generated test cases are too large
- `HealthCheck.LargeInitialTestCase` — The smallest natural input is very large

```typescript
hegel.test(
  (tc) => { /* ... */ },
  { suppressHealthCheck: [HealthCheck.FilterTooMuch] },
);
```

### Example database

Hegel persists failing examples to a `.hegel/` directory and replays them on subsequent runs. The database key is auto-derived from the test function's source, so replay works without extra setup. In CI environments the database is disabled automatically.

Override via the `database` setting:

```typescript
import { Database } from "@hegeldev/hegel";

// Custom directory
hegel.test((tc) => { /* ... */ }, { database: Database.fromPath("my_hegel_db") });

// Disable persistence
hegel.test((tc) => { /* ... */ }, { database: Database.disabled });
```

## TestCase Methods

| Method | Signature | Purpose |
|--------|-----------|---------|
| `draw` | `draw<T>(g: Generator<T>): T` | Draw a value from a generator; shown in counterexample output |
| `assume` | `assume(condition: boolean): void` | Reject this test case if `condition` is false |
| `note` | `note(message: string): void` | Record debug info (only printed on the final counterexample replay) |

### Usage

```typescript
hegel.test((tc) => {
  const a = tc.draw(gs.integers());
  const b = tc.draw(gs.integers());
  tc.assume(b !== 0);
  tc.note(`dividing ${a} by ${b}`);
  const q = Math.trunc(a / b);
  const r = a % b;
  if (a !== q * b + r) {
    throw new Error(`division identity violated: ${a} != ${q}*${b} + ${r}`);
  }
});
```

Signal a failure by throwing — any uncaught exception inside the test body is treated as a failing test case. Use your runner's normal assertion library (Vitest's `expect`, Node's `assert`, etc.).

## Generator Reference

All generators live in `@hegeldev/hegel/generators`. Idiomatic import:

```typescript
import * as gs from "@hegeldev/hegel/generators";
```

You can also reach them through the main module as `hegel.generators.integers()`, but the dedicated import is preferred.

Generators take their configuration as a single options object. Omit the object (or any field) to use defaults.

### Numeric Generators

**`gs.integers(options?)`** — Generate `number` values in the JS safe-integer range

```typescript
const n = tc.draw(gs.integers());                              // any safe integer
const bounded = tc.draw(gs.integers({ minValue: 0, maxValue: 100 }));
```

Fields:
- `minValue?: number` (default: `Number.MIN_SAFE_INTEGER`)
- `maxValue?: number` (default: `Number.MAX_SAFE_INTEGER`)

Throws at construction time if bounds are outside the safe-integer range. Use `bigIntegers()` for arbitrary precision.

**`gs.bigIntegers(options?)`** — Generate arbitrary-precision `bigint` values

```typescript
const n = tc.draw(gs.bigIntegers());                                       // unbounded
const big = tc.draw(gs.bigIntegers({ minValue: 0n, maxValue: 2n ** 256n })); // bigint bounds
```

Fields: `minValue?: bigint`, `maxValue?: bigint` (both default to unbounded).

**`gs.floats(options?)`** — Generate IEEE-754 `number` values (64-bit doubles)

```typescript
const f = tc.draw(gs.floats());
const unit = tc.draw(gs.floats({ minValue: 0, maxValue: 1 }));
const openInterval = tc.draw(gs.floats({
  minValue: 0, maxValue: 1, excludeMin: true, excludeMax: true,
}));
```

Fields:
- `minValue?: number`, `maxValue?: number`
- `excludeMin?: boolean`, `excludeMax?: boolean` (default `false`)
- `allowNan?: boolean` (default: `true` if completely unbounded, `false` otherwise)
- `allowInfinity?: boolean` (default: `true` if at least one bound is missing)

### Boolean Generator

```typescript
const b = tc.draw(gs.booleans());
```

### Text and Binary Generators

**`gs.text(options?)`** — Generate `string` values (full Unicode by default)

```typescript
const s = tc.draw(gs.text());
const bounded = tc.draw(gs.text({ minSize: 1, maxSize: 50 }));
const ascii = tc.draw(gs.text({ codec: "ascii" }));
const abc = tc.draw(gs.text({ alphabet: "abc" }));
```

Fields:
- `minSize?: number` (default `0`)
- `maxSize?: number`
- `alphabet?: string` — Fixed allowed characters (mutually exclusive with the other character filters)
- `codec?: string` — e.g. `"ascii"`, `"utf-8"`, `"latin-1"`
- `minCodepoint?: number` / `maxCodepoint?: number`
- `categories?: readonly string[]` — Unicode general categories (e.g. `["L", "Nd"]`)
- `excludeCategories?: readonly string[]`
- `includeCharacters?: string` — Always include these
- `excludeCharacters?: string` — Always exclude these

**`gs.characters(options?)`** — Generate a single-codepoint `string`. Same character-filtering options as `text` (no size fields, no `alphabet`).

**`gs.binary(options?)`** — Generate `Uint8Array`

```typescript
const bytes = tc.draw(gs.binary());
const sized = tc.draw(gs.binary({ minSize: 1, maxSize: 256 }));
```

Fields: `minSize?: number`, `maxSize?: number`.

**`gs.fromRegex(pattern, options?)`** — Generate strings matching a regex

```typescript
const code = tc.draw(gs.fromRegex("[A-Z]{3}-[0-9]{3}", { fullmatch: true }));
```

The pattern is a string (not a JS `RegExp` literal). `fullmatch` controls whether the entire string must match.

### Constant and Choice Generators

```typescript
const x = tc.draw(gs.just(42));                                      // always 42
const suit = tc.draw(gs.sampledFrom(["hearts", "diamonds", "clubs", "spades"]));
```

`sampledFrom` throws if the array is empty.

### Collection Generators

**`gs.arrays(elements, options?)`** — Generate `T[]`

```typescript
const xs = tc.draw(gs.arrays(gs.integers()));
const bounded = tc.draw(gs.arrays(gs.integers(), { minSize: 1, maxSize: 10 }));
const unique = tc.draw(gs.arrays(gs.integers(), { unique: true }));
```

Fields: `minSize?`, `maxSize?`, `unique?` (default `false`). Uniqueness is checked via `JSON.stringify` equality.

**`gs.sets(elements, options?)`** — Generate `Set<T>`. Fields: `minSize?`, `maxSize?` (uniqueness is implicit).

**`gs.maps(keys, values, options?)`** — Generate `Map<K, V>`. Fields: `minSize?`, `maxSize?`.

```typescript
const m = tc.draw(gs.maps(gs.text(), gs.integers(), { maxSize: 5 }));
```

### Tuple Generator

**`gs.tuples(gen1, gen2, ...)`** — Variadic; returns a generator of a tuple type whose element types are inferred from the arguments.

```typescript
const pair = tc.draw(gs.tuples(gs.integers(), gs.text()));   // [number, string]
const triple = tc.draw(gs.tuples(gs.booleans(), gs.integers(), gs.floats()));
```

### Optional

```typescript
const maybe = tc.draw(gs.optional(gs.integers()));   // number | null
```

`optional(g)` returns `Generator<T | null>` — note `null`, not `undefined`.

### Format Generators

```typescript
const email  = tc.draw(gs.emails());
const url    = tc.draw(gs.urls());
const domain = tc.draw(gs.domains({ maxLength: 50 }));
const date   = tc.draw(gs.dates());        // ISO 8601 date string
const time   = tc.draw(gs.times());        // ISO 8601 time string
const dt     = tc.draw(gs.datetimes());    // ISO 8601 datetime string
const ip     = tc.draw(gs.ipAddresses());            // IPv4 or IPv6
const ipv4   = tc.draw(gs.ipAddresses({ version: 4 }));
const ipv6   = tc.draw(gs.ipAddresses({ version: 6 }));
```

Date/time/datetime generators return ISO 8601 strings, not `Date` objects.

## Combinator Methods

Every `Generator<T>` has chainable methods. They return a new generator (cheap to create).

### `.map(f)`

Transform generated values:

```typescript
const positiveStr = gs.integers({ minValue: 1 }).map((n) => n.toString());
```

`.map()` preserves the underlying schema when the source is schema-backed, so mapped primitives are still generated efficiently on the server.

### `.filter(predicate)`

Keep only values matching a predicate:

```typescript
const even = gs.integers().filter((x) => x % 2 === 0);
```

`.filter()` retries up to 3 times, then calls `tc.assume(false)`. Prefer bounds or constructing valid inputs directly.

### `.flatMap(f)`

Dependent generation — use one value to choose the next generator:

```typescript
const sizedString = gs.integers({ minValue: 1, maxValue: 10 })
  .flatMap((len) => gs.text({ minSize: len, maxSize: len }));
```

In most cases, prefer sequential `tc.draw()` calls inside the test body — they read more naturally and produce the same shrinking behavior. Use `.flatMap()` when you need the result packaged as a `Generator<U>`.

## Composite Generators

Hegel provides two ways to build generators for composite types: `composite` (imperative) and `record` (declarative).

### `composite(fn)`

Build a generator from imperative code. The callback receives a `TestCase` and calls `tc.draw()` on inner generators:

```typescript
interface Person {
  name: string;
  age: number;
  drivingLicense: boolean;
}

const personGen = gs.composite<Person>((tc) => {
  const age = tc.draw(gs.integers({ minValue: 0, maxValue: 120 }));
  const name = tc.draw(gs.text({ minSize: 1, maxSize: 50 }));
  const drivingLicense = age >= 18 ? tc.draw(gs.booleans()) : false;
  return { name, age, drivingLicense };
});

hegel.test((tc) => {
  const p = tc.draw(personGen);
  // ...
});
```

The return type is inferred from the callback, or pass it as an explicit type argument (`gs.composite<Person>(...)`) when inference is ambiguous.

`composite` is the right tool when fields depend on each other, or when you need conditional or repeated draws.

### `record(schema)`

Declarative alternative when every field is independent. Pass an object mapping field names to generators:

```typescript
const userGen = gs.record({
  name: gs.text({ minSize: 1 }),
  age: gs.integers({ minValue: 0, maxValue: 120 }),
  active: gs.booleans(),
});

hegel.test((tc) => {
  const user = tc.draw(userGen);
  // user is typed as { name: string; age: number; active: boolean }
});
```

`record` infers the result type from the schema and uses the basic-schema path (no per-field span overhead) when all fields are schema-backed.

## TypeScript-Specific Examples

These examples show TypeScript-specific idioms. For general property patterns (round-trip, model-based, idempotence, etc.), see the main skill's Property Catalogue.

### Dependent generation with sequential draws

Hegel's imperative style means dependent generation is just sequential code — no `flatMap` needed:

```typescript
hegel.test((tc) => {
  const xs = tc.draw(gs.arrays(gs.integers(), { minSize: 1 }));
  const idx = tc.draw(gs.integers({ minValue: 0, maxValue: xs.length - 1 }));
  // idx is always a valid index
  void xs[idx];
});
```

### Async test bodies

`hegel.testAsync` awaits the test body before counting the case as passed:

```typescript
test("user round-trip", () =>
  hegel.testAsync(async (tc) => {
    const user = tc.draw(userGen);
    const saved = await db.save(user);
    const loaded = await db.load(saved.id);
    expect(loaded).toEqual(user);
  }));
```

Don't mix sync and async — `hegel.test` rejects async callbacks with a `TypeError`.

### Integers beyond the safe range

`gs.integers()` only produces JS `number` values up to `Number.MAX_SAFE_INTEGER` (`2^53 - 1`). For arbitrary-precision values, use `gs.bigIntegers()`:

```typescript
hegel.test((tc) => {
  const n = tc.draw(gs.bigIntegers());   // bigint, unbounded
  const u256 = tc.draw(gs.bigIntegers({ minValue: 0n, maxValue: 2n ** 256n - 1n }));
  // ...
});
```

If a function under test says it accepts arbitrarily large integers, test it with `bigIntegers()` — `integers()` will silently cap at `MAX_SAFE_INTEGER` and miss precision-loss bugs.

## Gotchas

1. **`hegel.test` runs immediately and returns `void`.** Wrap in `() => hegel.test(...)` to pass to a test-runner callback (`test("name", () => hegel.test(...))`). Calling `hegel.test(...)` as the value of `test("name", hegel.test(...))` would execute the property at module-load time, which is almost never what you want.

2. **`hegel.test` rejects async callbacks; use `hegel.testAsync`.** Passing an `async` function to `hegel.test` throws `TypeError`. Awaiting a `Promise` in `hegel.test` would resolve outside the test case lifecycle.

3. **`gs.integers()` only covers the JS safe-integer range.** It throws at construction if bounds exceed `Number.MAX_SAFE_INTEGER`. Use `gs.bigIntegers()` for arbitrary-precision integers.

4. **`gs.binary()` returns `Uint8Array`, not `Buffer` or `string`.** Convert with `Buffer.from(bytes)` if a downstream API expects a `Buffer`.

5. **Date/time/datetime generators return strings, not `Date`.** They emit ISO 8601 strings — parse with `new Date(s)` if you need a `Date` object.

6. **`gs.optional(g)` produces `T | null`, not `T | undefined`.** Check for `=== null`, not `=== undefined`.

7. **Float defaults include NaN and infinity when unbounded.** `gs.floats()` with no bounds generates `NaN` and `Infinity`. If your code doesn't handle them, pass `{ allowNan: false, allowInfinity: false }` — but consider whether the code *should* handle them first.

8. **Excessive `assume`/`filter` rejections fail the test.** If `tc.assume()` or `.filter()` rejects too many inputs, the `FilterTooMuch` health check fires. Restructure generators to produce valid inputs directly (use `.map()` or sequential draws).

9. **`note()` only prints on the final replay.** Don't rely on it for progress logging — it only appears when the minimal counterexample is displayed.

10. **Default collection sizes are small.** `gs.arrays(gen)` with no bounds rarely produces 100+ elements. To exercise deep traversals, draw the size separately:
    ```typescript
    const n = tc.draw(gs.integers({ minValue: 0, maxValue: 300 }));
    const xs = tc.draw(gs.arrays(gs.integers(), { minSize: n }));
    ```

11. **Use `unique: true` for key generation.** When generating keys for a `Map` or `Set`, prefer `gs.arrays(keyGen, { unique: true })` to avoid ambiguity about which value wins:
    ```typescript
    const keys = tc.draw(gs.arrays(gs.integers(), { maxSize: 50, unique: true }));
    ```

12. **Add `.hegel/` to `.gitignore`.** Hegel caches the server binary and the failing-example database under `.hegel/` in your project root.

13. **`fromRegex` takes a string pattern, not a `RegExp`.** Pass `"[A-Z]{3}"`, not `/[A-Z]{3}/`.

14. **`target()` and stateful testing are not yet available** in hegel-typescript. They are planned for future releases. Until stateful testing lands, write rule loops by hand inside `hegel.test`/`hegel.testAsync` (draw a rule choice with `gs.sampledFrom`, dispatch, assert invariants).

15. **Node 16+ only.** Bun and Deno are not currently supported.

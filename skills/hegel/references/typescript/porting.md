# Porting TypeScript PBT Libraries to Hegel

## From fast-check

[fast-check](https://fast-check.dev) is overwhelmingly the most common TypeScript PBT library. The main differences:

- fast-check is declarative: arbitraries are declared up front and threaded into the property via `fc.property(...)`. Hegel is imperative — your test receives a `TestCase` and calls `tc.draw()` whenever it needs a value.
- fast-check does shrinking in-process; hegel delegates shrinking to a Python server (Hypothesis).
- fast-check tests return `true`/`void`/throw; hegel tests use the surrounding runner's normal assertions (`expect`, `assert`, throw).
- fast-check has `fc.commands` for stateful testing and `fc.pre` for preconditions; hegel uses `tc.assume()` (stateful testing is not yet available in hegel-typescript).

### Test Structure

fast-check (with Vitest):

```typescript
import * as fc from "fast-check";

test("addition is commutative", () => {
  fc.assert(
    fc.property(fc.integer(), fc.integer(), (a, b) => {
      return a + b === b + a;
    }),
  );
});
```

Hegel:

```typescript
import * as hegel from "@hegeldev/hegel";
import * as gs from "@hegeldev/hegel/generators";

test("addition is commutative", () =>
  hegel.test((tc) => {
    const a = tc.draw(gs.integers());
    const b = tc.draw(gs.integers());
    expect(a + b).toBe(b + a);
  }));
```

Notes:
- `hegel.test(...)` runs immediately, so wrap it in `() => hegel.test(...)` to pass to the runner's callback.
- Use the runner's normal assertions inside the body. No need for `fc.property`'s "return false/true" pattern.
- Consider whether any bounds in the original fast-check test are justified. If a property is about addition over all integers, test the full safe-integer range — don't carry over a narrow `fc.integer({ min: 0, max: 100 })` unless the function's contract justifies it.

### Arbitrary → Generator Mapping

| fast-check | Hegel |
|------------|-------|
| `fc.integer()` | `gs.integers()` (safe-integer range) |
| `fc.integer({ min, max })` | `gs.integers({ minValue: min, maxValue: max })` |
| `fc.nat()` | `gs.integers({ minValue: 0 })` |
| `fc.nat(max)` | `gs.integers({ minValue: 0, maxValue: max })` |
| `fc.maxSafeInteger()` | `gs.integers()` |
| `fc.maxSafeNat()` | `gs.integers({ minValue: 0 })` |
| `fc.bigInt()` | `gs.bigIntegers()` |
| `fc.bigInt({ min, max })` | `gs.bigIntegers({ minValue: min, maxValue: max })` |
| `fc.bigUint()` | `gs.bigIntegers({ minValue: 0n })` |
| `fc.float()` / `fc.double()` | `gs.floats()` |
| `fc.float({ min, max })` | `gs.floats({ minValue: min, maxValue: max })` |
| `fc.boolean()` | `gs.booleans()` |
| `fc.string()` | `gs.text()` |
| `fc.string({ minLength, maxLength })` | `gs.text({ minSize: minLength, maxSize: maxLength })` |
| `fc.asciiString()` | `gs.text({ codec: "ascii" })` |
| `fc.unicodeString()` | `gs.text()` (hegel defaults to full Unicode) |
| `fc.char()` / `fc.unicode()` | `gs.characters()` |
| `fc.uint8Array()` | `gs.binary()` |
| `fc.stringMatching(re)` | `gs.fromRegex(re.source, { fullmatch: true })` |
| `fc.constant(x)` | `gs.just(x)` |
| `fc.constantFrom(...xs)` | `gs.sampledFrom([...xs])` |
| `fc.oneof(a, b)` | `gs.oneOf(a, b)` |
| `fc.option(a)` | `gs.optional(a)` *(returns `T \| null`, not `T \| undefined`)* |
| `fc.array(a)` | `gs.arrays(a)` |
| `fc.array(a, { minLength, maxLength })` | `gs.arrays(a, { minSize, maxSize })` |
| `fc.uniqueArray(a)` | `gs.arrays(a, { unique: true })` |
| `fc.set(a)` *(if used)* | `gs.sets(a)` *(returns `Set<T>`, not `T[]`)* |
| `fc.tuple(a, b)` | `gs.tuples(a, b)` |
| `fc.record({ k: a })` | `gs.record({ k: a })` |
| `fc.dictionary(k, v)` | `gs.maps(k, v)` *(returns `Map<K, V>`, not plain object)* |
| `fc.emailAddress()` | `gs.emails()` |
| `fc.webUrl()` | `gs.urls()` |
| `fc.domain()` | `gs.domains()` |
| `fc.ipV4()` / `fc.ipV6()` | `gs.ipAddresses({ version: 4 })` / `gs.ipAddresses({ version: 6 })` |
| `fc.date()` | `gs.datetimes()` *(returns ISO 8601 string, not `Date`)* |
| `a.map(f)` | `a.map(f)` |
| `a.filter(p)` | `a.filter(p)` |
| `a.chain(f)` | `a.flatMap(f)` |
| `fc.gen()` / custom arbitrary | `gs.composite(...)` |

A few mappings need extra care:

- **`fc.option(a)` returns `T | null` by default** in fast-check (configurable). `gs.optional(a)` always returns `T | null` — there is no equivalent `withUndefined` option. If the code under test expects `undefined`, use `.map((x) => x === null ? undefined : x)`.
- **`fc.dictionary(keyArb, valueArb)` returns a plain object**; `gs.maps(keyArb, valueArb)` returns a `Map`. If the code under test expects a plain object, use `gs.record({...})` for known keys or `gs.arrays(gs.tuples(keyArb, valueArb), { unique: true }).map(Object.fromEntries)` for arbitrary keys.
- **`fc.date()` returns a `Date` object**; `gs.datetimes()` returns an ISO 8601 string. Convert with `gs.datetimes().map((s) => new Date(s))` if you need `Date`.
- **`fc.bigInt()` has no default bounds**; `gs.bigIntegers()` is also unbounded by default. Both are fine.
- **`fc.integer()` is unbounded 32-bit by default**; `gs.integers()` uses the full JS safe-integer range. If the original test was relying on the smaller default, the broader hegel default may find new bugs — that's the point.

### Assertions and Preconditions

| fast-check | Hegel |
|------------|-------|
| `return true` / `return false` | Use the runner's assertions (`expect(...).toBe(...)`) and throw on failure |
| `expect(...).toBe(...)` inside `fc.property` | Same — `expect(...).toBe(...)` inside `hegel.test` |
| `fc.pre(cond)` | `tc.assume(cond)` |
| `throw new fc.PreconditionFailure()` | `tc.assume(false)` |

### Configuration

| fast-check | Hegel |
|------------|-------|
| `fc.assert(prop, { numRuns: 500 })` | `hegel.test((tc) => { ... }, { testCases: 500 })` |
| `fc.assert(prop, { seed: 42 })` | `hegel.test((tc) => { ... }, { seed: 42 })` |
| `fc.assert(prop, { endOnFailure: true })` | No equivalent — hegel always shrinks |
| `fc.assert(prop, { verbose: true })` | `hegel.test((tc) => { ... }, { verbosity: Verbosity.Verbose })` |
| Reproducing via `seed` + `path` | Use `seed` plus hegel's database replay (`.hegel/` directory) |

### Async Properties

fast-check:

```typescript
test("async fetch", () =>
  fc.assert(
    fc.asyncProperty(fc.integer({ min: 1 }), async (id) => {
      const user = await fetchUser(id);
      expect(user.id).toBe(id);
    }),
  ),
);
```

Hegel:

```typescript
test("async fetch", () =>
  hegel.testAsync(async (tc) => {
    const id = tc.draw(gs.integers({ minValue: 1 }));
    const user = await fetchUser(id);
    expect(user.id).toBe(id);
  }),
);
```

`hegel.test` rejects async callbacks with a `TypeError` — you must use `hegel.testAsync` for async bodies.

### Dependent Generation

fast-check (requires `chain`):

```typescript
fc.assert(
  fc.property(
    fc.array(fc.integer(), { minLength: 1 }).chain((arr) =>
      fc.tuple(fc.constant(arr), fc.integer({ min: 0, max: arr.length - 1 })),
    ),
    ([arr, idx]) => {
      expect(idx).toBeLessThan(arr.length);
    },
  ),
);
```

Hegel (just use sequential draws):

```typescript
hegel.test((tc) => {
  const arr = tc.draw(gs.arrays(gs.integers(), { minSize: 1 }));
  const idx = tc.draw(gs.integers({ minValue: 0, maxValue: arr.length - 1 }));
  expect(idx).toBeLessThan(arr.length);
});
```

This is one of hegel's main ergonomic advantages — dependent generation is just sequential code, no `chain`/`flatMap` gymnastics needed.

### Custom Arbitraries → composite / record

fast-check:

```typescript
const userArb = fc.record({
  name: fc.string({ minLength: 1 }),
  age: fc.integer({ min: 0, max: 120 }),
  active: fc.boolean(),
});
```

Hegel — declarative with `record`:

```typescript
const userGen = gs.record({
  name: gs.text({ minSize: 1 }),
  age: gs.integers({ minValue: 0, maxValue: 120 }),
  active: gs.booleans(),
});
```

For arbitraries with internal control flow (conditional fields, dependent draws), use `composite`:

```typescript
const personGen = gs.composite<Person>((tc) => {
  const age = tc.draw(gs.integers({ minValue: 0, maxValue: 120 }));
  const drivingLicense = age >= 18 ? tc.draw(gs.booleans()) : false;
  const name = tc.draw(gs.text({ minSize: 1 }));
  return { name, age, drivingLicense };
});
```

### Stateful Testing (`fc.commands`)

**Hegel-typescript does not yet support stateful testing.** If you're porting an `fc.commands` test, either keep that specific test on fast-check for now, or write the rule loop by hand inside `hegel.test`/`hegel.testAsync`:

```typescript
hegel.test((tc) => {
  const subject = new MyStack<number>();
  const model: number[] = [];
  const steps = tc.draw(gs.integers({ minValue: 1, maxValue: 50 }));
  for (let i = 0; i < steps; i++) {
    const op = tc.draw(gs.sampledFrom(["push", "pop"] as const));
    if (op === "push") {
      const v = tc.draw(gs.integers());
      subject.push(v);
      model.push(v);
    } else {
      expect(subject.pop()).toBe(model.pop());
    }
    expect(subject.size).toBe(model.length);
  }
});
```

When stateful testing lands in hegel-typescript, the reference will be updated with the proper API.

## From jsverify

[jsverify](https://github.com/jsverify/jsverify) is older and mostly unmaintained. The shape of the port is similar to fast-check:

- `jsc.forall(arb, (x) => true)` → `hegel.test((tc) => { const x = tc.draw(gen); ... })`
- `jsc.integer` / `jsc.nat` / `jsc.bool` / `jsc.string` / `jsc.array` map to `gs.integers` / `gs.integers({minValue:0})` / `gs.booleans` / `gs.text` / `gs.arrays`
- `arb.smap(f, g)` (with inverse) → `arb.map(f)` (no inverse needed; hegel handles shrinking)
- Custom arbitraries via `jsc.bless({ generator, shrink })` → `gs.composite(...)` (no shrink function needed)

## From testcheck-js

testcheck-js is a port of Clojure's test.check and is rare in modern codebases. Same pattern as fast-check porting:

- `check.property([gens...], (...) => true)` → `hegel.test((tc) => { ... })`
- `gen.int` / `gen.posInt` / `gen.array` / `gen.string` map to the obvious `gs.*` equivalents
- `gen.bind` → sequential `tc.draw()` calls

## Porting Checklist

When porting tests from fast-check (or any TypeScript PBT library):

1. **Add hegel and remove the old dependency** from `package.json` if no other tests use it: `npm install --save-dev @hegeldev/hegel`.
2. **Replace `fc.assert(fc.property(...))` with `hegel.test(...)`.** Wrap in `() => hegel.test(...)` to pass to the runner.
3. **Convert arbitraries to `tc.draw(gs.*)` calls.** Start with the broadest generators — don't carry over narrow bounds from the old test unless they're justified by the function's contract.
4. **Replace bool-return assertions with the runner's normal assertions.** `expect(...).toBe(...)` inside the body works directly.
5. **Replace `fc.pre(cond)` with `tc.assume(cond)`.**
6. **Convert async properties.** Use `hegel.testAsync` for async bodies — `hegel.test` will throw if given an async function.
7. **Simplify dependent generation.** If the old test used `chain` just to thread one value into the next, rewrite as sequential `tc.draw()` calls inside the body.
8. **Convert `fc.record(...)` to `gs.record(...)` and `fc.gen(...)` / custom arbitraries to `gs.composite(...)`.**
9. **Watch the `T | null` vs `T | undefined` difference for `optional` / `option`.**
10. **For `fc.commands`-style stateful tests, hand-roll the rule loop** (see above) until stateful testing lands in hegel-typescript.
11. **Run the tests.** If they fail on inputs the old framework didn't find, investigate — that's the point.

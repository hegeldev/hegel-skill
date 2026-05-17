# Hegel Go Reference

## Table of Contents

- [Setup](#setup)
- [Test Structure](#test-structure) — `hegel.Test`, HealthCheck, database
- [T vs TestCase](#t-vs-testcase)
- [Draw and TestCase Methods](#draw-and-testcase-methods) — `hegel.Draw`, `Assume`, `Note`, `Target`
- [Generator Reference](#generator-reference) — Numeric, boolean, text, characters, binary, collections, OneOf, optional, format, regex
- [Combinator Functions](#combinator-functions) — `Map`, `Filter`, `FlatMap`
- [Composite Generators](#composite-generators) — `hegel.Composite`
- [Stateful Testing](#stateful-testing) — `hegel.RunStateful`, rules, invariants
- [Workloads](#workloads) — `hegel.Workload` for standalone CLI binaries
- [Project Configuration](#project-configuration)
- [Go-Specific Examples](#go-specific-examples) — Dependent generation, wrapping arithmetic
- [Gotchas](#gotchas)

## Setup

Add to your module:

```bash
go get hegel.dev/go/hegel@latest
```

Run tests with `go test`. Hegel tests integrate directly with the standard Go test runner via `hegel.Test(t, ...)`.

If something goes wrong with server installation, see https://hegel.dev/reference/installation.

## Test Structure

### `hegel.Test` (preferred)

`Test` is the standard way to write hegel tests. It takes a `*testing.T`, a test function, and optional `Option` values:

```go
import (
    "math"
    "testing"

    "hegel.dev/go/hegel"
)

func TestAdditionCommutes(t *testing.T) {
    hegel.Test(t, func(ht *hegel.T) {
        a := hegel.Draw(ht, hegel.Integers(math.MinInt64, math.MaxInt64))
        b := hegel.Draw(ht, hegel.Integers(math.MinInt64, math.MaxInt64))
        if a+b != b+a {
            ht.Fatalf("not commutative: %d + %d", a, b)
        }
    })
}
```

With configuration:

```go
func TestWithConfig(t *testing.T) {
    hegel.Test(t, func(ht *hegel.T) {
        // ...
    }, hegel.WithTestCases(500))
}
```

Options:
- `hegel.WithTestCases(n int)` — Number of test cases (default: 100)
- `hegel.SuppressHealthCheck(checks ...hegel.HealthCheck)` — Suppress specific health checks
- `hegel.WithDatabase(db hegel.DatabaseSetting)` — Configure example-database persistence (see below)
- `hegel.WithDerandomize(b bool)` — Use a fixed seed for reproducible runs (default: `true` in CI)
- `hegel.WithSeed(seed int64)` — Pin a specific seed for reproducible runs

### HealthCheck

`HealthCheck` variants (the wire-protocol name in parentheses is what you'll see in failure messages):
- `hegel.FilterTooMuch` (`filter_too_much`) — Too many test cases rejected via `Assume()`
- `hegel.TooSlow` (`too_slow`) — Test execution is too slow
- `hegel.TestCasesTooLarge` (`test_cases_too_large`) — Generated test cases are too large
- `hegel.LargeInitialTestCase` (`large_initial_test_case`) — The smallest natural input is very large

```go
// Suppress a specific health check
hegel.Test(t, func(ht *hegel.T) {
    // ...
}, hegel.SuppressHealthCheck(hegel.FilterTooMuch))

// Suppress all health checks
hegel.Test(t, func(ht *hegel.T) {
    // ...
}, hegel.SuppressHealthCheck(hegel.AllHealthChecks()...))
```

### Example database

By default, hegel persists failing examples to a `.hegel/` directory in your project root and replays them on subsequent runs. In CI environments the database is automatically disabled.

To override the location or disable it explicitly, use `WithDatabase`:

```go
// Persist failing examples to a custom directory
hegel.Test(t, func(ht *hegel.T) { /* ... */ },
    hegel.WithDatabase(hegel.Database("my_hegel_database")))

// Disable example persistence entirely
hegel.Test(t, func(ht *hegel.T) { /* ... */ },
    hegel.WithDatabase(hegel.DatabaseDisabled()))
```

## T vs TestCase

Hegel provides two test context types:

- **`*hegel.T`** — Used with `hegel.Test`. Embeds the `hegel.TestCase` interface and wraps `*testing.T`, so you can use standard Go test methods (`ht.Fatal`, `ht.Error`, `ht.Log`, `ht.Skip`) and they work correctly with hegel's shrinking.
- **`hegel.TestCase`** — An interface used inside `hegel.Composite` and stateful-testing rules. Exposes hegel-specific methods (`Assume`, `Note`, `Target`). Signal failures via `panic`.

`*hegel.T` shadows these `testing.T` methods for hegel compatibility:

| Method | Behavior in hegel |
|--------|-------------------|
| `Fatal`, `Fatalf`, `FailNow` | Marks test case as INTERESTING (failing), triggers shrinking |
| `Error`, `Errorf`, `Fail` | Marks test case as failed but continues running |
| `Failed` | Reports whether the test case has been marked as failed |
| `Skip`, `Skipf`, `SkipNow` | Discards the current test case (calls `Assume(false)`) |
| `Log`, `Logf` | Routes through `Note` (only shown on final replay) |
| `Run` | Panics — nested sub-tests are not supported inside hegel tests |

`hegel.TestCase` also satisfies the `TestingT` interfaces used by popular assertion libraries (testify, gotest.tools, gomega), so assertions from those libraries work directly inside `Composite` callbacks, `hegel.Test` bodies, and stateful-testing rules.

## Draw and TestCase Methods

### `hegel.Draw`

```go
func Draw[T any](tc TestCase, g Generator[T]) T
```

`Draw` is a **top-level generic function**, not a method. It produces a value from a Generator using the given test context (`*hegel.T` or `hegel.TestCase`):

```go
n := hegel.Draw(ht, hegel.Integers(0, 100))
s := hegel.Draw(ht, hegel.Text().MinSize(0).MaxSize(50))
```

### TestCase Methods

`hegel.TestCase` is an interface; `*hegel.T` and the value passed to `hegel.Composite` callbacks both satisfy it.

| Method | Signature | Purpose |
|--------|-----------|---------|
| `Assume` | `Assume(condition bool)` | Reject this test case if condition is false |
| `Note` | `Note(message string)` | Print debug info (only on final counterexample replay) |
| `Target` | `Target(value float64, label string)` | Guide test generation toward maximizing a metric |

### Usage

```go
func TestDivision(t *testing.T) {
    hegel.Test(t, func(ht *hegel.T) {
        a := hegel.Draw(ht, hegel.Integers(math.MinInt64, math.MaxInt64))
        b := hegel.Draw(ht, hegel.Integers(math.MinInt64, math.MaxInt64))
        ht.Assume(b != 0)
        ht.Note(fmt.Sprintf("dividing %d by %d", a, b))
        q := a / b
        r := a % b
        if a != q*b+r {
            ht.Fatalf("division identity violated: %d != %d*%d + %d", a, q, b, r)
        }
    })
}
```

## Generator Reference

All generators are top-level functions in the `hegel` package.

### Numeric Generators

**`hegel.Integers[T](minVal, maxVal T)`** — Generate any integer type

Supported types: `int`, `int8`, `int16`, `int32`, `int64`, `uint`, `uint8`, `uint16`, `uint32`, `uint64`, `uintptr`.

Go infers the type parameter from the bounds, so the explicit `[T]` is usually unnecessary. Provide it when you want a non-`int` type and the bounds don't already pin it:

```go
n := hegel.Draw(ht, hegel.Integers(0, 100))
b := hegel.Draw(ht, hegel.Integers[uint8](1, 100))
big := hegel.Draw(ht, hegel.Integers(math.MinInt64, math.MaxInt64))
```

Min and max are **required constructor arguments**. For unbounded generation, use the full range of the type:

```go
hegel.Integers(math.MinInt, math.MaxInt)
hegel.Integers(math.MinInt32, math.MaxInt32)
hegel.Integers[uint64](0, math.MaxUint64)
```

**`hegel.Floats[T]()`** — Generate `float32` or `float64`

The type parameter is required (there are no bounds for inference):

```go
f := hegel.Draw(ht, hegel.Floats[float64]())
bounded := hegel.Draw(ht, hegel.Floats[float64]().Min(0).Max(1))
```

Builder methods:
- `.Min(T)` — Inclusive lower bound
- `.Max(T)` — Inclusive upper bound
- `.ExcludeMin()` — Make lower bound exclusive
- `.ExcludeMax()` — Make upper bound exclusive
- `.AllowNaN(bool)` — Default: `true` if unbounded, `false` if bounded
- `.AllowInfinity(bool)` — Default: `true` unless both bounds are set

### Boolean Generator

```go
b := hegel.Draw(ht, hegel.Booleans())
```

### Text and Binary Generators

**`hegel.Text()`** — Generate `string`. Configure with builder methods.

```go
s := hegel.Draw(ht, hegel.Text())                              // any size, full Unicode
bounded := hegel.Draw(ht, hegel.Text().MinSize(1).MaxSize(50))
ascii := hegel.Draw(ht, hegel.Text().Codec("ascii"))
abc := hegel.Draw(ht, hegel.Text().Alphabet("abc"))
```

Builder methods:
- `.MinSize(int)` — Minimum codepoint count (default: 0)
- `.MaxSize(int)` — Maximum codepoint count (no default — unbounded)
- `.Codec(string)` — Restrict to characters encodable in the given codec (e.g. `"ascii"`, `"utf-8"`, `"latin-1"`)
- `.MinCodepoint(rune)` / `.MaxCodepoint(rune)` — Restrict the Unicode codepoint range
- `.Categories([]string)` — Restrict to characters in the given Unicode general categories (e.g. `[]string{"L", "Nd"}`)
- `.ExcludeCategories([]string)` — Exclude characters in the given Unicode general categories
- `.IncludeCharacters(string)` — Always include these characters, even if excluded by other filters
- `.ExcludeCharacters(string)` — Always exclude these characters
- `.Alphabet(string)` — Restrict to only the given characters (mutually exclusive with the other character filters)

Go strings are UTF-8 and cannot represent surrogates, so the `Cs` Unicode category is always excluded automatically.

**`hegel.Characters()`** — Generate single-codepoint strings

Same character-filtering builder methods as `Text` (codec, codepoints, categories, etc.):

```go
c := hegel.Draw(ht, hegel.Characters())
ascii := hegel.Draw(ht, hegel.Characters().Codec("ascii"))
```

**`hegel.Binary(minSize, maxSize int)`** — Generate `[]byte`

```go
bytes := hegel.Draw(ht, hegel.Binary(0, 50))
unbounded := hegel.Draw(ht, hegel.Binary(0, -1))  // pass maxSize < 0 for unbounded
```

Pass `maxSize < 0` for unbounded.

### Constant and Choice Generators

```go
// Always returns the same value
x := hegel.Draw(ht, hegel.Just(42))

// Sample from a fixed set
suit := hegel.Draw(ht, hegel.SampledFrom([]string{"hearts", "diamonds", "clubs", "spades"}))
```

### Collection Generators

**`hegel.Lists(elements)`** — Generate `[]T`

```go
v := hegel.Draw(ht, hegel.Lists(hegel.Integers(math.MinInt, math.MaxInt)))
bounded := hegel.Draw(ht, hegel.Lists(hegel.Integers(math.MinInt, math.MaxInt)).MinSize(1).MaxSize(10))
```

Builder methods:
- `.MinSize(int)` — Minimum length (default: 0)
- `.MaxSize(int)` — Maximum length

**`hegel.Maps(keys, values)`** — Generate `map[K]V`

```go
m := hegel.Draw(ht, hegel.Maps(
    hegel.Text().MinSize(0).MaxSize(10),
    hegel.Integers(math.MinInt, math.MaxInt),
).MaxSize(5))
```

Builder methods:
- `.MinSize(int)` — Minimum number of entries (default: 0)
- `.MaxSize(int)` — Maximum number of entries

### OneOf

Choose between multiple generators of the same type:

```go
n := hegel.Draw(ht, hegel.OneOf(
    hegel.Just(0),
    hegel.Integers(1, 100),
    hegel.Integers(-100, -1),
))
```

All generators must return the same type.

### Optional

```go
maybe := hegel.Draw(ht, hegel.Optional(hegel.Integers(0, 100)))
// maybe is *int — nil or a pointer to the value
if maybe != nil {
    fmt.Println(*maybe)
}
```

**Important:** `Optional` returns `*T` (pointer), not an option type. Check for `nil` to distinguish "absent" from "present".

### Format Generators

```go
email := hegel.Draw(ht, hegel.Emails())
url := hegel.Draw(ht, hegel.URLs())
domain := hegel.Draw(ht, hegel.Domains().MaxLength(50))
```

**`hegel.Dates()`** and **`hegel.Datetimes()`** return `time.Time`:

```go
import "time"

date := hegel.Draw(ht, hegel.Dates())       // time.Time (date only)
dt := hegel.Draw(ht, hegel.Datetimes())     // time.Time (date + time)
```

**`hegel.IPAddresses()`** returns `netip.Addr`:

```go
import "net/netip"

ip := hegel.Draw(ht, hegel.IPAddresses())           // IPv4 or IPv6
ipv4 := hegel.Draw(ht, hegel.IPAddresses().IPv4())  // IPv4 only
ipv6 := hegel.Draw(ht, hegel.IPAddresses().IPv6())  // IPv6 only
```

### Regex Generator

```go
code := hegel.Draw(ht, hegel.FromRegex(`[A-Z]{3}-[0-9]{3}`, true))
```

The second argument (`fullmatch`) controls whether the pattern must match the entire string.

## Combinator Functions

In Go, combinators are **top-level generic functions**, not methods on Generator:

```go
import "hegel.dev/go/hegel"
```

### `hegel.Map`

Transform generated values:

```go
positiveStr := hegel.Draw(ht, hegel.Map(
    hegel.Integers[uint32](1, math.MaxUint32),
    func(n uint32) string { return fmt.Sprintf("%d", n) },
))
```

### `hegel.Filter`

Keep only values matching a predicate:

```go
even := hegel.Draw(ht, hegel.Filter(
    hegel.Integers(math.MinInt, math.MaxInt),
    func(x int) bool { return x%2 == 0 },
))
```

Note: `Filter` retries up to 3 times, then calls `Assume(false)`. Prefer bounds or `Map` over filters when possible.

### `hegel.FlatMap`

Dependent generation — use one value to choose the next generator:

```go
pair := hegel.Draw(ht, hegel.FlatMap(
    hegel.Integers(1, 5),
    func(n int) hegel.Generator[[]int] {
        return hegel.Lists(hegel.Integers(math.MinInt, math.MaxInt)).MinSize(n).MaxSize(n)
    },
))
```

In most cases, prefer sequential `hegel.Draw` calls over `FlatMap` — they read more naturally and produce the same shrinking behavior. Use `FlatMap` only when you need the result as a packaged `hegel.Generator[U]` (e.g. to pass to another combinator).

## Composite Generators

`hegel.Composite` packages an imperative draw sequence into a reusable `Generator[T]`. The function receives a `hegel.TestCase` (the same interface test bodies satisfy) and may call `hegel.Draw` any number of times — including conditionally, in loops, or recursively.

```go
type Person struct {
    Name           string
    Age            int
    DrivingLicense bool
}

personGen := hegel.Composite(func(tc hegel.TestCase) Person {
    age := hegel.Draw(tc, hegel.Integers(0, 120))
    name := hegel.Draw(tc, hegel.Text())
    p := Person{Age: age, Name: name}
    if age >= 18 {
        p.DrivingLicense = hegel.Draw(tc, hegel.Booleans())
    }
    return p
})

func TestPerson(t *testing.T) {
    hegel.Test(t, func(ht *hegel.T) {
        p := hegel.Draw(ht, personGen)
        // assert properties of p
    })
}
```

You can also draw the fields inline in the test body without wrapping them in `Composite` — both styles are idiomatic. Reach for `Composite` when you want the generator to be reusable across tests, named for clarity, or passed to combinators like `Lists` or `Optional`.

`Composite` supports recursive generators. Pass the recursion budget as a parameter so each call has its own depth — this is safer than a shared mutable counter, which can leak depth across test cases if a draw panics:

```go
type Node struct {
    Value int
    Left  *Node
    Right *Node
}

var nodeGen func(depth int) hegel.Generator[*Node]
nodeGen = func(depth int) hegel.Generator[*Node] {
    return hegel.Composite(func(tc hegel.TestCase) *Node {
        n := &Node{Value: hegel.Draw(tc, hegel.Integers(0, 100))}
        if depth > 0 && hegel.Draw(tc, hegel.Booleans()) {
            n.Left = hegel.Draw(tc, nodeGen(depth-1))
            n.Right = hegel.Draw(tc, nodeGen(depth-1))
        }
        return n
    })
}

func TestTree(t *testing.T) {
    hegel.Test(t, func(ht *hegel.T) {
        _ = hegel.Draw(ht, nodeGen(5))
    })
}
```

## Stateful Testing

`hegel.RunStateful` enables model-based testing. Define a struct whose methods follow a naming convention: methods prefixed with `Rule` are actions hegel can apply, and methods prefixed with `Invariant` are checked after every successful rule. Both kinds of method take a single `hegel.TestCase` argument.

```go
type stateCounter struct{ n int }

// RuleIncrement bumps the counter up by one.
func (c *stateCounter) RuleIncrement(_ hegel.TestCase) { c.n++ }

// RuleDecrement bumps the counter down by one.
func (c *stateCounter) RuleDecrement(_ hegel.TestCase) { c.n-- }

// InvariantSensible panics if the counter has drifted out of range.
func (c *stateCounter) InvariantSensible(_ hegel.TestCase) {
    if c.n < -10000 || c.n > 10000 {
        panic("counter out of sensible range")
    }
}

func TestCounter(t *testing.T) {
    hegel.Test(t, func(ht *hegel.T) {
        c := &stateCounter{}                // fresh machine per test case
        hegel.RunStateful(ht, c)
    })
}
```

Notes:
- Pass a **pointer** to your machine to `RunStateful` so rules can mutate it.
- Inside a rule, call `tc.Assume(false)` to skip when the rule doesn't apply — hegel will try a different rule.
- `RunStateful` panics if the machine has no `Rule*` methods or if any `Rule*`/`Invariant*` method has the wrong signature.

**Shared external state.** If the machine wraps something you can't cheaply re-create per test case (a database, a temp directory, a long-lived network connection), keep that resource hoisted outside the test body and reset/clean it inside the body before calling `RunStateful`. Otherwise — for in-memory models, which is the common case — just allocate fresh.

For models that need to track dynamically created resources (handles, IDs, accounts), generate them inside a rule and store them on the machine; pull from that store in other rules. Use `Draw(tc, hegel.SampledFrom(c.handles))` (with `tc.Assume(len(c.handles) > 0)` first) to act on existing resources.

## Workloads

`hegel.Workload` runs a property test as a standalone CLI binary — useful for soak tests, fuzzing harnesses, or long-running workloads outside of `go test`. It parses standard hegel flags (`-test-cases`, `-seed`, `-verbosity`, etc.) and exits non-zero on failure.

```go
package main

import "hegel.dev/go/hegel"

func main() {
    hegel.Workload(func(tc hegel.TestCase) {
        n := hegel.Draw(tc, hegel.Integers(0, 100))
        _ = n
    })
}
```

`Workload` takes the same `Option` values as `hegel.Test` (e.g. `hegel.WithTestCases(...)`). CLI flags override `Option` values, which override defaults.

## Project Configuration

### `hegel.SetHegelDirectory`

Override the automatically detected `.hegel` data directory. Hegel walks up from the working directory looking for `go.mod`, `.git`, `go.sum`, `Makefile`, or `justfile`/`Justfile` to identify the project root. Call `SetHegelDirectory` before any hegel tests run (e.g. in `TestMain`) when auto-detection isn't suitable:

```go
func TestMain(m *testing.M) {
    hegel.SetHegelDirectory("/path/to/project/.hegel")
    os.Exit(m.Run())
}
```

## Go-Specific Examples

These examples show Go-specific idioms. For general property patterns (round-trip, model-based, idempotence, etc.), see the main skill's Property Catalogue.

### Dependent generation with sequential draws

Hegel's imperative style means dependent generation is just sequential code:

```go
func TestValidIndex(t *testing.T) {
    hegel.Test(t, func(ht *hegel.T) {
        v := hegel.Draw(ht, hegel.Lists(hegel.Integers(math.MinInt, math.MaxInt)).MinSize(1))
        idx := hegel.Draw(ht, hegel.Integers(0, len(v)-1))
        // idx is always a valid index
        _ = v[idx]
    })
}
```

### Avoiding silent overflow in test values

Go's integer arithmetic wraps silently on overflow — it doesn't panic. That's a hazard in *test* code: an expression like `m[k] = k * 10` doesn't crash near `math.MaxInt`, it just stores a wrapped negative value, and your test ends up exercising a meaningless input rather than the value you thought you were generating.

To keep generated data inside a range your test arithmetic can handle, draw a smaller type and widen:

```go
// Risky — k * 10 silently wraps for k near math.MaxInt, producing a useless input
k := hegel.Draw(ht, hegel.Integers(math.MinInt, math.MaxInt))
m[k*10] = k

// Better — draw a smaller type so widening into int can't overflow the arithmetic
k16 := hegel.Draw(ht, hegel.Integers[int16](math.MinInt16, math.MaxInt16))
k := int(k16)
m[k*10] = k
```

Distinguish "this constraint protects the library's contract" (keep it) from "this constraint prevents my test arithmetic from wrapping" (use a narrower draw).

## Gotchas

1. **Type parameters are inferred for `Integers` from its arguments.** `hegel.Integers(0, 100)` compiles as `Integers[int]`. Use explicit `[T]` when you need a non-`int` type and the bounds don't already pin it (e.g. `hegel.Integers[uint8](1, 100)`).

2. **`Floats` requires an explicit type parameter.** It has no arguments to infer from, so write `hegel.Floats[float64]()` (or `[float32]`).

3. **Integers min/max are required arguments.** Unlike Hegel-rust's builder pattern, Go's `Integers` takes min and max as constructor args. For unbounded, use `math.MinInt`/`math.MaxInt` (or the type-specific constants).

4. **Text uses a builder; Binary takes constructor args with -1 for unbounded.** `hegel.Text().MinSize(0).MaxSize(50)` for strings; `hegel.Binary(0, -1)` for unbounded byte slices.

5. **Combinators are free functions, not methods.** Write `hegel.Map(gen, fn)`, not `gen.Map(fn)`. This is because Go's type system does not allow methods on interface types with different return type parameters.

6. **Optional returns `*T`, not an option type.** Go has no `Option[T]`, so `hegel.Optional(gen)` returns `Generator[*T]`. Check the result for `nil`.

7. **Dates/Datetimes return `time.Time`.** Not strings. Import `"time"` to work with the results.

8. **IPAddresses returns `netip.Addr`.** Not strings. Import `"net/netip"` to work with the results.

9. **`t.Run()` is not supported inside hegel tests.** Calling `ht.Run()` will panic. Nested sub-tests cannot work with hegel's shrinking model.

10. **Float defaults include NaN and infinity.** `hegel.Floats[float64]()` with no bounds generates NaN and infinity by default. If your code doesn't handle these, use `.AllowNaN(false)` and/or `.AllowInfinity(false)` — but consider whether the code *should* handle them first.

11. **Excessive assume/filter rejections fail the test.** If `Assume()` or `Filter` rejects too many inputs, hegel gives up. Restructure your generators to produce valid inputs directly.

12. **`Note` only prints on the final replay.** Don't rely on `Note` for progress logging — it only appears when displaying the minimal counterexample. Notes route through `t.Log`.

13. **Default collection sizes are small.** `hegel.Lists(gen)` with no bounds rarely produces 100+ elements. If you need large collections (e.g., to test tree traversal at depth), draw the size separately:
    ```go
    n := hegel.Draw(ht, hegel.Integers(0, 300))
    items := hegel.Draw(ht, hegel.Lists(hegel.Integers(math.MinInt, math.MaxInt)).MinSize(n))
    ```

14. **Add `.hegel/` to `.gitignore`.** Hegel creates a `.hegel/` directory at your project root for caching the server binary, the example database, and per-process server logs.

15. **In CI, the example database is disabled by default.** Hegel detects common CI environment variables (`CI`, `GITHUB_ACTIONS`, `BUILDKITE`, etc.) and skips persistence. Override this with `hegel.WithDatabase(hegel.Database("..."))` if you want CI runs to share a database.

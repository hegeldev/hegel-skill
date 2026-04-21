# Hegel Go Reference

## Table of Contents

- [Setup](#setup)
- [Test Structure](#test-structure) — `hegel.Case`, `hegel.Run`, `hegel.MustRun`, HealthCheck
- [T vs TestCase](#t-vs-testcase)
- [Draw and TestCase Methods](#draw-and-testcase-methods) — `hegel.Draw`, `Assume`, `Note`, `Target`
- [Generator Reference](#generator-reference) — Numeric, boolean, text, binary, collections, OneOf, optional, format, regex
- [Combinator Functions](#combinator-functions) — `Map`, `Filter`, `FlatMap`
- [Composite Generators](#composite-generators)
- [Project Configuration](#project-configuration)
- [Go-Specific Examples](#go-specific-examples) — Dependent generation, wrapping arithmetic
- [Gotchas](#gotchas)

## Setup

Add to your module:

```bash
go get hegel.dev/go/hegel@latest
```

Run tests with `go test`. Hegel tests integrate directly with the standard Go test runner via `t.Run`.

If something goes wrong with server installation, see https://hegel.dev/reference/installation.

## Test Structure

### `hegel.Case` (preferred)

`Case` returns a `func(*testing.T)` for use with `t.Run`. This is the standard way to write hegel tests in Go:

```go
import (
    "math"
    "testing"

    "hegel.dev/go/hegel"
)

func TestAdditionCommutes(t *testing.T) {
    t.Run("addition commutes", hegel.Case(func(ht *hegel.T) {
        a := hegel.Draw(ht, hegel.Integers[int64](math.MinInt64, math.MaxInt64))
        b := hegel.Draw(ht, hegel.Integers[int64](math.MinInt64, math.MaxInt64))
        if a+b != b+a {
            ht.Fatalf("not commutative: %d + %d", a, b)
        }
    }))
}
```

With configuration:

```go
func TestWithConfig(t *testing.T) {
    t.Run("with config", hegel.Case(func(ht *hegel.T) {
        // ...
    }, hegel.WithTestCases(500)))
}
```

Options:
- `hegel.WithTestCases(n int)` -- Number of test cases (default: 100)
- `hegel.SuppressHealthCheck(checks ...hegel.HealthCheck)` -- Suppress specific health checks

### `hegel.Run` (non-testing.T contexts)

`Run` executes a property test and returns an error. Note output goes to stderr. For use in standalone binaries.

```go
err := hegel.Run(func(tc *hegel.TestCase) {
    n := hegel.Draw(tc, hegel.Integers[int32](math.MinInt32, math.MaxInt32))
    if n != n {
        panic("integer not equal to itself")
    }
})
```

### `hegel.MustRun`

Like `Run`, but panics on failure:

```go
hegel.MustRun(func(tc *hegel.TestCase) {
    n := hegel.Draw(tc, hegel.Integers[int32](math.MinInt32, math.MaxInt32))
    if n != n {
        panic("integer not equal to itself")
    }
})
```

### HealthCheck

`HealthCheck` variants:
- `hegel.FilterTooMuch` -- Too many test cases rejected via `Assume()`
- `hegel.TooSlow` -- Test execution is too slow
- `hegel.TestCasesTooLarge` -- Generated test cases are too large
- `hegel.LargeInitialTestCase` -- The smallest natural input is very large

```go
// Suppress a specific health check
t.Run("filtered", hegel.Case(func(ht *hegel.T) {
    // ...
}, hegel.SuppressHealthCheck(hegel.FilterTooMuch)))

// Suppress all health checks
t.Run("all suppressed", hegel.Case(func(ht *hegel.T) {
    // ...
}, hegel.SuppressHealthCheck(hegel.AllHealthChecks()...)))
```

## T vs TestCase

Hegel provides two test context types:

- **`*hegel.T`** -- Used with `hegel.Case`. Embeds both `*hegel.TestCase` and `*testing.T`, so you can use standard Go test methods (`ht.Fatal`, `ht.Error`, `ht.Log`, `ht.Skip`) and they work correctly with hegel's shrinking.
- **`*hegel.TestCase`** -- Used with `hegel.Run` and `hegel.MustRun`. Only has hegel-specific methods (`Assume`, `Note`, `Target`). Signal failures via `panic`.

`*hegel.T` shadows these `testing.T` methods for hegel compatibility:

| Method | Behavior in hegel |
|--------|-------------------|
| `Fatal`, `Fatalf`, `FailNow` | Marks test case as INTERESTING (failing), triggers shrinking |
| `Error`, `Errorf`, `Fail` | Marks test case as failed but continues running |
| `Failed` | Reports whether the test case has been marked as failed |
| `Skip`, `Skipf`, `SkipNow` | Discards the current test case (calls `Assume(false)`) |
| `Log`, `Logf` | Routes through `Note` (only shown on final replay) |
| `Run` | Panics -- nested sub-tests are not supported inside hegel tests |

## Draw and TestCase Methods

### `hegel.Draw`

```go
func Draw[T any](tc testCase, g Generator[T]) T
```

`Draw` is a **top-level generic function**, not a method. It produces a value from a Generator using the given test context (`*hegel.T` or `*hegel.TestCase`):

```go
n := hegel.Draw(ht, hegel.Integers[int](0, 100))
s := hegel.Draw(ht, hegel.Text(0, 50))
```

### TestCase Methods

| Method | Signature | Purpose |
|--------|-----------|---------|
| `Assume` | `func (s *TestCase) Assume(condition bool)` | Reject this test case if condition is false |
| `Note` | `func (s *TestCase) Note(message string)` | Print debug info (only on final counterexample replay) |
| `Target` | `func (s *TestCase) Target(value float64, label string)` | Guide test generation toward maximizing a metric |

### Usage

```go
func TestDivision(t *testing.T) {
    t.Run("division", hegel.Case(func(ht *hegel.T) {
        a := hegel.Draw(ht, hegel.Integers[int64](math.MinInt64, math.MaxInt64))
        b := hegel.Draw(ht, hegel.Integers[int64](math.MinInt64, math.MaxInt64))
        ht.Assume(b != 0)
        ht.Note(fmt.Sprintf("dividing %d by %d", a, b))
        q := a / b
        r := a % b
        if a != q*b+r {
            ht.Fatalf("division identity violated: %d != %d*%d + %d", a, q, b, r)
        }
    }))
}
```

## Generator Reference

All generators are top-level functions in the `hegel` package.

### Numeric Generators

**`hegel.Integers[T](minVal, maxVal T)`** -- Generate any integer type

Supported types: `int`, `int8`, `int16`, `int32`, `int64`, `uint`, `uint8`, `uint16`, `uint32`, `uint64`, `uintptr`.

```go
n := hegel.Draw(ht, hegel.Integers[int](math.MinInt, math.MaxInt))
bounded := hegel.Draw(ht, hegel.Integers[uint8](1, 100))
```

Unlike Rust, min and max are **required constructor arguments**. For unbounded generation, use the full range of the type:

```go
hegel.Integers[int](math.MinInt, math.MaxInt)
hegel.Integers[int32](math.MinInt32, math.MaxInt32)
hegel.Integers[uint64](0, math.MaxUint64)
```

**`hegel.Floats[T]()`** -- Generate `float32` or `float64`

Uses a builder pattern for configuration:

```go
f := hegel.Draw(ht, hegel.Floats[float64]())
bounded := hegel.Draw(ht, hegel.Floats[float64]().Min(0).Max(1))
```

Builder methods:
- `.Min(T)` -- Inclusive lower bound
- `.Max(T)` -- Inclusive upper bound
- `.ExcludeMin()` -- Make lower bound exclusive
- `.ExcludeMax()` -- Make upper bound exclusive
- `.AllowNaN(bool)` -- Default: `true` if unbounded, `false` if bounded
- `.AllowInfinity(bool)` -- Default: `true` unless both bounds are set

### Boolean Generator

```go
b := hegel.Draw(ht, hegel.Booleans())
```

### Text and Binary Generators

**`hegel.Text(minSize, maxSize int)`** -- Generate `string`

```go
s := hegel.Draw(ht, hegel.Text(0, 100))
unbounded := hegel.Draw(ht, hegel.Text(0, -1))  // pass maxSize < 0 for unbounded
```

**`hegel.Binary(minSize, maxSize int)`** -- Generate `[]byte`

```go
bytes := hegel.Draw(ht, hegel.Binary(0, 50))
unbounded := hegel.Draw(ht, hegel.Binary(0, -1))
```

Both take min/max as constructor arguments. Pass `maxSize < 0` for unbounded.

### Constant and Choice Generators

```go
// Always returns the same value
x := hegel.Draw(ht, hegel.Just(42))

// Sample from a fixed set
suit := hegel.Draw(ht, hegel.SampledFrom([]string{"hearts", "diamonds", "clubs", "spades"}))
```

### Collection Generators

**`hegel.Lists(elements)`** -- Generate `[]T`

```go
v := hegel.Draw(ht, hegel.Lists(hegel.Integers[int](math.MinInt, math.MaxInt)))
bounded := hegel.Draw(ht, hegel.Lists(hegel.Integers[int](math.MinInt, math.MaxInt)).MinSize(1).MaxSize(10))
```

Builder methods:
- `.MinSize(int)` -- Minimum length (default: 0)
- `.MaxSize(int)` -- Maximum length

**`hegel.Dicts(keys, values)`** -- Generate `map[K]V`

```go
m := hegel.Draw(ht, hegel.Dicts(
    hegel.Text(0, 10),
    hegel.Integers[int](math.MinInt, math.MaxInt),
).MaxSize(5))
```

Builder methods:
- `.MinSize(int)` -- Minimum number of entries (default: 0)
- `.MaxSize(int)` -- Maximum number of entries

### OneOf

Choose between multiple generators of the same type:

```go
n := hegel.Draw(ht, hegel.OneOf(
    hegel.Just(0),
    hegel.Integers[int](1, 100),
    hegel.Integers[int](-100, -1),
))
```

All generators must return the same type.

### Optional

```go
maybe := hegel.Draw(ht, hegel.Optional(hegel.Integers[int](0, 100)))
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
dt := hegel.Draw(ht, hegel.Datetimes())      // time.Time (date + time)
```

**`hegel.IPAddresses()`** returns `netip.Addr`:

```go
import "net/netip"

ip := hegel.Draw(ht, hegel.IPAddresses())          // IPv4 or IPv6
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
    hegel.Integers[int](math.MinInt, math.MaxInt),
    func(x int) bool { return x%2 == 0 },
))
```

Note: `Filter` retries up to 3 times, then calls `Assume(false)`. Prefer bounds or `Map` over filters when possible.

### `hegel.FlatMap`

Dependent generation -- use one value to choose the next generator:

```go
pair := hegel.Draw(ht, hegel.FlatMap(
    hegel.Integers[int](1, 5),
    func(n int) hegel.Generator[[]int] {
        return hegel.Lists(hegel.Integers[int](math.MinInt, math.MaxInt)).MinSize(n).MaxSize(n)
    },
))
```

## Composite Generators

Go does not have macros, so there is no `#[hegel::composite]` or `compose!` equivalent. Instead, write a plain function that returns a `Generator`:

```go
func points(maxCoord float64) hegel.Generator[[2]float64] {
    return hegel.Map(
        hegel.Floats[float64]().Min(-maxCoord).Max(maxCoord),
        func(x float64) [2]float64 {
            // Note: this only generates one coordinate from hegel.
            // For two independent coordinates, use FlatMap or draw separately.
            return [2]float64{x, x}
        },
    )
}
```

For generators that need multiple draws, draw them separately in the test body:

```go
func TestPoints(t *testing.T) {
    t.Run("points", hegel.Case(func(ht *hegel.T) {
        x := hegel.Draw(ht, hegel.Floats[float64]().Min(-100).Max(100))
        y := hegel.Draw(ht, hegel.Floats[float64]().Min(-100).Max(100))
        // use x, y
    }))
}
```

This is idiomatic Go -- sequential `Draw` calls replace the need for composite generator macros.

## Project Configuration

### `hegel.SetHegelDirectory`

Override the automatically detected `.hegel` data directory. Call before any hegel tests run (e.g., in `TestMain`):

```go
func TestMain(m *testing.M) {
    hegel.SetHegelDirectory("/path/to/project/.hegel")
    os.Exit(m.Run(nil))
}
```

## Go-Specific Examples

These examples show Go-specific idioms. For general property patterns (round-trip, model-based, idempotence, etc.), see the main skill's Property Catalogue.

### Dependent generation with sequential draws

Hegel's imperative style means dependent generation is just sequential code:

```go
func TestValidIndex(t *testing.T) {
    t.Run("valid index", hegel.Case(func(ht *hegel.T) {
        v := hegel.Draw(ht, hegel.Lists(hegel.Integers[int](math.MinInt, math.MaxInt)).MinSize(1))
        idx := hegel.Draw(ht, hegel.Integers[int](0, len(v)-1))
        // idx is always a valid index
        _ = v[idx]
    }))
}
```

### Wrapping arithmetic in test values

Go does not have wrapping arithmetic operators. To avoid overflow panics in *test* code, draw a smaller type and widen:

```go
// BAD -- panics when k is near math.MaxInt
m[k] = k * 10

// GOOD -- draw a smaller type and widen to prevent overflow
k16 := hegel.Draw(ht, hegel.Integers[int16](math.MinInt16, math.MaxInt16))
k := int(k16)
m[k] = k * k  // can't overflow int
```

## Gotchas

1. **Type parameters are required for numeric generators.** `hegel.Integers(0, 100)` won't compile -- you must write `hegel.Integers[int](0, 100)` (or whatever type you need).

2. **Integers min/max are required arguments.** Unlike the Hegel-rust's builder pattern, Go's `Integers` takes min and max as constructor args. For unbounded, use `math.MinInt`/`math.MaxInt` (or the type-specific constants).

3. **Text/Binary use -1 for unbounded max.** `hegel.Text(0, -1)` means no upper bound. This is different from Rust where you omit `.max_size()`.

4. **Combinators are free functions, not methods.** Write `hegel.Map(gen, fn)`, not `gen.Map(fn)`. This is because Go's type system does not allow methods on interface types with different return type parameters.

5. **Optional returns `*T`, not an option type.** Go has no `Option<T>`, so `hegel.Optional(gen)` returns `Generator[*T]`. Check the result for `nil`.

6. **Dates/Datetimes return `time.Time`.** Not strings. Import `"time"` to work with the results.

7. **IPAddresses returns `netip.Addr`.** Not strings. Import `"net/netip"` to work with the results.

8. **`t.Run()` is not supported inside hegel tests.** Calling `ht.Run()` will panic. Nested sub-tests cannot work with hegel's shrinking model.

9. **Float defaults include NaN and infinity.** `hegel.Floats[float64]()` with no bounds generates NaN and infinity by default. If your code doesn't handle these, use `.AllowNaN(false)` and/or `.AllowInfinity(false)` -- but consider whether the code *should* handle them first.

10. **Excessive assume/filter rejections fail the test.** If `Assume()` or `Filter` rejects too many inputs, hegel gives up. Restructure your generators to produce valid inputs directly.

11. **`Note` only prints on the final replay.** Don't rely on `Note` for progress logging -- it only appears when displaying the minimal counterexample.

12. **Default collection sizes are small.** `hegel.Lists(gen)` with no bounds rarely produces 100+ elements. If you need large collections (e.g., to test tree traversal at depth), draw the size separately:
    ```go
    n := hegel.Draw(ht, hegel.Integers[int](0, 300))
    items := hegel.Draw(ht, hegel.Lists(hegel.Integers[int](math.MinInt, math.MaxInt)).MinSize(n))
    ```

13. **Add `.hegel/` to `.gitignore`.** Hegel creates a `.hegel/` directory for caching the server binary and storing the database of previous failures.

14. **Use `hegel.Case` with `t.Run`, not standalone.** The `Case` function returns a `func(*testing.T)` -- wrap it in `t.Run("name", ...)` to give the test a name and proper integration with `go test`.

# Hegel Go SDK Reference

## Setup

Install via `go get`:

```bash
go get -t hegel.dev/go/hegel@latest
```

Import in your test files:

```go
import "hegel.dev/go/hegel"
```

Requires Go 1.25+. Run tests with `go test`. Hegel tests use `hegel.Case()` with `t.Run()` and
integrate directly with Go's standard test runner.

## Test Structure

### `hegel.Case()` (preferred — for use with `go test`)

```go
import (
    "math"
    "testing"

    "hegel.dev/go/hegel"
)

func TestAdditionCommutes(t *testing.T) {
    t.Run("add_commutes", hegel.Case(func(ht *hegel.T) {
        a := hegel.Draw(ht, hegel.Integers(math.MinInt, math.MaxInt))
        b := hegel.Draw(ht, hegel.Integers(math.MinInt, math.MaxInt))
        if a+b != b+a {
            ht.Fatal("addition is not commutative")
        }
    }))
}
```

`hegel.Case()` returns a `func(*testing.T)` for use with `t.Run()`. The test
function receives `*hegel.T`, which embeds `*testing.T` and overrides
`Fatal`, `Fatalf`, `Error`, `Errorf`, `Fail`, `FailNow`, `Skip`, `Log`, etc.
to work correctly inside a hegel test body.

With options:

```go
t.Run("many_cases", hegel.Case(func(ht *hegel.T) {
    // ...
}, hegel.WithTestCases(500)))
```

### `hegel.Run()` (standalone — for non-test binaries)

```go
err := hegel.Run(func(s *hegel.TestCase) {
    n := hegel.Draw(s, hegel.Integers(0, 100))
    if n < 0 || n > 100 {
        panic("out of range")
    }
}, hegel.WithTestCases(50))
```

Returns an error if the test fails. `hegel.MustRun()` is a convenience
wrapper that panics on failure.

### Options

- `hegel.WithTestCases(n int)` — Number of test cases (default: 100)
- `hegel.SuppressHealthCheck(checks ...HealthCheck)` — Suppress specific
  health checks

### HealthCheck

```go
const (
    FilterTooMuch        // Too many test cases rejected via Assume
    TooSlow              // Test execution too slow
    TestCasesTooLarge     // Generated test cases too large
    LargeInitialTestCase // Smallest natural input is very large
)
```

Use `hegel.AllHealthChecks()` to get all variants.

## TestCase / T Methods

| Method | On | Purpose |
|--------|----|---------|
| `Assume(condition bool)` | `TestCase`, `T` | Reject this test case if condition is false |
| `Note(message string)` | `TestCase`, `T` | Print debug info (only on final counterexample replay) |
| `Target(value float64, label string)` | `TestCase`, `T` | Guide generation toward maximizing a metric |
| `Fatal(args ...any)` | `T` | Log via Note and fail the test case |
| `Fatalf(format string, args ...any)` | `T` | Log formatted via Note and fail |
| `Error(args ...any)` | `T` | Log via Note and set failed flag (test continues) |
| `Errorf(format string, args ...any)` | `T` | Log formatted via Note and set failed flag |
| `Fail()` | `T` | Set failed flag without stopping |
| `FailNow()` | `T` | Fail and stop immediately |
| `Failed() bool` | `T` | Check if test case has been marked failed |
| `Log(args ...any)` | `T` | Route through Note (final replay only) |
| `Logf(format string, args ...any)` | `T` | Route formatted through Note |
| `Skip(args ...any)` | `T` | Discard current test case (calls Assume(false)) |

**Note:** `t.Run()` inside a hegel test body is **not supported** and will
panic. Use separate top-level `t.Run()` calls for separate hegel tests.

## Drawing Values

Use the top-level generic function `hegel.Draw()`:

```go
n := hegel.Draw(ht, hegel.Integers(0, 100))
s := hegel.Draw(ht, hegel.Text(0, 50))
items := hegel.Draw(ht, hegel.Lists(hegel.Integers(0, 10)))
```

`Draw` works with both `*hegel.T` (from `Case`) and `*hegel.TestCase` (from
`Run`).

## Generator Reference

### Numeric Generators

**`hegel.Integers[T](minVal, maxVal T)`** — Generate any integer type

Supported types: `int`, `int8`, `int16`, `int32`, `int64`, `uint`, `uint8`,
`uint16`, `uint32`, `uint64`.

Both bounds are required. For unbounded generation, use the full type range:

```go
import "math"

n := hegel.Draw(ht, hegel.Integers(math.MinInt, math.MaxInt))
bounded := hegel.Draw(ht, hegel.Integers[uint8](1, 100))
```

**`hegel.Floats[T]()`** — Generate `float32` or `float64`

```go
f := hegel.Draw(ht, hegel.Floats[float64]())
bounded := hegel.Draw(ht, hegel.Floats[float64]().Min(0).Max(1))
```

Builder methods:
- `.Min(T)` — Lower bound (inclusive by default)
- `.Max(T)` — Upper bound (inclusive by default)
- `.ExcludeMin()` — Make lower bound exclusive
- `.ExcludeMax()` — Make upper bound exclusive
- `.AllowNaN(bool)` — Default: `true` if unbounded, `false` if bounded
- `.AllowInfinity(bool)` — Default: `true` unless both bounds set

### Boolean Generator

```go
b := hegel.Draw(ht, hegel.Booleans())
```

### Text and Binary Generators

**`hegel.Text(minSize, maxSize int)`** — Generate `string`

Both bounds are required. Pass `maxSize < 0` for unbounded.

```go
s := hegel.Draw(ht, hegel.Text(0, 50))
unbounded := hegel.Draw(ht, hegel.Text(0, -1))
```

**`hegel.Binary(minSize, maxSize int)`** — Generate `[]byte`

```go
bytes := hegel.Draw(ht, hegel.Binary(0, 100))
unbounded := hegel.Draw(ht, hegel.Binary(0, -1))
```

### Constant and Choice Generators

```go
// Always returns the same value
x := hegel.Draw(ht, hegel.Just(42))

// Sample uniformly from a fixed set
suit := hegel.Draw(ht, hegel.SampledFrom([]string{
    "hearts", "diamonds", "clubs", "spades",
}))
```

### Collection Generators

**`hegel.Lists[T](elements)`** — Generate `[]T`

```go
v := hegel.Draw(ht, hegel.Lists(hegel.Integers(0, 100)))
bounded := hegel.Draw(ht, hegel.Lists(hegel.Integers(0, 100)).
    MinSize(1).MaxSize(10))
```

Builder methods:
- `.MinSize(int)` — Minimum length (default: 0)
- `.MaxSize(int)` — Maximum length

**`hegel.Dicts[K, V](keys, values)`** — Generate `map[K]V`

```go
m := hegel.Draw(ht, hegel.Dicts(
    hegel.Text(1, 10),
    hegel.Integers(0, 100),
).MaxSize(5))
```

Builder methods:
- `.MinSize(int)` — Minimum entries (default: 0)
- `.MaxSize(int)` — Maximum entries

### Optional Generator

```go
maybe := hegel.Draw(ht, hegel.Optional(hegel.Integers(0, 100)))
// maybe is *int — nil or pointer to value
if maybe != nil {
    fmt.Println(*maybe)
}
```

### Format Generators

```go
email := hegel.Draw(ht, hegel.Emails())
url := hegel.Draw(ht, hegel.URLs())
domain := hegel.Draw(ht, hegel.Domains().MaxLength(50))
```

**`hegel.Dates()`** — Generate `time.Time` (date only, YYYY-MM-DD)

**`hegel.Datetimes()`** — Generate `time.Time` (YYYY-MM-DDTHH:MM:SS)

```go
import "time"

date := hegel.Draw(ht, hegel.Dates())
dt := hegel.Draw(ht, hegel.Datetimes())
```

**`hegel.IPAddresses()`** — Generate `netip.Addr`

```go
import "net/netip"

ip := hegel.Draw(ht, hegel.IPAddresses())
ipv4 := hegel.Draw(ht, hegel.IPAddresses().IPv4())
ipv6 := hegel.Draw(ht, hegel.IPAddresses().IPv6())
```

### Regex Generator

```go
code := hegel.Draw(ht, hegel.FromRegex(`[A-Z]{3}-[0-9]{3}`, true))
```

Parameters:
- `pattern string` — Regular expression
- `fullmatch bool` — `true` to match entire string, `false` for substring

## Combinator Functions

### `hegel.Map(g, fn)`

Transform generated values:

```go
positiveStr := hegel.Draw(ht, hegel.Map(
    hegel.Integers[uint32](1, 1000),
    func(n uint32) string { return fmt.Sprintf("%d", n) },
))
```

### `hegel.Filter(g, pred)`

Keep only values matching a predicate. Retries up to 3 times per test case,
then rejects via `Assume(false)`.

```go
even := hegel.Draw(ht, hegel.Filter(
    hegel.Integers(math.MinInt, math.MaxInt),
    func(n int) bool { return n%2 == 0 },
))
```

Prefer `Map` over `Filter` when possible (e.g. `Map(Integers(0, 50), func(n
int) int { return n * 2 })` is better than filtering for even numbers).

### `hegel.FlatMap(g, fn)`

Dependent generation — use one value to choose the next generator:

```go
result := hegel.Draw(ht, hegel.FlatMap(
    hegel.Integers(1, 5),
    func(n int) hegel.Generator[[]int] {
        return hegel.Lists(hegel.Integers(math.MinInt, math.MaxInt)).
            MinSize(n).MaxSize(n)
    },
))
```

### `hegel.OneOf(generators...)`

Choose between multiple generators of the same type:

```go
n := hegel.Draw(ht, hegel.OneOf(
    hegel.Just(0),
    hegel.Integers(1, 100),
    hegel.Integers(-100, -1),
))
```

## Idiomatic Patterns

### Round-trip (serialize/deserialize)

```go
func TestJSONRoundTrip(t *testing.T) {
    t.Run("roundtrip", hegel.Case(func(ht *hegel.T) {
        original := hegel.Draw(ht, hegel.Text(0, 100))
        data, err := json.Marshal(original)
        if err != nil {
            ht.Fatalf("marshal: %v", err)
        }
        var recovered string
        if err := json.Unmarshal(data, &recovered); err != nil {
            ht.Fatalf("unmarshal: %v", err)
        }
        if original != recovered {
            ht.Fatalf("roundtrip failed: %q != %q", original, recovered)
        }
    }))
}
```

### Invariant preservation

```go
func TestSortPreservesLength(t *testing.T) {
    t.Run("length", hegel.Case(func(ht *hegel.T) {
        v := hegel.Draw(ht, hegel.Lists(hegel.Integers(math.MinInt, math.MaxInt)))
        originalLen := len(v)
        sort.Ints(v)
        if len(v) != originalLen {
            ht.Fatalf("sort changed length: %d -> %d", originalLen, len(v))
        }
    }))
}
```

### Oracle / reference implementation

```go
func TestMySortMatchesStd(t *testing.T) {
    t.Run("vs_std", hegel.Case(func(ht *hegel.T) {
        v := hegel.Draw(ht, hegel.Lists(hegel.Integers(math.MinInt, math.MaxInt)))
        expected := make([]int, len(v))
        copy(expected, v)
        sort.Ints(expected)
        actual := mySort(v)
        if !slices.Equal(actual, expected) {
            ht.Fatalf("mismatch: got %v, want %v", actual, expected)
        }
    }))
}
```

### No-crash / robustness

```go
func TestParseNoPanic(t *testing.T) {
    t.Run("no_panic", hegel.Case(func(ht *hegel.T) {
        input := hegel.Draw(ht, hegel.Text(0, -1))
        _ = MyParser(input) // should never panic
    }))
}
```

### Dependent generation

```go
func TestValidIndex(t *testing.T) {
    t.Run("valid_index", hegel.Case(func(ht *hegel.T) {
        n := hegel.Draw(ht, hegel.Integers(1, 20))
        v := hegel.Draw(ht, hegel.Lists(hegel.Integers(math.MinInt, math.MaxInt)).
            MinSize(n).MaxSize(n))
        idx := hegel.Draw(ht, hegel.Integers(0, len(v)-1))
        _ = v[idx] // always valid
    }))
}
```

### Model-based testing (data structures)

```go
func TestMyMapModel(t *testing.T) {
    t.Run("model", hegel.Case(func(ht *hegel.T) {
        myMap := NewMyMap()
        model := make(map[int]int)

        numOps := hegel.Draw(ht, hegel.Integers(0, 100))
        for range numOps {
            op := hegel.Draw(ht, hegel.Integers[uint8](0, 3))
            switch op {
            case 0:
                k := hegel.Draw(ht, hegel.Integers(math.MinInt, math.MaxInt))
                v := hegel.Draw(ht, hegel.Integers(math.MinInt, math.MaxInt))
                myMap.Insert(k, v)
                model[k] = v
            case 1:
                k := hegel.Draw(ht, hegel.Integers(math.MinInt, math.MaxInt))
                myMap.Delete(k)
                delete(model, k)
            case 2:
                k := hegel.Draw(ht, hegel.Integers(math.MinInt, math.MaxInt))
                got, ok1 := myMap.Get(k)
                exp, ok2 := model[k]
                if ok1 != ok2 || got != exp {
                    ht.Fatalf("get(%d): got (%d, %v), want (%d, %v)", k, got, ok1, exp, ok2)
                }
            default:
                if myMap.Len() != len(model) {
                    ht.Fatalf("len: %d != %d", myMap.Len(), len(model))
                }
            }
        }
    }, hegel.WithTestCases(1000)))
}
```

### Commutativity

```go
func TestSetUnionCommutes(t *testing.T) {
    t.Run("commutes", hegel.Case(func(ht *hegel.T) {
        aSlice := hegel.Draw(ht, hegel.Lists(hegel.Integers(math.MinInt, math.MaxInt)))
        bSlice := hegel.Draw(ht, hegel.Lists(hegel.Integers(math.MinInt, math.MaxInt)))
        a := toSet(aSlice)
        b := toSet(bSlice)
        if !setsEqual(union(a, b), union(b, a)) {
            ht.Fatal("union is not commutative")
        }
    }))
}
```

### Idempotence (normalization)

```go
func TestNormalizeIdempotent(t *testing.T) {
    t.Run("idempotent", hegel.Case(func(ht *hegel.T) {
        s := hegel.Draw(ht, hegel.Text(0, -1)) // full Unicode
        once := normalize(s)
        twice := normalize(once)
        if once != twice {
            ht.Fatalf("not idempotent for %q: %q != %q", s, once, twice)
        }
    }, hegel.WithTestCases(1000)))
}
```

### Parse robustness

```go
func TestParseNoError(t *testing.T) {
    t.Run("no_panic", hegel.Case(func(ht *hegel.T) {
        s := hegel.Draw(ht, hegel.Text(0, -1))
        _ = MyType.Parse(s) // should never panic, just return error
    }, hegel.WithTestCases(1000)))
}
```

### Wrapping arithmetic in test values

When computing test values from generated data, guard against overflow in
your *test* code:

```go
// BAD — overflows when k is near math.MaxInt
m[k] = k * 10

// GOOD — generate smaller values for intermediate computation
k := hegel.Draw(ht, hegel.Integers[int16](math.MinInt16, math.MaxInt16))
k32 := int(k)
m[k32] = k32 * k32 // can't overflow int
```

### Using Assume for constraints

```go
func TestDivision(t *testing.T) {
    t.Run("division", hegel.Case(func(ht *hegel.T) {
        a := hegel.Draw(ht, hegel.Integers(-1000, 1000))
        b := hegel.Draw(ht, hegel.Integers(-1000, 1000))
        ht.Assume(b != 0)
        q, r := a/b, a%b
        if a != q*b+r {
            ht.Fatalf("%d != %d*%d + %d", a, q, b, r)
        }
    }))
}
```

### Using Target to guide generation

```go
func TestSeekLargeValues(t *testing.T) {
    t.Run("target", hegel.Case(func(ht *hegel.T) {
        x := hegel.Draw(ht, hegel.Integers(0, 10000))
        ht.Target(float64(x), "maximize_x")
        if x > 9999 {
            ht.Fatalf("x=%d exceeds limit", x)
        }
    }, hegel.WithTestCases(1000)))
}
```

## Gotchas

1. **`Integers` requires both bounds.** Unlike Rust's `integers::<T>()` which
   defaults to the full range, Go's `Integers()` requires explicit min and
   max. Use `math.MinInt`/`math.MaxInt` (or typed equivalents like
   `math.MinInt32`) for unbounded generation.

2. **`Text` and `Binary` require both size bounds.** Pass `maxSize < 0` for
   unbounded: `Text(0, -1)`.

3. **`Optional` returns a pointer type.** `Optional(Integers(0, 10))` returns
   `Generator[*int]`, not `Generator[int]`. Check for `nil` before
   dereferencing.

4. **`IPAddresses` returns `netip.Addr`, not `string`.** Import
   `"net/netip"`. Use `.String()` if you need the string form.

5. **`Dates` and `Datetimes` return `time.Time`, not `string`.** Import
   `"time"`.

6. **Float defaults include NaN and infinity.** `Floats[float64]()` with no
   bounds generates NaN and infinity by default. Use `.AllowNaN(false)` and/or
   `.AllowInfinity(false)` — but consider whether the code *should* handle
   them first.

7. **Excessive Assume rejections fail the test.** If `Assume()` or `Filter()`
   rejects too many inputs, hegel gives up. Restructure generators to produce
   valid inputs directly.

8. **`Note()` only prints on the final replay.** Don't rely on it for progress
   logging — it only appears when displaying the minimal counterexample.

9. **Nested `t.Run()` inside a hegel test panics.** Each hegel test must be
   a separate `t.Run()` call at the top level.

10. **Default collection sizes are small.** `Lists(gen)` with no bounds rarely
    produces 100+ elements. If you need large collections, draw the size
    separately:
    ```go
    n := hegel.Draw(ht, hegel.Integers(50, 300))
    items := hegel.Draw(ht, hegel.Lists(hegel.Integers(math.MinInt, math.MaxInt)).MinSize(n))
    ```

11. **Add `.hegel/` to `.gitignore`.** Hegel creates a `.hegel/` directory for
    caching the server binary and storing the failure database.

12. **`hegel.T` satisfies `testing.TB`.** You can pass `ht` to helper
    functions that accept `testing.TB`. For functions that require
    `*testing.T` specifically, use `ht.T` to access the embedded field.

## Configuration

### SetHegelDirectory

Override the automatically detected hegel data directory:

```go
func TestMain(m *testing.M) {
    hegel.SetHegelDirectory("/custom/path/.hegel")
    os.Exit(m.Run())
}
```

Use this in `TestMain` if automatic project root detection fails.

## Features Not Yet Available in Go

The following features exist in the Rust SDK but are **not available** in Go:

- **Stateful testing** (`#[hegel::state_machine]`, `Variables`, `stateful::run`)
- **`.Unique()` on list generators** — deduplicate keys manually after generation
- **`generators::randoms()`** — hegel-controlled RNG for testing code that uses randomness
- **`#[hegel::composite]` / `compose!`** — build composite generators by combining `FlatMap`, `Map`, and helper functions instead
- **`draw_silent()`** — all draws in Go are silent (no `Debug` bound exists in Go)
- **`derive(DefaultGenerator)` / `derive_generator!`** — no auto-derived generators for structs

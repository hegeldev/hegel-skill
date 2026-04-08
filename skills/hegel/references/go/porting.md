# Porting Go PBT Libraries to Hegel

## From rapid (pgregory.net/rapid)

rapid is the most popular Go PBT library. The main differences:

- rapid uses `t.Draw()` as a method on its test context; hegel uses `hegel.Draw(ht, gen)` as a free function.
- rapid does shrinking in-process; hegel delegates to a server.
- rapid's `rapid.Check` takes a `func(*rapid.T)`; hegel's `hegel.Case` returns a `func(*testing.T)` for use with `t.Run`.

### Test Structure

rapid:

```go
import "pgregory.net/rapid"

func TestAddition(t *testing.T) {
    rapid.Check(t, func(t *rapid.T) {
        a := t.Draw(rapid.IntRange(0, 99), "a")
        b := t.Draw(rapid.IntRange(0, 99), "b")
        if a+b < a || a+b < b {
            t.Fatal("overflow")
        }
    })
}
```

Hegel:

```go
import "hegel.dev/go/hegel"

func TestAddition(t *testing.T) {
    t.Run("addition", hegel.Case(func(ht *hegel.T) {
        a := hegel.Draw(ht, hegel.Integers[int](0, 99))
        b := hegel.Draw(ht, hegel.Integers[int](0, 99))
        if a+b < a || a+b < b {
            ht.Fatal("overflow")
        }
    }))
}
```

But consider: should those bounds be there at all? If the property is about addition, test the full range unless there's a reason not to.

### Generator Mapping

| rapid | Hegel |
|-------|-------|
| `rapid.Int()` | `hegel.Integers[int](math.MinInt, math.MaxInt)` |
| `rapid.IntRange(lo, hi)` | `hegel.Integers[int](lo, hi)` |
| `rapid.Int8()` | `hegel.Integers[int8](math.MinInt8, math.MaxInt8)` |
| `rapid.Uint64()` | `hegel.Integers[uint64](0, math.MaxUint64)` |
| `rapid.Float64()` | `hegel.Floats[float64]()` |
| `rapid.Float64Range(lo, hi)` | `hegel.Floats[float64]().Min(lo).Max(hi)` |
| `rapid.Bool()` | `hegel.Booleans()` |
| `rapid.String()` | `hegel.Text(0, -1)` |
| `rapid.StringN(min, max, -1)` | `hegel.Text(min, max)` |
| `rapid.SliceOf(gen)` | `hegel.Lists(gen)` |
| `rapid.SliceOfN(gen, min, max)` | `hegel.Lists(gen).MinSize(min).MaxSize(max)` |
| `rapid.MapOf(k, v)` | `hegel.Dicts(k, v)` |
| `rapid.Just(value)` | `hegel.Just(value)` |
| `rapid.SampledFrom(slice)` | `hegel.SampledFrom(slice)` |
| `rapid.OneOf(g1, g2)` | `hegel.OneOf(g1, g2)` |
| `rapid.Ptr(gen, true)` | `hegel.Optional(gen)` |
| `rapid.Map(gen, fn)` | `hegel.Map(gen, fn)` |
| `rapid.Filter(gen, fn)` | `hegel.Filter(gen, fn)` |
| `rapid.StringMatching(re)` | `hegel.FromRegex(re, true)` |

### Drawing Values

rapid uses a method on the test context with a label:

```go
a := t.Draw(rapid.Int(), "a")
```

Hegel uses a free function (no label needed -- hegel tracks draws automatically):

```go
a := hegel.Draw(ht, hegel.Integers[int](math.MinInt, math.MaxInt))
```

### Configuration

| rapid | Hegel |
|-------|-------|
| `rapid.Check(t, fn)` (100 cases) | `hegel.Case(fn)` (100 cases) |
| No direct option for case count | `hegel.Case(fn, hegel.WithTestCases(500))` |
| `t.SkipIf(cond)` | `ht.Assume(!cond)` |

### Dependent Generation

rapid (requires `Map` or `Bind`):

```go
rapid.Check(t, func(t *rapid.T) {
    n := t.Draw(rapid.IntRange(1, 10), "n")
    slice := t.Draw(rapid.SliceOfN(rapid.Int(), n, n), "slice")
    // ...
})
```

Hegel (just use sequential draws):

```go
t.Run("dependent", hegel.Case(func(ht *hegel.T) {
    n := hegel.Draw(ht, hegel.Integers[int](1, 10))
    slice := hegel.Draw(ht, hegel.Lists(hegel.Integers[int](math.MinInt, math.MaxInt)).MinSize(n).MaxSize(n))
    // ...
}))
```

This is one of hegel's main ergonomic advantages -- dependent generation is just sequential code.

## From gopter

gopter is an older Go PBT library inspired by ScalaCheck. It uses a more complex API with explicit property composition.

### Test Structure

gopter:

```go
import (
    "github.com/leanovate/gopter"
    "github.com/leanovate/gopter/gen"
    "github.com/leanovate/gopter/prop"
)

func TestReverse(t *testing.T) {
    properties := gopter.NewProperties(nil)
    properties.Property("reverse involution", prop.ForAll(
        func(xs []int) bool {
            return slices.Equal(reverse(reverse(xs)), xs)
        },
        gen.SliceOf(gen.Int()),
    ))
    properties.TestingRun(t)
}
```

Hegel:

```go
func TestReverse(t *testing.T) {
    t.Run("reverse involution", hegel.Case(func(ht *hegel.T) {
        xs := hegel.Draw(ht, hegel.Lists(hegel.Integers[int](math.MinInt, math.MaxInt)))
        if !slices.Equal(reverse(reverse(xs)), xs) {
            ht.Fatal("reverse is not an involution")
        }
    }))
}
```

Key differences:
- gopter separates generator declaration from test body; hegel puts draws inline.
- gopter properties return `bool`; hegel uses standard assertions (`Fatal`, `Fatalf`).
- gopter uses `gen.Int()`, `gen.SliceOf()`, etc.; see the rapid mapping table above for hegel equivalents.

## From testing/quick

Go's stdlib `testing/quick` package provides basic property testing. It infers generators from function signatures.

### Test Structure

testing/quick:

```go
import "testing/quick"

func TestReverse(t *testing.T) {
    f := func(xs []int) bool {
        return slices.Equal(reverse(reverse(xs)), xs)
    }
    if err := quick.Check(f, nil); err != nil {
        t.Error(err)
    }
}
```

Hegel:

```go
func TestReverse(t *testing.T) {
    t.Run("reverse involution", hegel.Case(func(ht *hegel.T) {
        xs := hegel.Draw(ht, hegel.Lists(hegel.Integers[int](math.MinInt, math.MaxInt)))
        if !slices.Equal(reverse(reverse(xs)), xs) {
            ht.Fatal("reverse is not an involution")
        }
    }))
}
```

Key differences:
- testing/quick infers generators from function parameter types; hegel uses explicit `Draw` calls with configurable generators.
- testing/quick has limited shrinking; hegel provides full automatic shrinking.
- testing/quick defaults to 100 iterations; hegel also defaults to 100 but is configurable via `WithTestCases`.

### Generate Interface

If you implemented `testing/quick.Generator` for custom types, replace with explicit draws:

testing/quick:

```go
type Point struct{ X, Y float64 }

func (Point) Generate(rand *rand.Rand, size int) reflect.Value {
    return reflect.ValueOf(Point{
        X: rand.Float64() * 200 - 100,
        Y: rand.Float64() * 200 - 100,
    })
}
```

Hegel:

```go
t.Run("point test", hegel.Case(func(ht *hegel.T) {
    p := Point{
        X: hegel.Draw(ht, hegel.Floats[float64]().Min(-100).Max(100)),
        Y: hegel.Draw(ht, hegel.Floats[float64]().Min(-100).Max(100)),
    }
    // ...
}))
```

No interface implementation needed. Hegel's imperative style makes custom type generation straightforward.

## Porting Checklist

When porting tests from rapid, gopter, or testing/quick:

1. **Remove the old dependency** from `go.mod` (if no other tests use it) and add hegel: `go get hegel.dev/go/hegel@latest`.
2. **Replace the test structure** with `t.Run("name", hegel.Case(func(ht *hegel.T) { ... }))`.
3. **Convert generators to `hegel.Draw()` calls.** Start with the broadest generators -- don't carry over narrow bounds from the old framework unless they're justified by the function's contract.
4. **Replace framework-specific assertions** (bool returns, `t.SkipIf`) with standard Go assertions (`ht.Fatal`, `ht.Fatalf`) and `ht.Assume()`.
5. **Simplify dependent generation.** If the old test used `Bind` or `FlatMap` chains just to make later values depend on earlier ones, rewrite as sequential `hegel.Draw()` calls.
6. **Remove custom `Generator`/`Arbitrary` implementations.** Replace with inline `hegel.Draw()` calls that construct the value imperatively.
7. **Run the tests.** If they fail on inputs the old framework didn't find, investigate -- that's the point.

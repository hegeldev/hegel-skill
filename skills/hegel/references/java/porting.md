# Porting Java PBT Libraries to Hegel

## From jqwik

[jqwik](https://jqwik.net) is the most widely used Java PBT library. The main differences:

- jqwik is declarative: generated values arrive as `@ForAll` method parameters, constrained by annotations (`@IntRange`, `@StringLength`) or `@Provide` methods. Hegel is imperative — the test receives a `TestCase` and calls `tc.draw()` whenever it needs a value.
- jqwik ships its own JUnit Platform engine; hegel's `@HegelTest` is a plain JUnit 5 (Jupiter) extension, so no extra engine registration is needed.
- jqwik properties may return `boolean`; hegel tests use ordinary JUnit assertions and signal failure by throwing.
- jqwik does generation and shrinking in the JVM; hegel delegates both to libhegel, a native Rust engine (based on Hypothesis) loaded in-process via FFI.

### Test Structure

jqwik:

```java
import net.jqwik.api.*;

class AdditionProperties {
  @Property
  boolean additionCommutes(@ForAll int a, @ForAll int b) {
    return a + b == b + a;
  }
}
```

Hegel:

```java
import static dev.hegel.Generators.integers;
import static org.junit.jupiter.api.Assertions.assertEquals;

import dev.hegel.HegelTest;
import dev.hegel.TestCase;

class AdditionTest {
  @HegelTest
  void additionCommutes(TestCase tc) {
    int a = tc.draw(integers());
    int b = tc.draw(integers());
    assertEquals(a + b, b + a);
  }
}
```

Notes:
- Each `@ForAll` parameter becomes a `tc.draw(...)` call at the top of the body.
- Boolean returns become assertions.
- Consider whether any annotation constraints on the original parameters are justified. If a property is about addition, test the full `int` range — don't carry over an `@IntRange` unless the function's contract requires it.

### Arbitrary → Generator Mapping

| jqwik | Hegel |
|-------|-------|
| `@ForAll int x` | `int x = tc.draw(integers())` |
| `@ForAll @IntRange(min = 0, max = 100) int x` | `tc.draw(integers().min(0).max(100))` |
| `@ForAll long x` | `tc.draw(longs())` |
| `@ForAll double x` | `tc.draw(doubles())` |
| `@ForAll boolean b` | `tc.draw(booleans())` |
| `@ForAll String s` | `tc.draw(text())` |
| `@ForAll @StringLength(min = 1, max = 10) String s` | `tc.draw(text().minSize(1).maxSize(10))` |
| `Arbitraries.integers().between(a, b)` | `integers().min(a).max(b)` |
| `Arbitraries.longs()` | `longs()` |
| `Arbitraries.doubles().between(a, b)` | `doubles().min(a).max(b)` |
| `Arbitraries.strings()` | `text()` |
| `Arbitraries.strings().withCharRange('a', 'z')` | `text().codepoints('a', 'z')` |
| `Arbitraries.strings().numeric()` | `fromRegex("[0-9]*")` |
| `Arbitraries.just(v)` | `just(v)` |
| `Arbitraries.of(v1, v2, v3)` | `sampledFrom(v1, v2, v3)` |
| `Arbitraries.of(Color.class)` (enum) | `forType(Color.class)` |
| `Arbitraries.oneOf(a1, a2)` | `oneOf(g1, g2)` |
| `arbitrary.list()` | `lists(gen)` |
| `arbitrary.list().ofMinSize(1).ofMaxSize(5)` | `lists(gen).minSize(1).maxSize(5)` |
| `arbitrary.set()` | `sets(gen)` |
| `Arbitraries.maps(k, v)` | `maps(k, v)` |
| `arbitrary.optional()` | `optional(gen)` |
| `arbitrary.map(f)` | `gen.map(f)` |
| `arbitrary.filter(p)` | `gen.filter(p)` |
| `arbitrary.flatMap(f)` | `gen.flatMap(f)` (or sequential draws) |
| `Combinators.combine(a, b).as(f)` | sequential `tc.draw` calls (or `composite`) |
| `Arbitraries.lazy(...)` / `Arbitraries.recursive(...)` | `deferred()` |
| `Arbitraries.defaultFor(type)` / `@UseType` | `forType(type)` |
| `@Provide` method | `composite(...)` held in a static field |
| `Assume.that(cond)` | `tc.assume(cond)` |
| `Statistics.collect(...)` | no equivalent |

### Configuration

| jqwik | Hegel |
|-------|-------|
| `@Property` (1000 tries default) | `@HegelTest` (100 test cases default) |
| `@Property(tries = 500)` | `@HegelTest(testCases = 500)` |
| `@Property(seed = "42")` (string) | `@HegelTest(seed = 42)` (long) |
| `@Property(shrinking = ShrinkingMode.OFF)` | `@HegelTest(phases = {Phase.EXPLICIT, Phase.REUSE, Phase.GENERATE, Phase.TARGET})` (omit `SHRINK`) |
| `@Report(Reporting.GENERATED)` | `@HegelTest(verbosity = Verbosity.VERBOSE)` |
| Failure database in `.jqwik-database` | Failure database in `.hegel/` (automatic; disabled in CI) |

Note the differing defaults: jqwik runs 1000 tries per property, hegel runs 100 test cases. Don't blindly set `testCases = 1000` when porting — hegel's default is deliberate; raise it only for properties that warrant extra search.

### `@Provide` methods → composite generators

jqwik:

```java
@Property
boolean sorted(@ForAll("sortedLists") List<Integer> xs) { /* ... */ }

@Provide
Arbitrary<List<Integer>> sortedLists() {
  return Arbitraries.integers().list().map(l -> { Collections.sort(l); return l; });
}
```

Hegel — a plain generator value, referenced directly:

```java
static final Generator<List<Integer>> SORTED_LISTS =
    lists(integers()).map(l -> { var out = new ArrayList<>(l); Collections.sort(out); return out; });

@HegelTest
void sorted(TestCase tc) {
  List<Integer> xs = tc.draw(SORTED_LISTS);
  // ...
}
```

No string-keyed indirection — generators are ordinary values. Use `composite(...)` when the construction needs multiple dependent draws.

### Dependent Generation

jqwik (requires `flatMap` or `Combinators`):

```java
@Provide
Arbitrary<Tuple2<Integer, List<Integer>>> sizedLists() {
  return Arbitraries.integers().between(1, 10)
      .flatMap(n -> Arbitraries.integers().list().ofSize(n).map(l -> Tuple.of(n, l)));
}
```

Hegel (just use sequential draws):

```java
@HegelTest
void sizedLists(TestCase tc) {
  int n = tc.draw(integers().min(1).max(10));
  List<Integer> xs = tc.draw(lists(integers()).minSize(n).maxSize(n));
  // ...
}
```

This is one of hegel's main ergonomic advantages — dependent generation is just sequential code.

### Stateful Testing

jqwik's action-based stateful testing (`ActionChain` / `ActionSequence`) has no hegel-java equivalent yet. Port those tests as a hand-rolled rule loop: draw a bounded step count, draw a rule choice with `sampledFrom` each iteration, dispatch, and assert invariants after each step.

## From junit-quickcheck

junit-quickcheck is JUnit 4-only and injects generated values into `@Property` method parameters, with constraints as parameter annotations. Porting to hegel also moves the test to JUnit 5.

junit-quickcheck:

```java
import com.pholser.junit.quickcheck.Property;
import com.pholser.junit.quickcheck.generator.InRange;
import com.pholser.junit.quickcheck.runner.JUnitQuickcheck;
import org.junit.runner.RunWith;

@RunWith(JUnitQuickcheck.class)
public class StringProperties {
  @Property
  public void concatLength(String s1, String s2) {
    assertEquals(s1.length() + s2.length(), (s1 + s2).length());
  }

  @Property
  public void bounded(@InRange(minInt = 0, maxInt = 100) int x) {
    assertTrue(x >= 0 && x <= 100);
  }
}
```

Hegel:

```java
class StringTest {
  @HegelTest
  void concatLength(TestCase tc) {
    String s1 = tc.draw(text());
    String s2 = tc.draw(text());
    assertEquals(s1.length() + s2.length(), (s1 + s2).length());
  }
}
```

Key differences:
- `@RunWith(JUnitQuickcheck.class)` disappears; `@HegelTest` is per-method and needs no class-level runner.
- Each method parameter becomes a `tc.draw(...)` call; `@InRange(minInt = a, maxInt = b)` becomes `integers().min(a).max(b)` — but reconsider whether the range belongs there at all.
- `@When(satisfies = ...)` and `Assume.assumeThat(...)` become `tc.assume(...)`.
- `@Property(trials = 250)` becomes `@HegelTest(testCases = 250)`.
- Custom `Generator<T>` subclasses (overriding `generate(SourceOfRandomness, GenerationStatus)`) become `composite(...)` generators, or `forType`/`records` for records and enums. Hegel generators are deterministic functions of engine choices, so there is no `SourceOfRandomness` to thread through — draw from other generators instead.

## From QuickTheories

QuickTheories uses a fluent `qt().forAll(...)` DSL with predicate-style checks.

QuickTheories:

```java
import static org.quicktheories.QuickTheory.qt;
import static org.quicktheories.generators.SourceDSL.*;

@Test
public void additionCommutes() {
  qt()
      .forAll(integers().all(), integers().all())
      .check((a, b) -> a + b == b + a);
}
```

Hegel:

```java
@HegelTest
void additionCommutes(TestCase tc) {
  int a = tc.draw(integers());
  int b = tc.draw(integers());
  assertEquals(a + b, b + a);
}
```

Key differences:
- The subjects of `forAll(...)` become `tc.draw(...)` calls; `check(predicate)` becomes an assertion (`checkAssert` bodies port directly).
- `qt().withExamples(n)` becomes `@HegelTest(testCases = n)`; `qt().withFixedSeed(s)` becomes `@HegelTest(seed = s)`.
- `.assuming(predicate)` becomes `tc.assume(...)` on the drawn values.
- `.as(f)` / `.asWithPrecursor(f)` (mapping tuples into domain objects) becomes a `.map` on a generator, or plain construction from sequential draws — no precursor machinery needed, since hegel shrinks through `tc.draw` calls directly.

Generator mapping:

| QuickTheories | Hegel |
|---------------|-------|
| `integers().all()` | `integers()` |
| `integers().between(a, b)` | `integers().min(a).max(b)` |
| `integers().allPositive()` | `integers().min(1)` |
| `longs().all()` | `longs()` |
| `doubles().between(a, b)` | `doubles().min(a).max(b)` |
| `booleans().all()` | `booleans()` |
| `strings().allPossible().ofLengthBetween(a, b)` | `text().minSize(a).maxSize(b)` |
| `strings().basicLatinAlphabet().ofLength(n)` | `text().codepoints(0x20, 0x7E).minSize(n).maxSize(n)` |
| `lists().of(g).ofSizeBetween(a, b)` | `lists(g).minSize(a).maxSize(b)` |
| `maps().of(k, v).ofSize(n)` | `maps(k, v).minSize(n).maxSize(n)` |
| `arbitrary().pick(list)` | `sampledFrom(list)` |
| `arbitrary().constant(v)` | `just(v)` |
| `arbitrary().enumValues(E.class)` | `forType(E.class)` |
| `gen.map(f)` | `gen.map(f)` |
| `gen.flatMap(f)` | `gen.flatMap(f)` (or sequential draws) |

## Porting Checklist

When porting tests from jqwik, junit-quickcheck, or QuickTheories:

1. **Swap the dependency.** Remove the old library from the build (if no other tests use it) and add `dev.hegel:hegel` (test scope). Ensure Java 22+ and `--enable-native-access=ALL-UNNAMED` on the test JVM.
2. **Replace the test structure** with `@HegelTest` methods taking a single `TestCase` parameter (JUnit 5).
3. **Convert injected parameters and arbitraries to `tc.draw()` calls.** Start with the broadest generators — don't carry over `@IntRange`/`between` bounds from the old framework unless they're justified by the function's contract.
4. **Replace boolean-return checks and framework assumptions** with JUnit assertions and `tc.assume()`.
5. **Simplify dependent generation.** `flatMap`/`Combinators.combine` chains that only existed to make later values depend on earlier ones become sequential `tc.draw()` calls.
6. **Replace custom generator classes and `@Provide` methods** with generator values: `composite(...)`, `forType(...)`, or `records(...).with(...)`.
7. **Run the tests.** If they fail on inputs the old framework didn't find, investigate — that's the point.

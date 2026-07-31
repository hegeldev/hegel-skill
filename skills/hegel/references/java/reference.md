# Hegel Java Reference

## Table of Contents

- [Setup](#setup)
- [Test Structure](#test-structure) — `@HegelTest`, `Hegel.test`, `Settings`, `HealthCheck`, database
- [TestCase Methods](#testcase-methods) — `draw`, `assume`, `note`, `target`
- [Generator Reference](#generator-reference) — Numeric, boolean, text, binary, collections, tuples, optional, format, date/time, regex
- [Combinator Methods](#combinator-methods) — `.map()`, `.filter()`, `.flatMap()`
- [Composite Generators](#composite-generators) — `composite`, `deferred`, `forType`, `records`
- [Java-Specific Examples](#java-specific-examples) — Dependent generation, derived records, recursive data, overflow
- [Gotchas](#gotchas)

## Setup

Published to Maven Central as `dev.hegel:hegel`. Maven:

```xml
<dependency>
  <groupId>dev.hegel</groupId>
  <artifactId>hegel</artifactId>
  <!-- latest version: https://central.sonatype.com/artifact/dev.hegel/hegel -->
  <version>VERSION</version>
  <scope>test</scope>
</dependency>
```

Gradle: `testImplementation("dev.hegel:hegel:VERSION")`.

Hegel requires **Java 22+** (it uses the Foreign Function & Memory API to call libhegel, the native Rust engine, in-process). Pass `--enable-native-access=ALL-UNNAMED` to the test JVM to silence the native-access warning:

```xml
<!-- Maven Surefire -->
<argLine>--enable-native-access=ALL-UNNAMED</argLine>
```

```kotlin
// Gradle
tasks.test { jvmArgs("--enable-native-access=ALL-UNNAMED") }
```

`@HegelTest` is a JUnit 5 (Jupiter) extension, so the project needs JUnit 5 on the test classpath (hegel declares it as an optional dependency — you bring your own). The programmatic `Hegel.test(...)` entry point has no JUnit requirement.

Supported platforms: Linux (amd64 and arm64) and macOS (Apple Silicon). Windows and Intel macOS are not supported.

**Sandboxed/offline environments:** the jar is self-contained — the native engine is bundled inside it for every supported platform and unpacked to a per-user cache on first use. Nothing is downloaded at runtime; network access is only needed for Maven/Gradle to resolve the artifact itself (skip even that if it's already in the local repository cache). To use a locally built engine instead, set `HEGEL_LIBHEGEL_PATH` to the shared-library path.

## Test Structure

### `@HegelTest` (preferred)

Annotate a JUnit 5 method with `@HegelTest` (in place of `@Test`) and give it a single `TestCase` parameter. Use ordinary JUnit assertions in the body:

```java
import static dev.hegel.Generators.integers;
import static org.junit.jupiter.api.Assertions.assertEquals;

import dev.hegel.HegelTest;
import dev.hegel.TestCase;

class ArithmeticTest {
  @HegelTest
  void additionCommutes(TestCase tc) {
    int x = tc.draw(integers());
    int y = tc.draw(integers());
    assertEquals(x + y, y + x);
  }
}
```

The test appears as a single entry in the JUnit test tree. Any uncaught exception in the body marks the case as failing and triggers shrinking; the original assertion error (with shrunk values) is rethrown by default.

With configuration — every `Settings` knob has a corresponding annotation attribute:

```java
@HegelTest(testCases = 500, seed = 42)
void thorough(TestCase tc) { /* ... */ }
```

Attributes:
- `testCases` (`long`, default 100) — Number of valid test cases to run
- `seed` (`long`) — Fixed RNG seed for reproducibility (default: no fixed seed)
- `verbosity` (`Verbosity`, default `NORMAL`) — `QUIET`, `NORMAL`, `VERBOSE`, `DEBUG`
- `derandomize` (`OptBoolean`, default `DEFAULT`) — Force deterministic (`TRUE`) or random (`FALSE`) input selection; the default is deterministic in CI, random otherwise
- `phases` (`Phase[]`, default all) — Enable only the listed phases: `EXPLICIT`, `REUSE`, `GENERATE`, `TARGET`, `SHRINK`
- `suppressHealthCheck` (`HealthCheck[]`, default none) — Suppress specific health checks
- `database` (`String`) — `""` (default) keeps the engine default; `Database.DISABLED` (a sentinel constant) disables persistence; any other string is used as the database directory
- `mode` (`Mode`, default `TEST_RUN`) — `SINGLE_TEST_CASE` runs exactly one case with no shrinking, replay, or database
- `reportMultipleFailures` (`boolean`, default `false`) — Keep searching after the first failure and aggregate distinct failures into one report
- `name` (`String`) — Property name used to derive a stable database key (defaults to the method name)

### `Hegel.test` (programmatic)

The escape hatch when a setting must come from a runtime value (annotation attributes are compile-time constants — e.g. a `@TempDir` database path) or when running a property outside a JUnit method:

```java
import dev.hegel.Hegel;
import dev.hegel.Settings;

Hegel.test(tc -> {
  int x = tc.draw(integers());
  int y = tc.draw(integers());
  assertEquals(x + y, y + x);
});

// with settings
Hegel.test(tc -> { /* ... */ }, new Settings().testCases(500).seed(42));
```

### Settings

`Settings` is an immutable fluent builder — each method returns a new instance:

- `.testCases(long)` — Number of test cases (default: 100)
- `.seed(long)` — Pin the RNG seed
- `.derandomize(boolean)` — Force deterministic/random input selection (default: deterministic in CI)
- `.database(Database)` — `Database.unset()` (engine default), `Database.disabled()`, or `Database.path(String)`
- `.suppressHealthCheck(HealthCheck...)` — Suppress checks; calls accumulate
- `.phases(Phase...)` — Enable only the listed phases (an empty call runs no phases)
- `.verbosity(Verbosity)` — Output level
- `.mode(Mode)` — `Mode.TEST_RUN` (default) or `Mode.SINGLE_TEST_CASE`
- `.reportMultipleFailures(boolean)` — Aggregate every distinct failure instead of rethrowing the first directly (default: `false` — a directly rethrown failure keeps its type and stack trace)
- `.name(String)` — Property name for the database key

### HealthCheck

`HealthCheck` variants:
- `HealthCheck.FILTER_TOO_MUCH` — Too many test cases rejected via `assume()` or `.filter()`
- `HealthCheck.TOO_SLOW` — Test execution is too slow
- `HealthCheck.TEST_CASES_TOO_LARGE` — Generated test cases are too large
- `HealthCheck.LARGE_INITIAL_TEST_CASE` — The smallest natural input is very large

A firing health check aborts the run with a `HealthCheckFailure` (a subclass of `HegelException`, distinct from a property's own failure). If the flagged behaviour is intentional, suppress it:

```java
@HegelTest(suppressHealthCheck = {HealthCheck.FILTER_TOO_MUCH})
void heavilyFiltered(TestCase tc) { /* ... */ }
```

### Example database

Hegel persists failing examples (engine default: `.hegel/examples` relative to the working directory — under Maven, the module root) and replays them on subsequent runs. The database key derives from the test method name (override with `name`). In CI environments (`CI`, `GITHUB_ACTIONS`, `GITLAB_CI`, `BUILDKITE`, `CIRCLECI`) the database is disabled automatically.

```java
// Custom directory (annotation form takes a string)
@HegelTest(database = "my_hegel_db")
void custom(TestCase tc) { /* ... */ }

// Disable persistence
@HegelTest(database = Database.DISABLED)
void noDb(TestCase tc) { /* ... */ }

// Runtime path — use the programmatic form
Hegel.test(tc -> { /* ... */ }, new Settings().database(Database.path(tempDir.toString())));
```

## TestCase Methods

| Method | Signature | Purpose |
|--------|-----------|---------|
| `draw` | `<T> T draw(Generator<T> g)` | Draw a value; printed as `draw_N = ...;` in counterexample output |
| `draw` (labelled) | `<T> T draw(Generator<T> g, String label)` | Draw a value named `label` in counterexample output |
| `assume` | `void assume(boolean condition)` | Reject this test case if condition is false (discarded, not failed) |
| `note` | `void note(String message)` | Record debug info (only printed on the final counterexample replay) |
| `target` | `void target(double value)` | Guide generation toward maximizing a score |
| `target` (labelled) | `void target(double value, String label)` | Like `target`, with an explicit label so multiple targets optimize independently |

Signal failure by throwing — use JUnit's `Assertions` (or any assertion library). On the final replay of a minimal failing example, each top-level `draw` is printed as an assignment:

```
xs = [0, 0];
```

Pass a label — `tc.draw(lists(integers()), "xs")` — to get `xs = ...` instead of `draw_1 = ...`.

### Usage

```java
@HegelTest
void divisionIdentity(TestCase tc) {
  int a = tc.draw(integers(), "a");
  int b = tc.draw(integers(), "b");
  tc.assume(b != 0);
  tc.note("dividing " + a + " by " + b);
  int q = a / b;
  int r = a % b;
  assertEquals(a, q * b + r);
}
```

## Generator Reference

All generators are static factory methods on `dev.hegel.Generators`. Static-import them:

```java
import static dev.hegel.Generators.*;
```

The bound- and size-bearing generators are fluent builders that *are* the generator — each configuration method returns a new immutable generator.

### Numeric Generators

**`integers()`** — Generate `int` across the full `int` range. **`longs()`** — Generate `long` across the full `long` range.

```java
int n = tc.draw(integers());                    // full int range
int bounded = tc.draw(integers().min(1).max(100));
long big = tc.draw(longs());                    // full long range
```

Config methods (both): `.min(v)` / `.max(v)` — inclusive bounds.

There are no generators for `short`/`byte`; use `integers().min(...).max(...)` and cast.

**`doubles()`** — Generate 64-bit `double`. **`floats()`** — Generate 32-bit `float`.

```java
double d = tc.draw(doubles());
double unit = tc.draw(doubles().min(0).max(1));
float f = tc.draw(floats().min(0).max(1));
```

Config methods (both):
- `.min(v)` / `.max(v)` — Inclusive bounds
- `.excludeMin(boolean)` / `.excludeMax(boolean)` — Make a bound exclusive
- `.allowNan(boolean)` — Default: `true` if unbounded, `false` if bounded
- `.allowInfinity(boolean)` — Default: `true` unless both bounds are set

### Boolean Generator

```java
boolean b = tc.draw(booleans());
```

### Text and Binary Generators

**`text()`** — Generate `String` (full Unicode by default; surrogates are always excluded so strings round-trip cleanly).

```java
String s = tc.draw(text());
String bounded = tc.draw(text().minSize(1).maxSize(50));
String ascii = tc.draw(text().codepoints(0, 127));
String upper = tc.draw(text().categories("Lu"));
```

Config methods:
- `.minSize(int)` — Minimum codepoint count (default: 0)
- `.maxSize(int)` — Maximum codepoint count (no default — unbounded)
- `.codepoints(min, max)` — Restrict to an inclusive Unicode codepoint range
- `.categories(String...)` — Restrict to Unicode general categories (e.g. `"Lu"`, `"Nd"`)
- `.excludeCategories(String...)` — Exclude Unicode general categories
- `.includeCharacters(String)` — Always include these characters
- `.excludeCharacters(String)` — Never include these characters

There is no `alphabet` or `codec` option. For ASCII use `.codepoints(0, 127)`; for a fixed alphabet use `fromRegex("[abc]*")` or `lists(sampledFrom("a", "b", "c"))` plus a joining `.map`.

**`characters()`** — Generate single-character `String`s (it is `text().minSize(1).maxSize(1)`, so all `text()` character filters apply).

**`binary()`** — Generate `byte[]`.

```java
byte[] bytes = tc.draw(binary());
byte[] sized = tc.draw(binary().minSize(1).maxSize(256));
```

Config methods: `.minSize(int)`, `.maxSize(int)`.

### Constant and Choice Generators

```java
int x = tc.draw(just(42));                                    // always 42
String suit = tc.draw(sampledFrom("hearts", "diamonds", "clubs", "spades"));
List<String> options = List.of("a", "b");
String o = tc.draw(sampledFrom(options));                     // List overload
```

`sampledFrom` throws `IllegalArgumentException` on an empty collection. The first element is the simplest for shrinking.

**`oneOf(g1, g2, ...)`** — Choose among alternative generators of a common type:

```java
int n = tc.draw(oneOf(just(0), integers().min(1).max(100), integers().min(-100).max(-1)));
```

**`optional(gen)`** — Generate `java.util.Optional<T>`, empty or wrapping a generated value:

```java
Optional<Integer> maybe = tc.draw(optional(integers()));
```

### Collection Generators

**`lists(element)`** — `List<T>`. **`sets(element)`** — `Set<T>` (distinct elements). **`maps(keys, values)`** — `Map<K, V>` (any key type).

```java
List<Integer> xs = tc.draw(lists(integers()));
List<Integer> bounded = tc.draw(lists(integers()).minSize(1).maxSize(10));
Set<String> tags = tc.draw(sets(text().maxSize(5)));
Map<String, Integer> counts = tc.draw(maps(text().maxSize(10), integers()).maxSize(5));
```

Config methods (all): `.minSize(int)` (default: 0), `.maxSize(int)`.

### Tuple Generators

Typed overloads for two to eight elements return `Tuple2` … `Tuple8` records with `value1()`, `value2()`, … accessors:

```java
import dev.hegel.Tuple2;

Tuple2<Integer, String> pair = tc.draw(tuples(integers(), text()));
int n = pair.value1();
String s = pair.value2();
```

Above eight elements, the variadic `tuples(Generator<?>...)` fallback returns `Generator<List<Object>>`.

### Format Generators

```java
String email = tc.draw(emails());
String url = tc.draw(urls());
String domain = tc.draw(domains());
String ip = tc.draw(ipAddresses());            // IPv4 or IPv6, as a String
String ipv4 = tc.draw(ipAddresses().v4());
String ipv6 = tc.draw(ipAddresses().v6());
UUID id = tc.draw(uuids());                    // java.util.UUID, any version
UUID v4 = tc.draw(uuids().version(4));         // pin an RFC 4122 version (1-5)
```

### Date/Time Generators

The temporal generators produce `java.time` types, not strings:

```java
import java.time.*;

LocalDate date = tc.draw(dates());
LocalTime time = tc.draw(times());
LocalDateTime dt = tc.draw(datetimes());
ZonedDateTime zoned = tc.draw(datetimes().timezones(zoneIds()));       // DST-aware
OffsetDateTime offset = tc.draw(datetimes().offsets(zoneOffsets()));   // fixed offset
ZoneOffset zo = tc.draw(zoneOffsets());        // .min(ZoneOffset)/.max(ZoneOffset)
ZoneId zone = tc.draw(zoneIds());              // any zone the JVM supports
Duration d = tc.draw(durations().max(Duration.ofSeconds(60)));
```

- `datetimes().timezones(zoneGen)` takes any `Generator<? extends ZoneId>` — pin one with `datetimes().timezones(just(ZoneId.of("Europe/London")))`.
- `durations()` generates **non-negative** durations, `[0, Long.MAX_VALUE]` nanoseconds (~292 years) by default; narrow with `.min(Duration)` / `.max(Duration)`.

### Regex Generator

```java
String code = tc.draw(fromRegex("[A-Z]{3}-[0-9]{3}"));
```

Patterns are Python-compatible regexes passed as strings. `fullmatch` defaults to `true` (the entire string matches); chain `.fullmatch(false)` to generate strings that merely contain a match.

## Combinator Methods

Combinators are default methods on the `Generator<T>` interface — chain them on any generator.

### `.map(f)`

Transform generated values. Preserves the efficient single-engine-call path when the source generator is schema-backed:

```java
Generator<String> positiveStr = integers().min(1).map(String::valueOf);
Generator<Integer> evens = integers().min(0).max(50).map(x -> x * 2);
```

### `.filter(predicate)`

Keep only values matching a predicate:

```java
Generator<Integer> even = integers().filter(x -> x % 2 == 0);
```

`.filter()` retries up to 3 times, then rejects the test case (as if by `assume(false)`). Prefer bounds or `.map()` over filters when possible.

### `.flatMap(f)`

Dependent generation — use one value to choose the next generator:

```java
Generator<List<Boolean>> sized =
    integers().min(0).max(10).flatMap(n -> lists(booleans()).minSize(n).maxSize(n));
```

In most cases, prefer sequential `tc.draw()` calls in the test body — they read more naturally and produce the same shrinking behavior. Use `.flatMap()` only when you need the result packaged as a `Generator<U>`.

## Composite Generators

### `composite(body)`

`Generators.composite` packages an imperative draw sequence into a reusable `Generator<T>`. The function receives the same `TestCase` the test body uses and may call `tc.draw` any number of times — conditionally, in loops, or recursively:

```java
record Person(String name, int age, boolean drivingLicense) {}

Generator<Person> personGen = composite(tc -> {
  int age = tc.draw(integers().min(0).max(120));
  String name = tc.draw(text());
  boolean license = age >= 18 && tc.draw(booleans());
  return new Person(name, age, license);
});

@HegelTest
void personProperty(TestCase tc) {
  Person p = tc.draw(personGen);
  // assert properties of p
}
```

You can also draw the fields inline in the test body — both styles are idiomatic. Reach for `composite` when the generator should be reusable, named, or passed to combinators like `lists` or `optional`.

### `deferred()` — recursive generators

`Generators.deferred()` creates a forward reference so a generator can refer to itself. Pass the `Deferred` into other generators, then call `.set(...)` exactly once with the real implementation:

```java
// A binary tree of integers: a leaf, or a branch of two subtrees.
Deferred<Tree> tree = deferred();
Generator<Tree> leaf = integers().map(Leaf::new);
Generator<Tree> branch = tuples(tree, tree).map(t -> new Branch(t.value1(), t.value2()));
tree.set(oneOf(leaf, branch));

Tree t = tc.draw(tree);
```

The engine's size control keeps generated structures finite. Drawing before `set` is called throws; `set` may be called only once.

### `forType(Class)` and `records(Class)` — type-directed derivation

`Generators.forType` derives a generator by reflection. Supported: scalars (`int`, `long`, `boolean`, `float`, `double`, `String`, `byte[]`, `UUID`, `Duration`, the `java.time` date/time types, and wrappers), enums, records (recursively), and `List`/`Set`/`Optional`/`Map` of supported element types:

```java
record Point(int x, int y) {}
enum Color { RED, GREEN, BLUE }

Point p = tc.draw(forType(Point.class));
Color c = tc.draw(forType(Color.class));
```

`Generators.records` derives a record generator with per-component overrides via `.with(name, generator)`:

```java
Point bounded = tc.draw(records(Point.class).with("x", integers().min(0).max(9)));
```

Unsupported types throw `HegelException` — supply a generator explicitly (e.g. a `.with(...)` override or a `composite`).

## Java-Specific Examples

These examples show Java-specific idioms. For general property patterns (round-trip, model-based, idempotence, etc.), see the main skill's Property Catalogue.

### Dependent generation with sequential draws

Hegel's imperative style means dependent generation is just sequential code — no `flatMap` needed:

```java
@HegelTest
void validIndex(TestCase tc) {
  List<Integer> xs = tc.draw(lists(integers()).minSize(1));
  int idx = tc.draw(integers().min(0).max(xs.size() - 1));
  // idx is always a valid index
  xs.get(idx);
}
```

### Derived record generators

For record-heavy code, derive the generator and override only the components the contract constrains:

```java
record Config(int maxRetries, long timeoutMs, String name) {}

@HegelTest
void configMerge(TestCase tc) {
  Config base = tc.draw(forType(Config.class));
  Config override = tc.draw(records(Config.class).with("maxRetries", integers().min(0).max(100)));
  Config merged = base.merge(override);
  assertEquals(override.name(), merged.name());
}
```

### Avoiding silent overflow in test values

Java's `int` and `long` arithmetic wraps silently. That's a hazard in *test* code: `k * 10` near `Integer.MAX_VALUE` doesn't throw — it quietly produces a wrapped negative value, so the test exercises a meaningless input rather than the value you thought you generated. Compute test values in a wider type:

```java
// Risky — k * 10 silently wraps for k near Integer.MAX_VALUE
int k = tc.draw(integers());
map.put(k * 10, k);

// Better — widen to long so the arithmetic can't wrap
long key = (long) k * 10;
```

Distinguish "this constraint protects the library's contract" (keep it) from "this constraint prevents my test arithmetic from wrapping" (widen instead).

## Gotchas

1. **`@HegelTest` replaces `@Test` — don't stack both.** `@HegelTest` is itself a JUnit `@TestTemplate`; adding `@Test` produces a second, non-hegel invocation. The method must take exactly one `TestCase` parameter.

2. **Java 22+ and `--enable-native-access=ALL-UNNAMED` are required.** Hegel calls the native engine via the FFM API. Without the flag the JVM prints a native-access warning (slated to become an error in future JDKs). Add it to the Surefire `<argLine>` or Gradle `jvmArgs`.

3. **`@HegelTest` needs JUnit 5 on the test classpath.** Hegel's JUnit dependency is optional — the project brings its own `junit-jupiter`. `Hegel.test(...)` works without JUnit.

4. **`integers()` is `int`, `longs()` is `long`; both default to the full range.** Bounds are fluent (`integers().min(0).max(100)`), not constructor arguments. If a value can legitimately exceed `int`, use `longs()` — don't narrow.

5. **`floats()` is 32-bit `float`; `doubles()` is 64-bit.** Unlike libraries where "floats" means the widest type. For most Java code, `doubles()` is what you want.

6. **Float/double defaults include NaN and infinity.** Unbounded `doubles()` generates `NaN` and infinities. If the code under test doesn't handle these, use `.allowNan(false)` / `.allowInfinity(false)` — but consider whether the code *should* handle them first.

7. **`text()` has no `alphabet` or `codec` option.** Use `.codepoints(0, 127)` for ASCII, `.categories(...)` for Unicode classes, or `fromRegex` for a fixed alphabet. `characters()` generates single-character `String`s, not `char`.

8. **`ipAddresses()` returns `String`; date/time generators return `java.time` types.** `dates()` → `LocalDate`, `times()` → `LocalTime`, `datetimes()` → `LocalDateTime` (`.timezones(...)` → `ZonedDateTime`, `.offsets(...)` → `OffsetDateTime`), `uuids()` → `java.util.UUID`.

9. **`durations()` is non-negative by default.** It spans `[0, Long.MAX_VALUE]` nanoseconds. If the code under test accepts negative `Duration` values, cover them explicitly, e.g. `oneOf(durations(), durations().map(Duration::negated))`.

10. **`optional(gen)` produces `java.util.Optional<T>`, never `null`.** Check `.isPresent()`, not `!= null`.

11. **Excessive `assume`/`filter` rejections fail the test.** Too many rejections trigger the `FILTER_TOO_MUCH` health check, which aborts with `HealthCheckFailure`. Restructure generators to produce valid inputs directly.

12. **`note()` and draw labels only print on the final replay.** Don't rely on `note` for progress logging — it appears only when the minimal counterexample is displayed. Unlabelled draws print as `draw_1 = ...;`; pass `tc.draw(gen, "name")` for readable output.

13. **Default collection sizes are small.** `lists(gen)` with no bounds rarely produces 100+ elements. To exercise deep traversals, draw the size separately:
    ```java
    int n = tc.draw(integers().min(0).max(300));
    List<Integer> xs = tc.draw(lists(integers()).minSize(n));
    ```

14. **Add `.hegel/` to `.gitignore`.** Hegel stores the failing-example database under `.hegel/` (relative to the working directory — the module root under Maven). In CI the database and randomness are handled automatically: the database is disabled and runs are derandomized.

15. **Stateful testing is not yet available in hegel-java.** Until it lands, write rule loops by hand inside a property: draw a rule choice with `sampledFrom`, dispatch on it in a loop, and assert invariants after each step.

16. **`sampledFrom` requires a non-empty collection; `Deferred.set` must be called before drawing** (and only once). Both throw immediately otherwise.

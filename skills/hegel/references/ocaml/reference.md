# Hegel OCaml Reference

## Table of Contents

- [Setup](#setup) — opam packages, dune stanza, libhegel engine, sandboxed/offline note
- [Test Structure](#test-structure) — `let%hegel_test`, `[@@settings]`, Settings, HealthCheck, database, `run_hegel_test`
- [Drawing and TestCase Primitives](#drawing-and-testcase-primitives) — `draw`, `draw_silent`, `assume`, `note`, `target`, `require`, `require_equal`
- [Printable vs Unprintable Generators](#printable-vs-unprintable-generators) — the phantom type, `with_printer`
- [Reproducing Failures](#reproducing-failures) — failure reports, `[@@failure_blobs]`
- [Generator Reference](#generator-reference) — Numeric, boolean, text, binary, collections, tuples, optional, functions, format, regex
- [Combinator Functions](#combinator-functions) — `map`, `flat_map`, `filter`
- [Composite Generators](#composite-generators) — `composite`, plain `tc -> 'a` functions, recursion
- [Derived Generators](#derived-generators) — `[@@deriving hegel_generator]`
- [Stateful Testing](#stateful-testing) — `Stateful.run`, rules, invariants, pools
- [Concurrency](#concurrency) — `clone`, `spawn`, `join`
- [Gotchas](#gotchas)

## Setup

Hegel for OCaml requires **OCaml 5.1+**. Three user-facing packages are published on opam: `hegel` (the library), `ppx_hegel_test` (the `let%hegel_test` extension and `dune runtest` integration — strongly recommended), and `ppx_hegel_generator` (the `[@@deriving hegel_generator]` deriver — only needed if you derive generators for your own types).

```bash
opam install hegel ppx_hegel_test
```

Add hegel to the **existing test library's** dune stanza (don't create a separate PBT-only library):

```
(library
 (name my_tests)
 (libraries hegel)
 (inline_tests (backend ppx_hegel_test))
 (preprocess (pps ppx_hegel_test)))
```

Run tests with `dune runtest`. The `inline_tests` backend makes dune discover and run every `let%hegel_test` automatically, printing a PASS/FAIL line per test.

Generation and shrinking are handled by **libhegel**, Hegel's native Rust engine, loaded in-process via ctypes FFI. There is no server or subprocess. The engine is located automatically at runtime, in order:

1. `$HEGEL_LIBHEGEL_PATH` — explicit path to the library file (or a directory containing it);
2. a prebuilt libhegel bundled into the installed package (opam release tarballs ship the matching-platform binary);
3. a sibling `../hegel-rust/target/release/` (then `.../debug/`) checkout relative to the working directory;
4. a SHA-256-verified download from the hegel-rust GitHub release, cached under `~/.cache/hegel-ocaml/libhegel/<version>/` (respects `$XDG_CACHE_HOME`).

Supported platforms: **Linux** (amd64/arm64) and **macOS** (Apple Silicon). macOS amd64 (Intel) has no published libhegel artifact — point `HEGEL_LIBHEGEL_PATH` at a locally built `libhegel.dylib`.

## Test Structure

### `let%hegel_test` (preferred)

```ocaml
open Hegel
open Hegel.Generators

let%hegel_test commutative_addition tc =
  let a = draw tc (integers ()) in
  let b = draw tc (integers ()) in
  require_equal tc Core.Int.sexp_of_t (a + b) (b + a)
;;
```

`let%hegel_test name tc = body` runs `body` on up to 100 generated cases and shrinks any failure to a minimal counterexample. It also defines `name` as a plain `unit -> unit` function, so you can call it directly from an executable or hand it to another harness (e.g. an Alcotest test case) — and it auto-registers with the runtime so `dune runtest` finds it.

Each `let%hegel_test` additionally emits an Antithesis "always" assertion recording whether the property held; outside Antithesis this is a no-op.

### Settings and `[@@settings]`

Override defaults by attaching a `[@@settings ...]` attribute carrying a `Hegel.settings` value:

```ocaml
let%hegel_test commutative_addition tc =
  let a = draw tc (integers ()) in
  let b = draw tc (integers ()) in
  require_equal tc Core.Int.sexp_of_t (a + b) (b + a)
[@@settings Hegel.settings ~test_cases:500 ()]
;;
```

`Hegel.settings ?test_cases ?seed ()` is the convenience constructor (defaults from `default_settings ()`). Refine any settings value with the `with_*` builders:

```ocaml
[@@settings Hegel.settings ~test_cases:500 () |> with_verbosity Verbose |> with_seed (Some 5)]
```

Builders (each takes the new value, then the settings):
- `with_test_cases : int -> ...` — number of test cases (default: 100)
- `with_seed : int option -> ...` — fixed seed for reproducible runs
- `with_derandomize : bool -> ...` — derive the seed from the test's identity (default: `true` in CI)
- `with_verbosity : verbosity -> ...` — `Quiet`, `Normal` (default), `Verbose`, `Debug`
- `with_database : database -> ...` — `Unset` (engine default location), `Disabled`, or `Path "dir"` (see below)
- `with_suppress_health_check : health_check list -> ...` — *replaces* any previously suppressed list, so pass every check to suppress in one call
- `with_phases : phase list -> ...` — restrict the run to specific phases: `Explicit` (reserved, currently no effect), `Reuse`, `Generate`, `Target`, `Shrink`
- `with_mode : mode -> ...` — `Test_run` (default) or `Single_test_case`: run the body exactly once with no shrinking or replay, for long-running workloads (stateful tests keep applying rules indefinitely in this mode)
- `with_stateful_step_count : int -> ...` — max steps per stateful test (default: 50)
- `with_print_blob : bool -> ...` — print the `rerun with:` replay line on failure (default: `true`)
- `with_report_multiple_failures : bool -> ...` — report every distinct failure rather than only the first (default: `false`)

`health_check` variants: `Filter_too_much` (too many cases rejected via `assume`), `Too_slow`, `Test_cases_too_large`, `Large_initial_test_case`.

### Example database

When the database is `Unset` (the default), the engine uses its default location — a `.hegel/` directory relative to the working directory — to persist failing examples and replay them on later runs. Each `let%hegel_test` gets a stable database key (`file:function_name`) automatically. In CI (detected via common environment variables) the database is `Disabled` and `derandomize` is `true` by default.

Note that under `dune runtest` the working directory is the test runner's build directory inside `_build`, so the database lands there. When running a test executable directly, add `.hegel/` to `.gitignore`.

### `run_hegel_test` (no PPX)

To drive a property from a plain executable or another harness without the PPX:

```ocaml
let () =
  Hegel.run_hegel_test ~settings:(Hegel.settings ~test_cases:50 ()) (fun tc ->
    let n = Hegel.draw tc (Hegel.Generators.integers ~min_value:0 ~max_value:9 ()) in
    assert (n >= 0 && n <= 9))
```

Optional arguments: `?settings`, `?test_location` (source location, used by the Antithesis integration), `?database_key`, `?failure_blobs` (replay blobs — only the first is run).

## Drawing and TestCase Primitives

Every test body receives a `tc : Hegel.test_case` handle. All primitives take it as their first argument.

| Primitive | Signature | Purpose |
|-----------|-----------|---------|
| `draw` | `?label:string -> test_case -> ('a, printable) generator -> 'a` | Draw a value; printed as `name = value` on the failing replay |
| `draw_silent` | `test_case -> ('a, 'p) generator -> 'a` | Draw from any generator (printable or not) without recording it |
| `assume` | `test_case -> bool -> unit` | Reject (discard, not fail) this test case if the condition is false |
| `note` | `test_case -> string -> unit` | Print debug info (only on the final counterexample replay under `Normal` verbosity) |
| `target` | `test_case -> float -> string -> unit` | Guide generation toward maximizing a labelled metric |
| `require` | `test_case -> ?msg:string -> bool -> unit` | Fail with `Failure msg` when the condition is false |
| `require_equal` | `test_case -> ?msg:string -> ('a -> Core.Sexp.t) -> 'a -> 'a -> unit` | Fail with a structural sexp diff when the values differ |

Inside a `let%hegel_test`, the PPX supplies the `let` binding name as the draw's label, so `let x = draw tc gen` prints as `x = value` in the failure report. A name that is shadowed or drawn in a loop is numbered (`x_1`, `x_2`, …). Pass `~label:"y"` to override.

Prefer `require_equal` over `assert (a = b)` for equality properties — it renders both sides and a `-`/`+` structural diff in the failure report instead of a bare "Assertion failed". A plain `assert` works too and still triggers shrinking.

```ocaml
let%hegel_test division_identity tc =
  let a = draw tc (integers ()) in
  let b = draw tc (integers ()) in
  assume tc (b <> 0);
  note tc (Printf.sprintf "dividing %d by %d" a b);
  require_equal tc Core.Int.sexp_of_t a ((a / b * b) + (a mod b))
;;
```

`target` sends an observation to the engine, which biases generation toward higher-scoring inputs:

```ocaml
let%hegel_test grow_size tc =
  let v = draw tc (integers ~min_value:0 ~max_value:1000 ()) in
  target tc (float_of_int v) "size";
  assert (v <= 1000)
;;
```

## Printable vs Unprintable Generators

Generators have type `('a, 'p) generator`, where the phantom `'p` records whether the generator carries a printer:

- **`printable`** generators can be drawn with `draw`, which prints the value in the failure report. All primitive generators (`integers`, `text`, `lists`, `tuples2`, format generators, …) are printable.
- **`unprintable`** generators arise whenever the result type is yours rather than the engine's: `map`, `flat_map`, `sampled_from`, `just`, `composite`, `functions`, pool draws, and `[@@deriving hegel_generator]` results. `draw` won't typecheck on them.

Two ways to use an unprintable generator:

1. **`draw_silent`** — produces the value, records nothing:
   ```ocaml
   let parity = draw_silent tc (map (fun n -> n mod 2) (integers ()))
   ```
2. **`with_printer`** — attach a printer (`'a -> Core.Sexp.t`) to get a printable generator:
   ```ocaml
   let parity = draw tc (with_printer Core.Int.sexp_of_t (map (fun n -> n mod 2) (integers ())))
   ```
   With `ppx_sexp_conv` in your `(preprocess (pps ...))`, `[%sexp_of: int]` is shorthand for the printer. Combinators like `lists` and `one_of` require *printable* element generators, so `with_printer` is also how a mapped/derived generator becomes usable as a list element.

Prefer `with_printer` for values that matter to the counterexample — a failure report you can read is the point.

## Reproducing Failures

A failing run prints a framed report: the test name and location, `Falsified after N test cases (M discarded):`, the drawn values (as s-expressions, named after their `let` bindings), any `note`s, the exception, and a copy-pasteable replay line:

```
--- Failure: every_int_is_small (my_tests.ml:3) ------------------
Falsified after 2 test cases (0 discarded):

  n = 50

Exception: File "my_tests.ml", line 5, characters 2-8: Assertion failed
rerun with: [@@failure_blobs [ "AAEAAAAACgEAAAAy" ]]
```

Paste the attribute onto the test to replay that exact case:

```ocaml
let%hegel_test every_int_is_small tc =
  let n = draw tc (integers ()) in
  assert (n < 50)
[@@failure_blobs [ "AAEAAAAACgEAAAAy" ]]
;;
```

A direct `run_hegel_test` caller passes `~failure_blobs:[ "..." ]` instead. Blobs encode the engine's choice sequence and are only guaranteed to reproduce within the same hegel version. Disable the line with `with_print_blob false`. On a terminal the report is colorized; `HEGEL_COLOR=1|0` forces color on or off.

## Generator Reference

All generators live in `Hegel.Generators`. The examples assume `open Hegel` and `open Hegel.Generators`. Every generator constructor takes a **trailing `()`** after its optional arguments.

### Numeric Generators

**`integers ?min_value ?max_value ()`** — native OCaml `int` (63-bit on 64-bit platforms). Defaults span the full native range.

```ocaml
let n = draw tc (integers ()) in
let die = draw tc (integers ~min_value:1 ~max_value:6 ()) in
```

**`floats ?min_value ?max_value ?exclude_min ?exclude_max ?allow_nan ?allow_infinity ()`** — 64-bit floats.

```ocaml
let f = draw tc (floats ()) in
let unit_interval = draw tc (floats ~min_value:0.0 ~max_value:1.0 ()) in
```

Defaults: `allow_nan` is `true` only when no bounds are set; `allow_infinity` is `true` when at most one bound is set; `exclude_min`/`exclude_max` are `false`.

### Boolean Generator

```ocaml
let b = draw tc (booleans ()) in
```

### Text and Binary Generators

**`text ?min_size ?max_size ?codec ?min_codepoint ?max_codepoint ?categories ?exclude_categories ?include_characters ?exclude_characters ?alphabet ()`** — Unicode `string` (size counts codepoints).

```ocaml
let s = draw tc (text ()) in                                  (* any size, full Unicode *)
let bounded = draw tc (text ~min_size:1 ~max_size:50 ()) in
let ascii = draw tc (text ~codec:"ascii" ()) in
let abc = draw tc (text ~alphabet:"abc" ()) in
```

- `codec` — restrict to characters encodable in a codec (e.g. `"ascii"`, `"utf-8"`, `"latin-1"`)
- `categories` / `exclude_categories` — Unicode general categories (e.g. `["L"; "Nd"]`); mutually exclusive with each other
- `include_characters` / `exclude_characters` — always include/exclude these characters
- `alphabet` — fixed allowed set; mutually exclusive with all other character filters

Surrogates (category `Cs`) are always excluded since OCaml strings are conventionally UTF-8.

**`characters ?codec ?min_codepoint ?max_codepoint ?categories ?exclude_categories ?include_characters ?exclude_characters ()`** — a single Unicode character, returned as a single-codepoint `string` (not `char` — an OCaml `char` is one byte and can't hold a codepoint).

**`binary ?min_size ?max_size ()`** — arbitrary byte `string`.

```ocaml
let bytes = draw tc (binary ~max_size:16 ()) in
```

### Constant and Choice Generators

```ocaml
(* Always the same value — unprintable, so draw_silent *)
let x = draw_silent tc (just 42) in

(* Sample from a fixed list — unprintable *)
let suit = draw_silent tc (sampled_from [ `Hearts; `Diamonds; `Clubs; `Spades ]) in
```

`sampled_from` requires a non-empty list, and sampling is **not uniform**: the engine over-weights boundary indices, so the first element (and to a lesser extent the last) is drawn noticeably more often than the middle ones.

**`one_of generators`** — pick among printable generators of the same type (at least one required):

```ocaml
let n = draw tc (one_of [ integers ~min_value:0 ~max_value:9 ()
                        ; integers ~min_value:90 ~max_value:99 () ]) in
```

### Collection Generators

**`lists elements ?min_size ?max_size ?unique ()`** — `'a list` from a *printable* element generator. `~unique:true` makes elements distinct.

```ocaml
let xs = draw tc (lists (integers ()) ()) in
let nonempty = draw tc (lists (integers ()) ~min_size:1 ()) in
let keys = draw tc (lists (integers ()) ~unique:true ()) in
```

**`assoc_lists keys values ?min_size ?max_size ()`** — `('k * 'v) list` with unique keys, in generation order.

**`hash_tables keys values ?min_size ?max_size ()`** — a polymorphic `Core.Hashtbl.t` with the same entry generation as `assoc_lists`.

```ocaml
let m = draw tc (hash_tables (text ~max_size:4 ()) (integers ()) ~max_size:5 ()) in
```

### Tuple Generators

`tuples2`, `tuples3`, `tuples4` combine printable component generators:

```ocaml
let n, b = draw tc (tuples2 (integers ()) (booleans ())) in
```

For wider products, draw the components sequentially (or use `composite`).

### Optional Generator

```ocaml
let o = draw tc (optional (integers ())) in
(* o : int option — None or Some value *)
```

### Function Generators

**`functions ?name ?sexp_of_arg ~returns:gen ()`** — generate a function `'a -> 'b` whose results are drawn from `returns`, memoized per argument (structural equality). Unprintable — draw with `draw_silent`. On a failing replay, each distinct argument the test applied prints as `f arg = result` (repeat applications of the same argument print only once under the default verbosity):

```ocaml
let%hegel_test map_length_preserved tc =
  let f = draw_silent tc (functions ~sexp_of_arg:Core.Int.sexp_of_t ~returns:(integers ()) ()) in
  let xs = draw tc (lists (integers ()) ()) in
  require_equal tc Core.Int.sexp_of_t (List.length (List.map f xs)) (List.length xs)
;;
```

`functions2` and `functions3` generate curried two- and three-argument functions, keyed on the argument tuple. Omitting `sexp_of_arg` (or using a non-printable `returns`) shows `<opaque>` in the report instead of the argument/result.

### Format Generators

```ocaml
let email = draw tc (emails ()) in                  (* RFC 5321/5322 address *)
let url = draw tc (urls ()) in                      (* RFC 3986, http/https *)
let domain = draw tc (domains ~max_length:64 ()) in (* RFC 1035 FQDN; max_length in [4, 255], default 255 *)
```

Dates and times are **typed Core values**, not strings:

```ocaml
let d = draw tc (dates ()) in       (* Core.Date.t, year in [1, 9999] *)
let t = draw tc (times ()) in       (* Core.Time_ns.Ofday.t, microsecond precision *)
let d, t = draw tc (datetimes ()) in  (* Core.Date.t * Core.Time_ns.Ofday.t *)
```

IP addresses are typed `Ipaddr.t` values:

```ocaml
let ip = draw tc (ip_addresses ()) in               (* V4 or V6 *)
let v4 = draw tc (ip_addresses ~version:`V4 ()) in
(* render with Ipaddr.to_string *)
```

### Regex Generator

**`from_regex pattern ?fullmatch ()`** — strings matching a regex in Python `re` syntax. `fullmatch` defaults to `true` (whole string must match); `~fullmatch:false` generates strings merely containing a match.

```ocaml
let code = draw tc (from_regex "[A-Z]{3}-[0-9]{3}" ()) in
```

## Combinator Functions

Combinators are plain functions in `Hegel.Generators`. Note the argument order: **the function/predicate comes first**, then the generator.

### `map f gen`

Transform generated values. The result is unprintable:

```ocaml
let even = draw_silent tc (map (fun n -> n * 2) (integers ())) in
```

### `flat_map f gen`

Dependent generation — `f` receives the generated value and returns the next generator. The result is unprintable:

```ocaml
let xs =
  draw_silent tc
    (flat_map
       (fun n -> lists (integers ()) ~min_size:n ~max_size:n ())
       (integers ~min_value:0 ~max_value:5 ()))
in
```

In test bodies, prefer sequential `draw` calls over `flat_map` — they read more naturally and shrink the same way. Reach for `flat_map` only when you need the dependency packaged as a single generator value.

### `filter predicate gen`

Keep only matching values. Uniquely among the combinators, `filter` **preserves printability**:

```ocaml
let even = draw tc (filter (fun x -> x mod 2 = 0) (integers ())) in
```

`filter` tries up to 3 times, then rejects the test case (as if by `assume tc false`). Prefer bounds or `map` over filters when possible.

## Composite Generators

The idiomatic way to generate a compound value is a plain function `test_case -> 'a` that draws its parts — no wrapper needed. Call it directly with the same `tc`:

```ocaml
type person = { age : int; name : string; driving_license : bool }

let generate_person tc =
  let age = draw tc (integers ~min_value:0 ~max_value:120 ()) in
  let name = draw tc (text ()) in
  let driving_license = if age >= 18 then draw tc (booleans ()) else false in
  { age; name; driving_license }

let%hegel_test license_implies_adult tc =
  let p = generate_person tc in
  require tc ~msg:"license implies adult" ((not p.driving_license) || p.age >= 18)
;;
```

Note the conditional draw: a later draw can depend on an earlier value — dependent generation is just sequential code.

**`composite generate_fn`** wraps such a function into a first-class `(person, unprintable) generator`, needed when you want to pass it to combinators (`lists`, `optional`, …) or `draw` it. Attach a printer to compose it further:

```ocaml
let person = composite generate_person
let printable_person = with_printer sexp_of_person person   (* needs a sexp_of; see with_printer *)

let%hegel_test people_roundtrip tc =
  let ps = draw tc (lists printable_person ()) in
  (* ... *)
;;
```

Recursive generators: pass the recursion budget as a parameter so each call has its own depth:

```ocaml
type tree = Leaf | Node of tree * int * tree

let rec generate_tree depth tc =
  if depth = 0 || draw_silent tc (booleans ())
  then Leaf
  else (
    let l = generate_tree (depth - 1) tc in
    let v = draw_silent tc (integers ()) in
    let r = generate_tree (depth - 1) tc in
    Node (l, v, r))

let%hegel_test tree_property tc =
  let t = generate_tree 5 tc in
  (* ... *)
;;
```

## Derived Generators

`[@@deriving hegel_generator]` (from the `ppx_hegel_generator` opam package) synthesizes a generator value `<type>_generator : (<type>, unprintable) generator` for your type. Add the deriver to the dune stanza:

```
(preprocess (pps ppx_hegel_test ppx_hegel_generator))
```

```ocaml
type point = { x : int; y : int } [@@deriving hegel_generator]

type shape =
  | Circle of float
  | Rect of int * int
  | Dot
[@@deriving hegel_generator]

let%hegel_test distance_nonnegative tc =
  let p = draw_silent tc point_generator in
  let x = float_of_int p.x and y = float_of_int p.y in
  assert (sqrt ((x *. x) +. (y *. y)) >= 0.0)
;;
```

Supported: records, variants (constructor picked via `sampled_from`, so engine-biased toward earlier constructors), and type aliases. Supported field/argument types: `int` (full native range, like `integers ()`), `bool`, `float` (finite: `~allow_nan:false ~allow_infinity:false`), `string` (`text ()`), `'a list` (engine-driven, same as `lists`), `'a option` (same as `optional`), tuples, and named types `t` / `Module.t` — which draw `t_generator` / `Module.t_generator`, assumed to be in scope (derive or hand-write them).

The derived generator is unprintable. Draw it with `draw_silent`, or add `[@@deriving sexp_of]` (from `ppx_sexp_conv`) and draw `with_printer sexp_of_point point_generator` to print the value on a failing replay.

## Stateful Testing

`Hegel.Stateful` applies a random sequence of *rules* to a model state and checks *invariants* before the first step and after every successful step. Rules are `Stateful.Rule.create ~name ~step`, where `step : test_case -> 'state -> 'state` performs one action, drawing any arguments it needs:

```ocaml
let push =
  Stateful.Rule.create ~name:"push" ~step:(fun tc stack ->
    let n = draw tc (integers ()) in
    n :: stack)

let pop =
  Stateful.Rule.create ~name:"pop" ~step:(fun tc stack ->
    assume tc (stack <> []);   (* skip this rule when it doesn't apply *)
    List.tl stack)

let%hegel_test stack_never_negative tc =
  Stateful.run
    ~init:[]
    ~rules:[ push; pop ]
    ~invariants:[ (fun stack -> assert (List.length stack >= 0)) ]
    ~sexp_of_state:(Core.List.sexp_of_t Core.Int.sexp_of_t)
    tc
;;
```

- `Stateful.run ~init ~rules ?invariants ?sexp_of_state tc` raises `Invalid_argument` if `rules` is empty.
- Inside a rule, `assume tc false` (or a failed `assume` condition) skips the rule; the engine tries another.
- Passing `~sexp_of_state` prints the model state after the initial state and after each step of a failing sequence. The failing replay shows each step (`Step 1: push`), the draws nested under it, and which invariant broke (`Invariant 0 violated after step 3`).
- The max number of steps per test case is `stateful_step_count` (default: 50, set via `with_stateful_step_count`).
- Rule selection uses swarm testing: each test case enables a random subset of the rules, which surfaces bugs that only appear under particular rule combinations.

### Pools

`Stateful.Pool` lets data flow between rules — a rule can act on a handle an earlier rule produced instead of a freshly drawn value:

```ocaml
type state = { live : Core.Int.Set.t; handles : int Stateful.Pool.t }

let alloc =
  Stateful.Rule.create ~name:"alloc" ~step:(fun _tc state ->
    let h = fresh_handle () in
    Stateful.Pool.add state.handles h;
    { state with live = Core.Set.add state.live h })

let free =
  Stateful.Rule.create ~name:"free" ~step:(fun tc state ->
    let h = draw_silent tc (Stateful.Pool.values_consumed state.handles) in
    release h;
    { state with live = Core.Set.remove state.live h })

let%hegel_test allocator tc =
  Stateful.run ~init:{ live = Core.Int.Set.empty; handles = Stateful.Pool.create tc }
    ~rules:[ alloc; free ] tc
;;
```

`Pool` API:
- `create tc` — pools are tied to a test case; don't reuse one across test cases
- `add pool value` / `size pool`
- `values_reusable pool` — unprintable generator returning a value *without* removing it
- `values_consumed pool` — unprintable generator that *removes* and returns a value

Both generators reject the test case (like `assume tc false`) when the pool is empty, and compose with the other combinators — e.g. `draw tc (with_printer Core.Int.sexp_of_t (Stateful.Pool.values_consumed pool))` prints the pick in the failure report.

## Concurrency

A `test_case` handle may only be drawn from by one thread at a time; sharing one across threads raises a concurrent-use error. To generate from multiple threads, give each its own **clone** — an independent stream of the same test case whose draws still shrink and replay deterministically (as long as your code is deterministic):

```ocaml
let%hegel_test two_hands_two_dice tc =
  let die = integers ~min_value:1 ~max_value:6 () in
  let other_hand = spawn tc (fun worker -> draw_silent worker die) in
  let this_hand = draw_silent tc die in
  assert (this_hand + join other_hand >= 2)
;;
```

- `clone tc` — a fresh handle for another thread
- `spawn tc f` — clones `tc` and runs `f clone` on a new thread; prefer it over `Thread.create` (which silently drops worker exceptions)
- `join worker` — awaits the worker, re-raising its exception on the caller's thread; join before the test body returns

A draw is a synchronous engine call holding the domain's runtime lock, so draws on one domain serialize — use separate domains (e.g. Domainslib, Eio + domain manager) for true parallelism, and create expensive resources like domain pools once outside the test body.

## Gotchas

1. **Generator constructors take a trailing `()`.** `integers ()`, `text ~max_size:50 ()`, `lists (integers ()) ()`. Forgetting the unit gives a confusing partial-application type error.

2. **`map`, `flat_map`, `sampled_from`, `just`, `composite`, `functions`, and derived generators are unprintable.** `draw` won't typecheck on them — use `draw_silent`, or attach a printer with `with_printer` so the value shows up in failure reports.

3. **Combinators take the function first.** `map (fun x -> x * 2) gen`, `filter (fun x -> x > 0) gen`, `flat_map f gen` — the opposite order from many OCaml PBT libraries' infix style.

4. **`characters` returns a `string`, not a `char`.** A Unicode codepoint doesn't fit OCaml's one-byte `char`; single characters are single-codepoint UTF-8 strings.

5. **`integers` generates native OCaml `int` (63-bit on 64-bit platforms).** There is no dedicated `int32`/`int64` generator; map through `Int64.of_int` if you need boxed types (this covers the 63-bit range, not full 64-bit).

6. **Float defaults include NaN and infinity when unbounded.** Pass `~allow_nan:false` / `~allow_infinity:false` only if the code under test genuinely shouldn't handle them — consider whether it *should* first.

7. **`sampled_from` is not uniform.** The engine over-weights boundary indices, so the first element is drawn noticeably more often. This is deliberate (boundary values find bugs) — don't use `sampled_from` where you need a uniform distribution.

8. **Excessive `assume`/`filter` rejections fail the test** via the `Filter_too_much` health check. Restructure generators to produce valid inputs by construction (e.g. `~min_size:1` instead of `assume tc (xs <> [])`).

9. **`note` output depends on verbosity.** Never under `Quiet`, only on the final failing replay under `Normal` (the default), on every case under `Verbose`/`Debug`. Don't rely on it for progress logging.

10. **Default collection sizes are small.** `lists gen ()` with no bounds rarely produces 100+ elements. Draw the size separately when you need large collections:
    ```ocaml
    let n = draw tc (integers ~min_value:0 ~max_value:300 ()) in
    let xs = draw tc (lists (integers ()) ~min_size:n ()) in
    ```

11. **Tuples stop at `tuples4`.** For wider products, draw components sequentially or use `composite`.

12. **`dates`/`times`/`datetimes` return Core values and `ip_addresses` returns `Ipaddr.t`** — not strings. Render with `Core.Date.to_string`, `Ipaddr.to_string`, etc.

13. **`with_suppress_health_check` replaces, not appends.** Pass every check to suppress in a single list.

14. **`with_seed` takes an `int option`** — write `with_seed (Some 5)`, or use the constructor form `settings ~seed:5 ()`.

15. **Don't share a `test_case` across threads.** Clone it (`clone`/`spawn`) per thread; concurrent draws on one handle raise an error, and sharing one collection, pool, or state machine across threads produces flaky results.

16. **Failure blobs are version-bound.** A `[@@failure_blobs ...]` attribute replays reliably only under the hegel version that produced it — remove it once the bug is fixed.

# Porting OCaml PBT Libraries to Hegel

## From QCheck (qcheck / qcheck2)

QCheck is the most common OCaml PBT library, with two API generations: the original `QCheck` (arbitraries with manual `~shrink`/`~print`) and `QCheck2` (integrated shrinking). The main differences from hegel:

- QCheck properties are `'a -> bool` functions over a *single* generator — multiple inputs get tupled up (`Gen.pair`, `Gen.tup3`). Hegel draws values inline with `draw tc gen`, one `let` per input.
- QCheck tests are values built with `Test.make` and run via `QCheck_runner.run_tests_main` or adapted to Alcotest with `QCheck_alcotest.to_alcotest`. Hegel tests are `let%hegel_test` definitions that `dune runtest` discovers via the `ppx_hegel_test` inline-tests backend (and each remains a plain `unit -> unit` function you can hand to Alcotest directly).
- QCheck1 custom arbitraries need hand-written `~shrink` and `~print`. Hegel shrinks through the engine for free, and prints each draw named after its `let` binding.

### Test Structure

QCheck2:

```ocaml
let test =
  QCheck2.Test.make ~name:"reverse involution" ~count:100
    QCheck2.Gen.(list int)
    (fun xs -> List.rev (List.rev xs) = xs)

let () = QCheck_runner.run_tests_main [ test ]
```

Hegel:

```ocaml
open Hegel
open Hegel.Generators

let%hegel_test reverse_involution tc =
  let xs = draw tc (lists (integers ()) ()) in
  require_equal tc (Core.List.sexp_of_t Core.Int.sexp_of_t) (List.rev (List.rev xs)) xs
;;
```

Note the property becomes an assertion (`require_equal`, `require`, or plain `assert`) instead of a returned `bool` — and `require_equal` shows a structural diff of the two sides on failure, which the `bool` never could.

### Generator Mapping

| QCheck2.Gen | Hegel |
|-------------|-------|
| `Gen.int` | `integers ()` |
| `Gen.int_range lo hi`, `lo -- hi` | `integers ~min_value:lo ~max_value:hi ()` |
| `Gen.int_bound n` | `integers ~min_value:0 ~max_value:n ()` |
| `Gen.nat`, `Gen.small_nat`, `Gen.small_int` | `integers ~min_value:0 ()` (see note below) |
| `Gen.bool` | `booleans ()` |
| `Gen.float` | `floats ()` |
| `Gen.float_range lo hi` | `floats ~min_value:lo ~max_value:hi ()` |
| `Gen.char` | `characters ~max_codepoint:255 ()` (returns a 1-char `string`, not `char`) |
| `Gen.string` | `text ()` |
| `Gen.string_size (Gen.int_bound n)` | `text ~max_size:n ()` |
| `Gen.string_printable` | `text ~min_codepoint:0x20 ~max_codepoint:0x7e ~include_characters:"\n" ()` (QCheck's printable set is `[0x20, 0x7e]` plus newline) |
| `Gen.list g` | `lists g ()` |
| `Gen.list_size (Gen.int_bound n) g` | `lists g ~max_size:n ()` |
| `Gen.array g` | `map Array.of_list (lists g ())` (unprintable — `draw_silent`) |
| `Gen.pair g1 g2`, `Gen.tup2` | `tuples2 g1 g2` (or two sequential draws) |
| `Gen.triple`, `Gen.tup3` | `tuples3 g1 g2 g3` |
| `Gen.quad`, `Gen.tup4` | `tuples4 g1 g2 g3 g4` |
| `Gen.option g` | `optional g` |
| `Gen.oneof [g1; g2]` | `one_of [ g1; g2 ]` |
| `Gen.oneofl xs` | `sampled_from xs` (unprintable; engine-biased toward the first element) |
| `Gen.frequency` / `Gen.frequencyl` | no weighted equivalent — draw an `integers` and branch, or use `one_of`/`sampled_from` |
| `Gen.pure x`, `Gen.return x` | `just x` (unprintable) |
| `Gen.map f g` | `map f g` (unprintable) |
| `Gen.bind g f`, `g >>= f` | sequential draws, or `flat_map f g` (note the flipped argument order) |
| `Gen.sized` / `Gen.fix` | recursive `tc -> 'a` function with an explicit depth parameter |
| `QCheck.assume cond` | `assume tc cond` |
| `cond ==> prop` | `assume tc cond; <assert prop>` |

The `small_*` and `nat` generators exist in QCheck partly to keep test data manageable for its shrinker. Hegel's engine shrinks well from anywhere in the range — don't carry the smallness over. Port `Gen.small_nat` to `integers ~min_value:0 ()` unless the function's contract genuinely bounds the input.

### Configuration

| QCheck | Hegel |
|--------|-------|
| `Test.make ~count:500 ...` | `[@@settings Hegel.settings ~test_cases:500 ()]` |
| `Test.make ~name:"..."` | the `let%hegel_test` binding name |
| `QCheck_runner.run_tests_main tests` | `dune runtest` (inline-tests backend); no runner list to maintain |
| `QCheck_alcotest.to_alcotest test` | pass the test function directly: `Alcotest.test_case "name" `Quick my_hegel_test` |
| `Test.make ~max_gen ...` (discard budget) | automatic — the `Filter_too_much` health check fails loudly instead |

### Dependent Generation

QCheck2 (requires `bind`):

```ocaml
QCheck2.Gen.(int_range 1 10 >>= fun n -> list_size (pure n) int)
```

Hegel (just sequential draws):

```ocaml
let n = draw tc (integers ~min_value:1 ~max_value:10 ()) in
let xs = draw tc (lists (integers ()) ~min_size:n ~max_size:n ()) in
```

This is one of hegel's main ergonomic advantages — dependent generation is ordinary sequential code, and a later draw can depend on any earlier value.

### Custom Arbitraries (QCheck1)

QCheck1 arbitraries built with `QCheck.make ~print ~shrink gen` port to a plain drawing function; both the printer and the shrinker are dropped (hegel shrinks through the engine, and printing comes from `draw`'s labels or `with_printer`):

```ocaml
(* QCheck1 *)
let arbitrary_point =
  QCheck.make ~print:print_point ~shrink:shrink_point gen_point

(* Hegel *)
let generate_point tc =
  let x = draw tc (integers ()) in
  let y = draw tc (integers ()) in
  { x; y }

(* used as: let p = generate_point tc *)
```

For record/variant types, `[@@deriving hegel_generator]` (from the `ppx_hegel_generator` package) can replace the whole hand-written arbitrary.

## From Crowbar

Crowbar targets fuzzing: a test is a list of generators plus an n-ary function, and the executable runs either in quickcheck mode or under afl-fuzz for coverage-guided input generation. Differences:

- Crowbar's `('f, 'a) gens` list maps positionally onto the function's arguments; hegel draws inline.
- `Crowbar.check` / `check_eq` become `require` / `require_equal`; `guard` becomes `assume`.
- Hegel's engine is not AFL — generation is Hypothesis-style random search plus targeted search (`target tc score label`) and integrated shrinking. Coverage-guided exploration under afl-fuzz does not port; the properties themselves port unchanged. (Every `let%hegel_test` also emits an Antithesis assertion automatically — a no-op outside Antithesis — so a ported suite is ready to run under Antithesis.)

### Test Structure

Crowbar:

```ocaml
let () =
  Crowbar.add_test ~name:"reverse involution" [ Crowbar.list Crowbar.int ]
    (fun xs -> Crowbar.check_eq (List.rev (List.rev xs)) xs)
```

Hegel:

```ocaml
let%hegel_test reverse_involution tc =
  let xs = draw tc (lists (integers ()) ()) in
  require_equal tc (Core.List.sexp_of_t Core.Int.sexp_of_t) (List.rev (List.rev xs)) xs
;;
```

### Generator Mapping

| Crowbar | Hegel |
|---------|-------|
| `Crowbar.int` | `integers ()` |
| `Crowbar.int8` / `Crowbar.uint8` | `integers ~min_value:(-128) ~max_value:127 ()` / `integers ~min_value:0 ~max_value:255 ()` |
| `Crowbar.int32` / `Crowbar.int64` | `map Int32.of_int (integers ~min_value:(-0x8000_0000) ~max_value:0x7fff_ffff ())`; no full-width `int64` generator — `map Int64.of_int (integers ())` covers the native 63-bit range |
| `Crowbar.float` | `floats ()` |
| `Crowbar.bool` | `booleans ()` |
| `Crowbar.bytes` | `binary ()` (arbitrary bytes) or `text ()` (Unicode text), depending on how the value is used |
| `Crowbar.range ~min n` | `integers ~min_value:min ~max_value:(min + n - 1) ()` |
| `Crowbar.const x` | `just x` (unprintable) |
| `Crowbar.choose gens` | `one_of gens` |
| `Crowbar.option g` | `optional g` |
| `Crowbar.list g` | `lists g ()` |
| `Crowbar.list1 g` | `lists g ~min_size:1 ()` |
| `Crowbar.pair g1 g2` | `tuples2 g1 g2` |
| `Crowbar.map [g1; g2] f` | sequential draws, or `map f (tuples2 g1 g2)` |
| `Crowbar.dynamic_bind g f` | sequential draws, or `flat_map f g` |
| `Crowbar.fix` | recursive `tc -> 'a` function with an explicit depth parameter |
| `Crowbar.with_printer pp g` | `with_printer sexp_of g` (takes an `'a -> Core.Sexp.t`, not a Format printer) |
| `Crowbar.guard cond` | `assume tc cond` |
| `Crowbar.bad_test ()` | `assume tc false` |
| `Crowbar.check b` | `require tc b` (or `assert b`) |
| `Crowbar.check_eq ?eq ?pp a b` | `require_equal tc sexp_of a b` |
| `Crowbar.fail msg` | `failwith msg` |

## Porting Checklist

When porting tests from QCheck or Crowbar:

1. **Update the dune stanza.** Remove `qcheck`/`crowbar` from `(libraries ...)` if nothing else uses them; add `hegel`, `(inline_tests (backend ppx_hegel_test))`, and `(preprocess (pps ppx_hegel_test))`. Install with `opam install hegel ppx_hegel_test`.
2. **Replace test registration** (`Test.make` + runner lists, `Crowbar.add_test`) with `let%hegel_test name tc = body`. Delete the runner `main` — `dune runtest` discovers registered tests.
3. **Convert generators to inline `draw` calls.** Untuple `Gen.pair`/`gens`-list inputs into one draw per value. Start broad — don't carry over `small_*` generators or narrow ranges unless the function's contract justifies them.
4. **Replace bool-returning properties and framework assertions** with `require_equal` (equality, with a diff), `require` (boolean with message), or plain `assert`.
5. **Replace `assume`/`==>`/`guard`/`bad_test`** with `Hegel.assume tc`.
6. **Simplify `bind`/`dynamic_bind` chains** into sequential `draw` calls.
7. **Drop manual shrinkers and printers.** Shrinking is integrated; printing comes from `draw`'s `let`-binding labels. Where a mapped or custom generator's value matters to the counterexample, attach `with_printer`.
8. **Run the tests.** If they fail on inputs the old framework never found, investigate — that's the point.

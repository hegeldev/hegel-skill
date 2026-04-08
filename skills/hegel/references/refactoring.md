# Refactoring for Property-Based Testing

Identify code that could be refactored to enable or improve property-based
testing with hegel.

## Quick Reference

| Pattern | Problem | Solution | Properties Enabled |
|---------|---------|----------|-------------------|
| I/O mixed with logic | Can't test without mocks | Extract pure core | Multiple |
| Encode without decode | No roundtrip possible | Add inverse operation | Roundtrip |
| Concrete RNG type | Can't use hegel's RNG | Accept `impl Rng` trait | Shrinkable randomness |
| In-place mutation | Hard to verify before/after | Return new value | Comparison properties |
| String building | Can't verify structure | Structured repr + render | Roundtrip |
| Implicit invariants | Can't test constraints | Add explicit validation | Invariant |

## Refactoring Patterns

### 1. Extract Pure Core from Impure Functions (High Impact)

**Problem**: Functions that mix I/O with logic can't be property-tested without
mocking infrastructure.

```rust
// BEFORE - hard to test
fn process_order(db: &Database, order_id: &str) -> Result<()> {
    let order = db.fetch(order_id)?;       // I/O
    let discount = calculate_discount(&order);  // Pure logic
    let total = apply_discount(&order, discount);  // Pure logic
    db.save(order_id, total)?;             // I/O
    Ok(())
}

// AFTER - pure core extracted
fn calculate_order_total(order: &Order, rules: &DiscountRules) -> Decimal {
    let discount = calculate_discount(order, rules);
    apply_discount(order, discount)
}

fn process_order(db: &Database, order_id: &str) -> Result<()> {
    let order = db.fetch(order_id)?;
    let total = calculate_order_total(&order, &get_discount_rules());
    db.save(order_id, total)?;
    Ok(())
}
```

Now `calculate_order_total` can be property-tested with generated `Order` values.

**Detection**: Functions that take database handles, HTTP clients, file handles,
or similar I/O resources AND contain business logic.

### 2. Add Missing Inverse Operations (High Impact)

**Problem**: One-way operations that should have pairs can't be roundtrip tested.

```rust
// BEFORE - only encode
fn encode_message(msg: &Message) -> Vec<u8> {
    // ...
}

// AFTER - add decode for roundtrip testing
fn encode_message(msg: &Message) -> Vec<u8> {
    // ...
}

fn decode_message(data: &[u8]) -> Result<Message, DecodeError> {
    // ...
}
```

The inverse is often useful beyond testing — if you need to encode, you
probably need to decode somewhere.

**Detection**: Find `encode` without `decode`, `serialize` without
`deserialize`, `to_*` without `from_*`, `pack` without `unpack`.

### 3. Accept Trait Bounds for RNG Types (High Impact for hegel)

**Problem**: Code that takes a concrete RNG type (e.g., `ChaCha8Rng`) can't
be tested with hegel's `gs::randoms()`.

```rust
// BEFORE - concrete type prevents hegel testing
fn sample(weights: &[f64], rng: &mut ChaCha8Rng) -> usize {
    // ...
}

// AFTER - trait bound works with any RNG
fn sample(weights: &[f64], rng: &mut impl Rng) -> usize {
    // ...
}
```

This is both better API design (more flexible) and enables hegel to control
the random decisions for shrinking.

**Detection**: Function parameters with concrete RNG types like `ChaCha8Rng`,
`StdRng`, `ThreadRng`.

### 4. Return Values Instead of Mutating (Medium Impact)

**Problem**: Functions that mutate in place make it hard to compare
before/after state.

```rust
// BEFORE - mutates in place
fn sort_tasks(tasks: &mut Vec<Task>) {
    tasks.sort_by_key(|t| t.priority);
}

// AFTER - returns new value
fn sorted_tasks(tasks: &[Task]) -> Vec<Task> {
    let mut result = tasks.to_vec();
    result.sort_by_key(|t| t.priority);
    result
}
```

Returning new values makes properties like idempotence and element
preservation easier to assert.

**Detection**: Functions returning `()` that modify their arguments via
`&mut` references, especially `sort`, `reverse`, `shuffle`, etc.

### 5. Structured Representation + Render (Medium Impact)

**Problem**: Functions that build strings by concatenation can't be
structurally verified.

```rust
// BEFORE - string building
fn build_query(table: &str, filters: &HashMap<String, String>) -> String {
    let mut q = format!("SELECT * FROM {}", table);
    if !filters.is_empty() {
        q += " WHERE ";
        q += &filters.iter()
            .map(|(k, v)| format!("{} = '{}'", k, v))
            .collect::<Vec<_>>()
            .join(" AND ");
    }
    q
}

// AFTER - structured representation
struct Query {
    table: String,
    filters: HashMap<String, String>,
}

fn render_query(q: &Query) -> String { /* ... */ }
fn parse_query(sql: &str) -> Result<Query, ParseError> { /* ... */ }
```

Now you can roundtrip test: `parse_query(render_query(q)) == q`.

### 6. Make Implicit Invariants Explicit (Lower Impact)

**Problem**: Constraints documented in comments but not enforced in code
can't be tested.

```rust
// BEFORE - constraint only in comment
/// Size must be positive and <= 1MB
fn allocate_buffer(size: usize) -> Vec<u8> {
    vec![0; size]
}

// AFTER - enforced with validation
const MAX_BUFFER_SIZE: usize = 1024 * 1024;

fn allocate_buffer(size: usize) -> Result<Vec<u8>, BufferError> {
    if size == 0 || size > MAX_BUFFER_SIZE {
        return Err(BufferError::InvalidSize(size));
    }
    Ok(vec![0; size])
}
```

Now you can property-test that valid inputs succeed and invalid inputs
return errors.

**Detection**: Comments containing "must be", "should be", "always",
"never", "requires", "precondition".

## Evaluation Criteria

For each refactoring opportunity, evaluate:

| Factor | Questions |
|--------|-----------|
| Properties enabled | What tests become possible? Roundtrip > Idempotence > No crash |
| Effort | How much code change? |
| Risk | Breaking changes? API impact? |
| Independent value | Is the refactoring worthwhile even without testing? |

Prioritize refactorings that:
1. Enable strong properties (roundtrip, idempotence)
2. Require low effort
3. Have independent value beyond testing (better API design)
4. Don't break existing callers

## When to Suggest Refactoring

Suggest refactoring when:
- The user asks for property-based tests but the code isn't testable as-is
- You detect a pattern from the Quick Reference table
- The refactoring has clear independent value (better separation of concerns,
  more flexible API)

Always **ask the user** before refactoring. Present the trade-off:

> "The `process_order` function mixes database I/O with discount calculation.
> Extracting the pure logic into `calculate_order_total` would let us write
> property-based tests for the business rules, and it's cleaner separation of
> concerns. Want me to refactor it?"

Do not refactor silently or refactor code that's out of scope for the current
task.

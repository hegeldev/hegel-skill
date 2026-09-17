# The scale probe

Once, outside the generators: build a very large instance — hundreds of thousands of elements, or deeply nested input — and exercise every operation and derived trait on it: format/Debug, clone, drop, iterate, compare, hash, serialize. Per-element recursion and accidental quadratic behavior only appear at scale; generator-sized values never reach them.

# Summary

- We introduce `#[salsa::interned]` queries which convert a `Key` type
  into a numeric index of type `Value`, where `Value` is either a
  `u32` or a newtype'd `u32`.
- Each interned query `foo` also produces an inverse `lookup_foo`
  method that converts back from the `Value` to the `Key` that was
  interned.
- The `Value` types can be any type that implements the
  `salsa::InternIndex` trait, also introduced by this RFC. This trait
  has two methods, `from_u32` and `as_u32`.
- The interning is integrated into the GC and tracked like any other
  query, which means that interned values can be garbage-collected,
  and any computation that was dependent on them will be collected.

# Motivation

## The need for interning

Many salsa applications wind up needing the ability to construct
"interned keys". Frequently this pattern emerges because we wish to
construct identifiers for things in the input. These identifiers
generally have a "tree-like shape". For example, in a compiler, there
may be some set of input files -- these are enumerated in the inputs
and serve as the "base" for a path that leads to items in the user's
input. But within an input file, there are additional structures, such
as `struct` or `impl` declarations, and these structures may contain
further structures within them (such as fields or methods). This gives
rise to a path like so that can be used to identify a given item:

```
Path = <file-name>
     | Path / <identifier>
```

These paths *could* be represented in the compiler with an `Arc`, but
because they are omnipresent, it is convenient to intern them instead
and use an integer. Integers are `Copy` types, which is convenient,
and they are also small (32 bits typically suffices in practice).

## Why interning is difficult today: garbage collection

Unfortunately, integrating interning into salsa at present presents
some hard choices, particularly with a long-lived application. You can
easily add an interning table into the database, but unless you do
something clever, **it will simply grow and grow forever**. But as the
user edits their programs, some paths that used to exist will no
longer be relevant -- for example, a given file or impl may be
removed, invalidating all those paths that were based on it. 

Due to the nature of salsa's recomputation model, it is not easy to
detect when paths that used to exist in a prior revision are no longer
relevant in the next revision. **This is because salsa never
explicitly computes "diffs" of this kind between revisions -- it just
finds subcomputations that might have gone differently and re-executes
them.** Therefore, if the code that created the paths (e.g., that
processed the result of the parser) is part of a salsa query, it will
simply not re-create the invalidated paths -- there is no explicit
"deletion" point.

In fact, the same is true of all of salsa's memoized query values. We
may find that in a new revision, some memoized query values are no
longer relevant. For example, in revision R1, perhaps we computed
`foo(22)` and `foo(44)`, but in the new input, we now only need to
compute `foo(22)`. The `foo(44)` value is still memoized, we just
never asked for its value. **This is why salsa includes a garbage
collector, which can be used to cleanup these memoized values that are
no longer relevant.**

But using a garbage collection strategy with a hand-rolled interning
scheme is not easy. You *could* trace through all the values in
salsa's memoization tables to implement a kind of mark-and-sweep
scheme, but that would require for salsa to add such a mechanism. It
might also be quite a lot of tracing! The current salsa GC mechanism has no
need to walk through the values themselves in a memoization table, it only
examines the keys and the metadata (unless we are freeing a value, of course).

## How this RFC changes the situation

This RFC presents an alternative. The idea is to move the interning
into salsa itself by creating special "interning
queries". Dependencies on these queries are tracked like any other
query and hence they integrate naturally with salsa's garbage
collection mechanisms.

# User's guide

## Declaring an interned query

You can declare an interned query like so:

```rust
#[salsa::query_group]
trait Foo {
  #[salsa::interned]
  fn intern_path(&self, key: Path) -> u32;
]
```

**Query keys.** Like any query, these queries can take any number of keys. If multiple
keys are provided, then the interned key is a tuple of each key
value. In order to be interned, the keys must implement `Clone`,
`Hash` and `Eq`. 

**Return type.** The return type of an interned key may be of any type
that implements `salsa::InternIndex`: salsa provides impls for `u32`
and `usize`, but you can implement it for your own.

**Inverse query.** For each interning query, we automatically generate
a reverse query that will invert the interning step. It is named
`lookup_XXX`, where `XXX` is the name of the query. Hence here it
would be `fn lookup_intern_path(&self, key: u32) -> Path`.

## Using an interned query

## Interaction with the garbage collector

# Reference guide

Describe implementation details or other things here.

# Alternatives and future work

Various downsides, rejected approaches, or other considerations.


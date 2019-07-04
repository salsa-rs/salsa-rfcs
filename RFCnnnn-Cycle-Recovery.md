# Summary

- Permit queries to define their return value in the case of a cycle.

# Motivation

Presently, if one Salsa query Q has a cyclic dependency on itself, the
result is a panic. This is because the semantics of cycles are
complex, and forcing an error is a conservative path. The current
intention is that a cyclic query dependency is considered a bug; i.e.,
the program is meant to detect and avoid possible cycles. However, in
many applications of salsa, it is difficult or impossible to prevent
cycles, because they can arise as a result of invalid user input.

This RFC proposes an extension that allows salsa programmers to detect
and "recover" from cycles in the form of reporting a controlled error
to the user. It does not intend to offer a more general treatment of
cycles, which can be done in follow-up work.

## Cycles might be out of the program's control

Consider a simple compiler for a Rust-like language, but
one where return types can be inferred:

```
fn foo(x: u32) { bar(x) + 2 } // return type inferred
fn bar(x: u32) { x + 1 } // return type inferred
```

In such a scenario, you might want to enforce a requirement that
functions with inferred return types are arranged in a DAG.  This way,
while type-checking `foo` above, we would type-check `bar`, and that
query could return that its result type is `u32`, which allows us to
infer the return type of `foo`.

In such a system, though, an input like this would be illegal:

```
fn foo(x: u32) { bar(x) + 2 }
fn bar(x: u32) { if x == 0 { 0 } else { foo(x) + 1 } }
```

This is illegal because type-checking `foo` requires type-checking
`bar` which then in turn requires `foo`. (One can imagine languages
that *do* permit such cycles as well, but we are considering a simple
case here.)

This case could in theory be conveniently handled by salsa: we could
have a query for type checking a function (`type_check(id)`) that
returns the function signature. In the process of doing that
computation, if we encounter a call to a function `x`, we would then
invoke `type_check(x)`. So long as the functions do not recursively
invoke one another, everything will work out fine. But if a cycle
*does* occur, we don't want the salsa program to *panic*: we want to
detect the cycle and report the error to the user.

## Ad-hoc cycle recovery can lead to nondeterministic results

One way to handle cycle recovery would be to add a way to invoke a
query `Q` that returns a `Result<T, CycleError>` -- i.e., in the case
of a cycle, you get back an `Err` result which can be handled. Such a
scheme however can easily lead to nondeterministic results.

Consider a case from rustc, which does offer such a "try" function
that permits ad-hoc cycle recovery. In rustc, there is a query for
computing the optimized MIR (mid-level IR) for a function:

- `mir_optimized(F)` -- builds the MIR and then applies optimizations

Part of this optimization involves inlining calls into one another.
If you have a function `foo` that calls `bar`, then computing
`mir_optimized(foo)` would compute `mir_optimized(bar)` to find the
MIR for `bar`. If the MIR is small enough, it can be inlined directly
into `foo`. As above, so long as there is no cycle, this works out
fine -- but in the event of a cycle, there would be cyclic queries.

We initially decided to handle this by invoking the `mir_optimized` in
cycle-recovery mode for each callee. The idea is that *if* a cycle
results, we just decide not to inline -- it's not the optimal result
but "good enough".  However, this means that the optimized MIR will
depend on the order in which we invoke queries.

For example, imagine we invoke `mir_optimized(foo)` first:

- it will find the call to `bar` and invoke `mir_optimized(bar)`
- `mir_optimized(bar)` will in turn invoke `mir_optimized(foo)`, which
  will return `Err(CycleError)`
    - therefore, `mir_optimized(bar)` will not inline the body of
      `foo`
- but the optimized MIR for `foo` will include `bar`

Alternatively, now imagine we had invoked `mir_optimized(bar)` first:
 
- it will find the call to `foo` and invoke `mir_optimized(foo)`
- `mir_optimized(foo)` will in turn invoke `mir_optimized(bar)`, which
  will return `Err(CycleError)`
    - therefore, `mir_optimized(foo)` will not inline the body of
      `bar`
- but the optimized MIR for `bar` will include `foo`

Basically, whichever optimized MIR we compute first includes the body
of the other function, but not vice versa. Nondeterminstic results
like this are a problem for salsa. Ad-hoc cycle recovery, however,
serves as a kind of "footgun" that makes such non-deterministic
results very easy.

(If you'd like to know a way to handle inlining that avoids cycles,
see the Appendix.)

# User's guide

Describe effects on end users here.

# Reference guide

Describe implementation details or other things here.

# Alternatives and future work

Various downsides, rejected approaches, or other considerations.

# Appendix: How to handle error recovery

You may be curious


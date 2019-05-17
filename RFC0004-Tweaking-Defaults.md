# Summary

Adjust query type names, semantics and defaults to be a better fit for
interactive IDE scenarios.

# Motivation

Salsa provides really impressive *incremental* computation features via
dependency tracking. However, for most IDE tasks a simpler *lazy* computation +
caching works perfectly. For example, many IDEs do per-function or
per-expression type inference lazily, but not incrementally.

At the same time, memory usage is always important, and while GC is useful to
collect garbadge eventually, a more deterministic memory reclamation strategy
can help with memory high water mark.

Additionally, current salsa terminology confuses at least some users.
Anecdotally, @matklad was befooled by the semantics of volatile queries at least
couple of times, despite being familiar with the implementation

# User's guide

This RFC proposes the following set of query types

* `input` -- the same as today's `input`
* `cached` -- classical memoization. If a cached query is called twice withing a
  single revision, it is computed only once. If the query is called in two
  different revisions, it is computed both times. Additionally, when revision
  advances all cached values are eagerly collected. Dependency information for
  cached queries is preserved across revisions, so cached queries do not
  automatically invalidate dependent queries. `cached` is the default salsa
  mode.
* `tracked` -- the same as today's default. Values of tracked queries are
  preserved across revisions, so tracked queries can be used to implement
  computation firewalls. Salsa "tracks" the value of a tracked query, and
  notices when it actually changes.
* `uncached` -- the same as today's transparent, just a usual function call,
  without any memoization whatsoever.

# Reference guide

The main difference with the status quo, besides the names, is that we don't
have a direct equivalent of a `volatile` query type. While `cached` looks
similar to `volatile`, it tracks dependencies and does not invalidate dependent
queries.

For example, if we have a `Tracked -> Cached -> Input` dependency chain and the
global revision advances (without changing this particular `I`), we drop the
value of `C`. However, if we then ask for `T`, we just re-validated `T` and `C`
and reuse the old value. If we explicitly ask for `C`, we recompute the value
and cache it.

The current semantics of volatile can be simulated by a `tracked` query with a
dummy input, which the user sets explicitly when needed.

# Alternatives and future work

We can preserve the `volatile` query type in addition to the proposed queries.
However, it is unclear if `volatile` really needs a dedicated query type, given
that it is rarely needed and can be misused.

Because of the dependency tracking, cached queries are **less** efficient than
traditional memoization. We however need dependencies to allow tracked queries
to invoke cached queries in a sound way. If a cached query is **not** used by
any tracked query, we can safely forgo recording dependencies. We can imagine an
`#[salsa::cached(deps = false)]` attribute which can be applied to a `cached`
query do completely disable dependency tracking. Calling such a query from a
tracked one should result in a run-time or compile-time error.

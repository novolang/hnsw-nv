# Changelog

Every published version, newest first.  This file is on the publish
allow-list, so it travels with the package: it is the only thing a
consumer deciding whether to upgrade can read.

## 0.0.1 — 2026-09-11

The **interface**, before anyone implements it.  Every signature, every
type and every effect row is published; every body is `todo()`, and the
release is stamped `NOT IMPLEMENTED — interface only`.  Adding this
package works and calling it panics.

- Six modules and no dependencies.  `hnswgraph` is the index,
  `hnswsearch` the query, `hnswparam` the build parameters,
  `hnswdist` the three distances, `hnswio` the serialised form and
  `hnswerr` the fault.
- **The uniform is an argument, and that is the layer's doing.**
  rand-nv is `host` because the entropy comes from the machine, and a
  `core` package may not reach a `host` one — so
  `hnswgraph.insert(ix, label, v, u)` takes one number in [0, 1) and
  `hnswparam.level_of` turns it into a level.  What that bought: a
  build is a pure function of the vectors and the uniform sequence, so
  the same inputs build the same graph on any machine, and a
  serialised index is comparable with the one that produced it.
  `insert_at_level` is the same call with the draw already done.
- **An index is a value.**  Every function takes one and returns one;
  an index can be snapshotted, handed on or rolled back, and the
  in-place update is Perceus's job when the caller holds the only
  reference.
- **A row is a `[Float]`**, not an `NdFloat`: an index holds N rows of
  one dimension it already knows, and the hot path wants two
  contiguous buffers.  embeddings-nv is where a caller coming from a
  model converts, in one line.
- **Everything is a distance and smaller is nearer**: L2 squared,
  cosine as `1 - cos`, inner product as `1 - dot`.  The squaring is
  deliberate — the root changes no ordering and costs a call per
  comparison — and the package does not normalise vectors on a
  caller's behalf, because an index that did would hand back a vector
  nobody stored.
- **`ef` below `k` is refused** rather than quietly returning fewer
  results, and `search_filtered` applies its predicate DURING the
  descent, where `ef` is the budget, rather than after it, where the
  result is short.
- **A delete is a mark**: the element stays a hop and stops being a
  result.  `insert_replacing` reclaims the slot and `drift` says how
  far the index has moved from a fresh build.
- **The serialised form is not hnswlib's.**  hnswlib writes its own
  memory with no magic, no version and no dimension; this format has
  all three, states every width and byte order, and runs the graph
  check on every read — a corrupt HNSW does not crash, it answers.
- **`search_layer` is public**, because it is the algorithm and a
  caller doing a sharded search, a two-stage re-rank or a recall
  harness should not have to write it again.
- No `@tier(embedded)` claim, and the README says why.
- `tests/` holds fifteen API tests across two files, every one red.
  The distances are computed by hand in the assertions, the level
  formula is the paper's with four uniforms chosen to land in the
  first four levels, and hnswlib over SIFT1M and GIST1M becomes a
  generated run beside them when the bodies land.
- The README's "What usearch does differently" names the one thing
  this interface has no room for — quantised STORAGE, because
  `HnswIndex.data` is a `[Float]` — and asks for the decision to be
  taken before the first body rather than after it.

# Changelog

Every published version, newest first.  This file is on the publish
allow-list, so it travels with the package: it is the only thing a
consumer deciding whether to upgrade can read.

## 0.1.0 — 2026-09-27

The first implementation of the interface published as 0.0.1: the
paper's insert and search with hnswlib's neighbour heuristic, delete by
mark with slot reuse, the three metrics, and a checked serialised form.

### Changed, breaking

- `HnswIndex` gains two fields.  `link_starts` holds where each
  element's run of links starts, so an element's links take room for
  its own levels only.  `replaced` counts the slots `insert_replacing`
  has reused, which `drift` reads.  A caller building an `HnswIndex`
  literal adds both.
- `hnswio.write_into` takes its buffer as `var into`.  Under 0.13.0's
  list rule a function writes into a caller's buffer only through a
  `var` parameter.
- A uniform of 0.0 is refused as well as 1.0: the interval is (0, 1),
  as `level_of`'s comment already said of 0.0.

### Behaviour the interface left open

- The insert is Algorithm 1 of Malkov and Yashunin with hnswlib's
  `getNeighborsByHeuristic2` as the neighbour selection: `M` links for
  the new element at every level, and a full neighbour keeping the best
  of its links and the new one by the same heuristic.  Marked elements
  are walked through during an insert and not linked to, as in hnswlib.
- The search at level 0 keeps walking while it holds fewer than `ef`
  accepted answers, as hnswlib's `searchBaseLayerST` does, so a filter
  or a mark does not shorten a result while enough accepted elements are
  reachable.  Ties are broken by slot everywhere.
- `insert_replacing` keeps the reused slot's level, as hnswlib's
  `updatePoint` does, and reuses a marked slot carrying the same label
  before any other.
- `check` does not require links to be reciprocal, since the neighbour
  heuristic removes back-links; it checks lengths, levels, run starts,
  counts, targets and labels, and names each.
- `drift` is `(marked + replaced) / count`, at most 1.0.
- `expected_at_level` is `count * exp(-level / mL)`, which is
  `count * M^-level` at the default multiplier.
- `element_bytes` is the element's cost in the serialised form.
- `expected_comparisons` is the mean degree at each upper level plus
  `ef` times the mean degree at level 0.
- The serialised form is laid out in `hnswio`'s module comment: a
  48-byte header with the magic "NVHW", then the fixed fields, the
  vectors as 64-bit floats and the graph.  `from_bytes` checks every
  size against the total before it reads, and runs `check`.
- `metric_of_code` answers `HnswBadParam` naming "metric" for an
  unknown code.

### Tests

- 27 tests in four suites.  The exact answer is the recall reference:
  recall@10 of at least 0.95 at `ef` 64 over 2,000 seeded vectors of 24
  coordinates, and at least 0.9 with marks, replacements and a filter.
  The paper's invariants are asserted over the same build, two builds
  from the same uniforms serialise to the same bytes, and every
  corruption `check` and `from_bytes` look for is refused by name.
- Every line under `src/` is executed by the suites;
  `bash tests/coverage.sh` prints the number.

## 0.0.2 — 2026-09-15

README rewritten to the package README style guide (docs/writing-a-readme.md); no change to the interface.

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

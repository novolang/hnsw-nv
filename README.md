# hnsw-nv

A nearest-neighbour index answers the question "which of these million
vectors is closest to this one" without comparing all of them. Hierarchical
Navigable Small World is the structure almost every vector database uses to
do it, described in Malkov and Yashunin,
[*Efficient and robust approximate nearest neighbor search using
Hierarchical Navigable Small World graphs*](https://arxiv.org/abs/1603.09320)
(2018). This package brings it to novo-lang as a value: an index is built,
searched, copied, written out and read back, and nothing in it is a handle.
The reference implementation is
[hnswlib](https://github.com/nmslib/hnswlib).

**Status: NOT IMPLEMENTED — interface only.** Every function is declared
with its full signature, but every body is a `todo()` that panics when
called. The package is published so its design can be reviewed and
depended on before it is implemented. Version 0.1.0 will be the first
working release.

## What HNSW is

An **element** is one vector with a **label**, an identifier the caller
chooses. The index holds every element's vector in one buffer and a graph
of links between them.

The graph is a stack of **levels**. Level 0 holds every element. Each level
above holds a thinning sample of the level below, so the top level has a
handful of elements and the bottom has all of them. A search starts at the
top with one entry point, walks greedily downhill to the nearest element it
can reach on that level, drops to the level below, and repeats. By the time
it reaches level 0 it is already near the answer.

Which level an element joins is drawn at random from a geometric
distribution: `floor(-ln(u) * mL)`, where `u` is a uniform random number in
[0, 1) and `mL` is the **level multiplier**. The paper recommends
`mL = 1 / ln(M)`, which makes the expected work per query grow with the
logarithm of the number of elements.

Three numbers shape an index. **M** is how many links each element keeps
per level, and it trades memory for recall. **ef_construction** is how many
candidates the builder considers while linking a new element, and it trades
build time for graph quality. **ef** is how many candidates a search keeps
as it descends, and it trades query time for how many of the true nearest
neighbours come back. **k** is only how many results the caller wants.

**Recall** is the fraction of the true nearest neighbours a search actually
found. HNSW is approximate, so recall is below 1, and `ef` is the one knob
that moves it at query time.

Every comparison this package makes is a **distance**, and smaller is
nearer, because the priority queues are minima and the descent walks
downhill.

| Metric | Is | Range |
| --- | --- | --- |
| `HnswL2` | `sum((a - b)²)` | 0 and above |
| `HnswCosine` | `1 - cos(a, b)` | 0 to 2 |
| `HnswInner` | `1 - dot(a, b)` | unbounded, and may be negative |

The defaults, which are hnswlib's:

| Parameter | Default |
| --- | --- |
| `m` | 16 |
| `ef_construction` | 200 |
| `capacity` | 1024 |
| Links per element, level 0 | 2 × `m` |
| Links per element, above level 0 | `m` |
| `m`, allowed range | 2 to 128 |
| Serialised header | 48 bytes |
| Serialised float width | 8 bytes |

This package performs no input or output. It opens no file, starts no
thread, reads no clock and draws no random numbers. Every effect row in it
is empty.

## Install

```
novo pkg add hnsw-nv
```

## Example

```novo
use hnswparam
use hnswgraph
use hnswsearch

// Build a two-dimensional index over three points and answer the one
// nearest a query. The uniform per insert is the caller's: the index
// turns it into the level the element joins at, and writing the
// numbers down makes the graph the same on every run.
fn nearest(query: [Float]) -> Result<[HnswHit], HnswFault>
    let p = hnswparam.default_params(2, HnswL2)!
    var ix = hnswgraph.index(p)
    ix = hnswgraph.insert(ix, 1, [0.0, 0.0], 0.11)!
    ix = hnswgraph.insert(ix, 2, [1.0, 0.0], 0.42)!
    ix = hnswgraph.insert(ix, 3, [0.0, 1.0], 0.73)!
    let hits = hnswsearch.search(ix, query, 1, 10)!
    Ok(hits)

fn main() [io]
    match nearest([0.9, 0.1])
        Err(f)   => println(f.message())
        Ok(hits) =>
            for h in hits
                // The label is the id this program gave the point, and
                // the distance is under the index's metric, where
                // smaller is nearer.
                println("${h.label} at ${h.distance}")
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a `not implemented: hnsw-nv.<module>.<fn>`
panic. The tests are the specification the implementation will have to
satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `hnswparam` | The build parameters and their bounds, the level multiplier, and the formula that turns a uniform into a level. |
| `hnswdist` | The three metrics, each distance on its own, and the vector helpers: the norm, normalisation and the finiteness check. |
| `hnswgraph` | The index as a value: insertion, the delete mark, slot reuse, growth, and the readers that look inside the graph. |
| `hnswsearch` | Searching: k-nearest, k-nearest with a filter, one level's walk, the exact answer, and the recall and cost measurements. |
| `hnswio` | The serialised form: a header, an exact size, the bytes, and a reader that checks the graph before it answers. |
| `hnswerr` | Every reason a call refuses, split into the caller's mistakes and a file's faults. |

## How to choose an entry point

**`hnswgraph.insert` takes a uniform and draws the level from it.** Use it
when you are building an index from a random number generator.
**`hnswgraph.insert_at_level` takes the level outright.** Use it when
rebuilding from a recorded level sequence, and in a test that wants a graph
of a known shape. `insert` is `hnswparam.level_of` in front of
`insert_at_level`, and both are public.

**`hnswsearch.search` takes `ef`.** Use it once you have measured one.
**`search_default`** picks `max(k, ef_construction / 4)`, which is a
starting point for a caller that has measured nothing yet.
**`search_filtered`** takes a predicate over labels and applies it during
the descent.

**`hnswsearch.brute_force` is the exact answer**, at a cost linear in the
index. It is not a fallback; it is what `recall_at` compares an approximate
answer against when you are choosing `ef`.

**`hnswsearch.search_layer` is one level's walk**, and it is public because
it is the algorithm. A caller doing a sharded search, a two-stage re-rank
or a recall harness should not have to write it again.

**`hnswio.peek_header` reads 48 bytes.** It tells a cache whether a stored
index is still the right one for its model without parsing gigabytes behind
it. `from_bytes` reads the whole thing.

## The rules a user needs

1. **The uniform is yours, and it is the whole of the randomness HNSW
   needs.** `hnswgraph.insert` takes one number in [0, 1) per element, and
   `hnswparam.level_of` applies the paper's Algorithm 1 formula
   `floor(-ln(u) * mL)`. A build is therefore a pure function of the
   vectors and the uniform sequence: the same inputs build the same graph
   on any machine. A caller with a random number generator passes
   `rng.next_float()`.
2. **An index is a value and every call answers a new one.** Writing
   `ix = hnswgraph.insert(ix, id, v, u)!` in a loop lets the runtime update
   in place, because the caller holds the only reference. A caller that
   keeps the old index gets a real copy, which is what makes a snapshot and
   a rollback possible.
3. **Smaller is nearer, everywhere in this package.** The two
   similarity-shaped metrics are stored as `1 - similarity`, so every
   comparison is a minimum. `embeddings-nv` uses the opposite convention
   and `embsim.as_distance` is the conversion.
4. **`HnswL2` is the squared distance, on purpose.** The square root is
   monotonic, so taking it changes no ordering, no queue and no graph, and
   it costs a call in the hottest loop there is. `hnswdist.l2` is the real
   distance, for reporting one to a person.
5. **Inner product is not a metric on vectors that are not unit length.**
   It breaks the triangle inequality, so the greedy descent has no proof
   behind it and recall becomes something to measure rather than something
   bounded. On normalised vectors it is the cosine distance and is cheaper,
   because it skips two norms.
6. **Nothing here normalises for you.** An index that normalised what it
   was given would hand back a vector the caller never stored.
   `hnswdist.wants_normalized` says which metrics expect it, and
   `hnswdist.normalize` is one call at the point where the caller knows.
7. **`ef` below `k` is refused.** Returning fewer results than were asked
   for is invisible at the call site and shows up as a product that is
   quietly worse.
8. **Measure `ef` on your own data.** `hnswsearch.brute_force` is the exact
   answer, `recall_at` compares an approximate answer with it, and
   `expected_comparisons` is the cost model. The number that comes out is a
   property of the data rather than of this package, which is why there is
   a measurement here and no recommendation.
9. **Filtering happens during the descent.** `search_filtered` takes a
   named function over labels, not a closure, and applies it as it walks.
   Filtering afterwards gives fewer than `k` results whenever the filter is
   at all selective. What it cannot promise: a filter that accepts almost
   nothing turns the search into a walk of the whole graph, and a caller
   whose filters are that selective wants one index per partition.
10. **A delete is a mark.** `mark_deleted` sets a flag; the element keeps
    its vector and its links, is skipped as a result, and is still walked
    through as a hop. Removing an element outright would cut the graph, and
    repairing the cut means re-running the neighbour selection for
    everything that pointed at it.
11. **A replaced slot does not give the graph a fresh build would.**
    `insert_replacing` reuses a deleted element's slot and its links as a
    starting point. `hnswgraph.drift` says how far the index has moved from
    a fresh build, and past about half of it a rebuild wins.
12. **An index refuses a duplicate label**, and it refuses a vector of the
    wrong length, one holding a value that is not a number, and a zero
    vector under cosine.
13. **Capacity is raised explicitly.** An insert past `capacity` is
    `HnswAtCapacity` rather than a silent reallocation, and
    `hnswgraph.grow` is the call. The copy is the expensive thing this
    structure does, and hiding it would hide the cost of having sized the
    index wrong.
14. **Search an index under the parameters it was built with.** They are
    fixed at construction and serialised with the index, because every one
    of them shaped the graph.
15. **A stored index is not hnswlib's file.** hnswlib writes its own memory
    with no magic, no version and no dimension, so a file read by the wrong
    build produces an index rather than an error. `hnswio` writes a magic,
    a version and every parameter at a stated width and byte order, and
    `from_bytes` runs `hnswgraph.check` before it answers. A corrupt HNSW
    does not crash: it answers, worse.
16. **Stored vectors are 64-bit floats.** A million 768-element vectors is
    6 GB. Writing them narrower would round every vector on the way out and
    produce an index that is not the one that was saved. A caller that
    minds quantises before inserting, or keeps the vectors outside the
    index and stores identifiers alone.

## What is not included

- **Randomness.** The uniform is a parameter. The entropy comes from the
  machine, and this package reaches nothing that touches it.
- **Threads.** An index is a value, so a caller that wants two searches at
  once hands the same value to both.
- **A file.** `hnswio` answers bytes and takes bytes. Writing them down is
  the caller's.
- **Quantised storage.** `HnswIndex.data` is a list of 64-bit floats.
  Storing vectors as bytes or bits, which is where usearch gets most of its
  memory advantage, would be a different field. The metric enum has room
  for another variant; the storage does not.
- **A memory-mapped index.** Reading a stored index without loading it
  needs machine effects. [vectorstore-nv](https://novo-lang.org/packages/vectorstore-nv)
  is the package that owns those.
- **A user-defined metric.** The three above are the ones the graph is
  defined over here.
- **A microcontroller build.** No module is declared to build for a device
  with no heap allocator. An index of any useful size is megabytes of heap.

## Related packages

- [embeddings-nv](https://novo-lang.org/packages/embeddings-nv) produces
  the vectors an index holds: pooling, normalisation, Matryoshka truncation
  and quantisation. It speaks `NdFloat`, because its subject is a
  token-embedding matrix. This package takes a row as a `[Float]`, because
  its subject is a graph over rows of one dimension it already knows.
  `ndfloat.to_list` of one row is the seam. The two packages compute the
  same three distances and share no code.
- [vectorstore-nv](https://novo-lang.org/packages/vectorstore-nv) keeps
  documents beside their vectors, writes them down and serves them. It is
  the host layer an index like this one sits inside.
- [ndarray-nv](https://novo-lang.org/packages/ndarray-nv) is the
  n-dimensional array. This package does not depend on it, so a program
  that wants an index does not download an array library to get one.
- `std.store` in the standard library is an in-memory vector store for
  retrieval-augmented generation. It compares every vector, which is the
  right thing up to a few thousand of them and the wrong thing past that.
- `std.vec` in the standard library is dot, cosine and norm over a list of
  floats, for a caller doing its own arithmetic on a handful of vectors.

## Tests

```bash
novo test tests/hnswdist_tests.nv    # 7 tests: the three metrics and the vector helpers
novo test tests/hnswindex_tests.nv   # 8 tests: insertion, search, deletion, the file
```

The distances are computed by hand and written out beside each assertion.
The level formula is the paper's, with four uniforms chosen to land in the
first four levels. The uniforms are written down rather than drawn, which
is what the design buys: the graph is the same on every machine and every
run, so an assertion about an edge is an assertion rather than a flake.

hnswlib is the oracle, and it arrives with the bodies: the same vectors,
the same parameters and the same level sequence, compared edge for edge,
with recall@10 over the SIFT1M and GIST1M query sets compared at four
values of `ef`.

The tests compile today and fail at run, each on the
`not implemented: hnsw-nv.<module>.<fn>` panic that is its body. That is
the expected state of an interface release. They turn green one at a time
as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| `hnswdist.metric_name`, `.metric_of_name`, `.metric_code`, `.metric_of_code`, `.wants_normalized` | no |
| `hnswdist.distance`, `.l2_squared`, `.l2`, `.cosine`, `.inner`, `.dot` | no |
| `hnswdist.norm`, `.normalize`, `.is_finite_vector` | no |
| `hnswparam.params`, `.default_params`, `.with_level_multiplier`, `.with_capacity` | no |
| `hnswparam.default_level_multiplier`, `.level_of`, `.max_links`, `.default_ef` | no |
| `hnswparam.element_bytes`, `.expected_at_level` | no |
| `hnswgraph.index`, `.count`, `.live_count`, `.deleted_count`, `.params_of` | no |
| `hnswgraph.insert`, `.insert_at_level`, `.insert_replacing`, `.grow` | no |
| `hnswgraph.mark_deleted`, `.unmark_deleted`, `.is_deleted` | no |
| `hnswgraph.slot_of`, `.label_at`, `.vector_of`, `.vector_base`, `.level_of_label` | no |
| `hnswgraph.link_base`, `.neighbours`, `.degree`, `.level_histogram`, `.check`, `.drift` | no |
| `hnswsearch.search`, `.search_default`, `.search_filtered` | no |
| `hnswsearch.search_layer`, `.greedy_step`, `.brute_force`, `.distance_to` | no |
| `hnswsearch.recall_at`, `.hit_labels`, `.expected_comparisons` | no |
| `hnswio.magic_le`, `.format_version`, `.supported_versions`, `.header_bytes` | no |
| `hnswio.size_bound`, `.to_bytes`, `.write_into` | no |
| `hnswio.peek_header`, `.from_bytes`, `.is_index`, `.header_matches` | no |
| `hnswerr.is_usage`, `.is_format`, `HnswFault.message` | no |

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->

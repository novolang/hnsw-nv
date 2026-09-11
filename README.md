# hnsw-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`.  Installing this package works;
calling it panics with `not implemented`.

## What this is

**Hierarchical Navigable Small World**: the approximate
nearest-neighbour index behind every vector database, as a **value**.
Build it, search it, mark elements deleted, copy it, serialise it, read
it back — with no threads, no file, no clock and no entropy of its own.

Six modules, and no dependencies.

| surface | module | reach for it when |
| --- | --- | --- |
| the **index** | `hnswgraph` | you are building one, or looking inside it |
| the **search** | `hnswsearch` | you have a query |
| the **parameters** | `hnswparam` | you are choosing M and ef, or measuring what they cost |
| the **distances** | `hnswdist` | you are choosing a metric, or doing your own arithmetic |
| the **file** | `hnswio` | you are caching an index between runs |
| the **faults** | `hnswerr` | you are telling somebody why their query did not run |

## Adding it, and checking it

```bash
novo pkg add hnsw-nv               # into your novo.toml
novo pkg build                     # type- and effect-check the package
novo test --isolate tests/hnswindex_tests.nv
```

`novo test` is red today and that is the point of the release: every
assertion fails with `not implemented: hnsw-nv.<module>.<fn>`.  They
turn green one at a time as bodies land.

## The one example that will work

```novo
use hnswgraph
use hnswparam

fn main() [io]
    match hnswparam.default_params(4, HnswL2)
        Err(f) => println(f.message())
        Ok(p)  => println("${hnswgraph.count(hnswgraph.index(p))}")
                  // : 0
```

## The load-bearing interface

```novo norun:pseudo
pub fn insert(ix: HnswIndex, label: Int, v: [Float], u: Float)
    -> Result<HnswIndex, HnswFault>
```

**`u` is one uniform in [0, 1), and it is the whole of the randomness
HNSW needs.**

HNSW is a stack of graphs: level 0 holds everything and each level above
holds a geometrically thinning sample, so a search starts at the top
with a handful of elements and descends.  Which level an element joins
is drawn from a geometric distribution — `floor(-ln(u) * mL)` — and
hnswlib holds a `std::mt19937` inside the index to draw it.

This package cannot.  **rand-nv is `host`**, because the entropy comes
from the machine, and a `core` package may not depend on a `host` one.
So the draw is the caller's and the arithmetic is the index's:
`hnswparam.level_of` turns `u` into a level, and every other thing an
insert does is deterministic.  fake-nv took the same decision for the
same reason, and stats-nv takes its uniforms as arguments too.

**What the layer took, and what it gave back.**  It took the
convenience of an index that seeds itself — a caller now writes
`ix = insert(ix, id, v, rng.next_float())!` and holds its own RNG.  It
gave **reproducibility**: a build is a pure function of the vectors and
the uniform sequence, so the same inputs build the same graph on any
machine in any process.  That is what makes a serialised index
comparable with the one that produced it, what makes an assertion about
an edge an assertion rather than a flake, and what turns "recall dropped
after that change" from an argument into a bisect.

`insert_at_level` is the same function with the draw already done, for
the two callers that have a level rather than a number: a rebuild from
a serialised index, and a test that wants a graph of a known shape.

## An index is a value, and copying one is a copy

Every function takes an index and returns one; nothing mutates.  That is
what a `core` package with no interior mutability can offer, and it has
one consequence worth sizing before you start: **an insert copies the
index**.  The implementation's job is to make that a Perceus in-place
update when the caller holds the only reference, which is what

```novo norun:pseudo
var ix = hnswgraph.index(p)
for i in 0..n
    ix = hnswgraph.insert(ix, ids[i], vecs[i], us[i])!
```

gives it.  A caller that keeps the old index around gets a real copy —
and being *able* to is the point.  An index you can snapshot, hand to
another thread, or roll back is a different thing from one you cannot.

## Why a row is a `[Float]` and not an `NdFloat`

An index holds N rows of **one** dimension, which it already knows from
its own parameters.  An `NdFloat` per row would carry a shape, a stride
set and an offset — three integers and a struct restating a fact the
index holds once — and the hot path here is a single walk down two
contiguous buffers, which a strided view would have to be packed into
first.

It also keeps the dependency list empty, which matters more than it
sounds: a program that wants a vector index should not download an array
library to get one.

The seam is one line, and **embeddings-nv** is where a caller coming
from a model meets it:

```novo norun:pseudo
let row = ndfloat.to_list(ndfloat.index_axis(matrix, 0, i)!)
ix = hnswgraph.insert(ix, id, row, u)!
```

embeddings-nv does speak `NdFloat`, because its subject is a
token-embedding matrix and a shape is what that is.  This package's
subject is a graph.

## Everything is a distance, and smaller is nearer

HNSW compares, and every comparison it makes is a minimum: the priority
queues are minima and the greedy descent walks downhill.  So the two
similarity-shaped metrics are defined as their distance forms, in one
place, rather than inverted at every call site:

| metric | is | range |
| --- | --- | --- |
| `HnswL2` | `sum((a - b)^2)` | `[0, ∞)` |
| `HnswCosine` | `1 - cos(a, b)` | `[0, 2]` |
| `HnswInner` | `1 - dot(a, b)` | unbounded, and may be negative |

**L2 is squared on purpose.**  The square root is monotonic, so taking
it changes no ordering, no queue and no graph — and it costs a call per
comparison in the hottest loop there is.  `hnswdist.l2` is there for a
caller reporting a real distance to a person.

**Inner product is not a metric**, for vectors that are not unit length:
it breaks the triangle inequality, so the greedy descent has no proof
behind it and recall becomes an empirical question rather than a bounded
one.  hnswlib makes the same choice, calls it `InnerProductSpace`, and
it works well in practice — on normalised vectors, where it *is* the
cosine distance and is cheaper because it skips two norms.  On
unnormalised ones, measure.

**This package does not normalise for you.**  An index that normalised
what it was given would answer a cosine query correctly and hand back a
vector the caller did not store, and a caller that wanted L2 over
magnitudes would find them gone.  `hnswdist.normalize` is one call, at
the point where the caller knows whether it wants it, and
`hnswdist.wants_normalized` is what the package will say instead.

## `ef` is the knob, and it is the only per-query one

Everything else is fixed when the index is built.  A search keeps a
candidate list of `ef` elements as it descends; larger finds more and
costs more, and `k` is only how many of them come back.

**`ef` below `k` is refused**, rather than quietly returning fewer
results than were asked for — that failure is invisible at the call
site and shows up as a product that is subtly worse.

To choose one: `hnswsearch.brute_force` is the exact answer,
`hnswsearch.recall_at` compares it with an approximate one, and
`hnswsearch.expected_comparisons` is the cost model.  Run both over a
sample of real queries and read the curve; the number that comes out is
a property of the data, not of the library, which is why this package
ships the measurement rather than a recommendation.

**Filtering happens during the descent**, not after it.
`search_filtered` takes a named `fn(Int) -> Bool` over the caller's own
labels.  Filtering afterwards gives fewer than `k` results whenever the
filter is at all selective, and the usual workaround — ask for ten times
as many — costs ten times as much and still has no bound.  What a
during-the-descent filter cannot promise: a filter that accepts almost
nothing turns the search into a walk of the graph, and a caller whose
filters are that selective wants one index per partition.

## A delete is a mark

`mark_deleted` sets a flag.  The element keeps its vector and its links;
searches skip it as a **result** and still walk **through** it as a hop.

That is hnswlib's behaviour and it is not a shortcut.  Removing an
element cuts the graph, and repairing the cut means re-running the
neighbour selection for everything that pointed at it — at which point a
rebuild is cheaper and better.  `insert_replacing` reclaims the slot,
`hnswgraph.drift` says how far the index has moved from a fresh build,
and past about half, rebuilding wins.

## The file is not hnswlib's file

hnswlib's `saveIndex` writes its own memory: the element size in bytes,
then the raw per-element blocks as the process laid them out.  It is
specific to the pointer width, the endianness and the library version of
the machine that wrote it, and it carries **no magic, no version and no
dimension** — so a file read by the wrong build produces an index rather
than an error.

`hnswio` writes a magic, a version, the parameters in full, and every
field at a stated width in a stated byte order.  `peek_header` reads 48
bytes and tells a cache whether the entry is still for its model without
parsing six gigabytes behind it.  A read runs `hnswgraph.check` before
it answers, because a corrupt HNSW does not crash — it answers, worse.

**The floats are 64-bit**, and that is the format's one extravagance: a
million 768-element vectors is 6 GB rather than 3.  Writing f32 would
round every vector on the way out and produce an index that is not the
one that was saved.  A caller that minds has two better answers:
quantise before inserting, with embeddings-nv, so the index stores what
it will compare; or keep the vectors outside and index ids alone.

## What usearch does differently, and whether there is room

usearch's headline difference is **quantised distances**: it stores
vectors as f16, i8 or single bits and computes the metric in that
representation, which is where most of its speed and nearly all of its
memory advantage come from.  It also does hardware SIMD dispatch, memory
maps an index rather than loading it, and supports user-defined metrics.

Of those, three fit inside this interface without changing it.  SIMD
dispatch is an implementation detail of `hnswdist`.  A memory-mapped
index is a `host` package over `hnswio`'s format — the row for it is
**vectorstore-nv**, already on the grid.  A user-defined metric is a
named `fn([Float], [Float]) -> Float` and would be a variant on
`HnswMetric` carrying one.

**Quantised storage does not fit**, and this package says so rather than
implying otherwise.  `HnswIndex.data` is a `[Float]`; storing i8 or bits
means a different field, and changing a public field is a breaking
change.  So the honest position for 0.0.1 is: the *distance* enum has
room for another variant, the *storage* does not, and the two ways
forward are a scalar-kind field on `HnswIndex` at 0.1 — decided before
the first body, while it is still free — or a separate binary index over
embeddings-nv's Hamming distance, which is a different structure and
probably a different row.  **This lane's report asks for that decision
rather than making it.**

## The layer, and the claim this package does not make

`core`, and every effect row in the package is empty.

**No `@tier(embedded)` claim.**  An index of any useful size is
megabytes of heap, the construction allocates, and the consumer is a
server or a notebook.  The audit's `core-embedded` row passes and says
the package makes no claim.

## Naming

Every public type and every enum variant starts `Hnsw`, and every module
file starts `hnsw`.  Struct and enum identity is keyed by **name** across
a whole assembly, dependencies included, so two packages that both
declare `Metric` cannot be used by one program.  The rule covers variant
names, which is why the metrics are `HnswL2` and `HnswCosine` rather
than the bare nouns — embeddings-nv computes the same three distances
and would collide on every one of them.

## What is the algorithm, and what is this package's choice

**Malkov and Yashunin's paper and hnswlib, and binding**: the level
formula and its `1 / ln(M)` multiplier; `2M` links at level 0 and `M`
above; `ef_construction` during a build and `ef` during a search; the
greedy descent with `ef = 1` through the upper levels; the neighbour
heuristic; and delete-by-mark with slot replacement.

**This package's choice**: the uniform as an argument (the layer's
doing, and reproducibility is the compensation); `[Float]` rows;
refusing `ef < k` rather than returning short; refusing a zero vector
under cosine rather than picking a number; refusing a duplicate label;
a magic and a version on the serialised form; running the graph check
on every read; and making `search_layer` public, because it is the
algorithm and a caller doing something the top-level API does not cover
should not have to write it again.

## The reference implementation

[hnswlib](https://github.com/nmslib/hnswlib), and Malkov and Yashunin,
*Efficient and robust approximate nearest neighbor search using
Hierarchical Navigable Small World graphs* (2018).
[usearch](https://github.com/unum-cloud/usearch) is the second reading,
and the section above says where the two designs part.

The oracle arrives with the bodies: hnswlib built over the same vectors
with the same parameters and the same recorded level sequence, its graph
compared edge for edge, and recall@10 over the SIFT1M and GIST1M query
sets compared at four values of `ef`, as a generated run beside the two
suites in `tests/`.

Apache-2.0.

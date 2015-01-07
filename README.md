One of Haskell's strengths is immutable data structures. These structures make
it easier to reason about code, simplify concurrency and parallelism, and in
some case can improve performance by allowing sharing. However, there are still
classes of problems where mutable data structures can both be more convenient,
and provide a performance boost. This library is meant to provide such
structures in a performant, well tested way. It also provides a simple
abstraction over such data structures via typeclasses.

We'll first talk about the general approach to APIs in this package. Next,
there are two main sets of abstractions provided, which we'll cover in the
following two sections, along with their concrete implementations. Finally,
we'll cover benchmarks.

## API structure

The API takes heavy advantage of the `PrimMonad` typeclass from the primitive
package. This allows our data structures to work in both `IO` and `ST` code.
Each data structure has an associated type, `MCState`, which gives the
primitive state for that structure. For example, in the case of `IORef`, that
state is `RealWorld`, whereas for `STRef s`, it would be `s`. This associated
type is quite similar to the `PrimState` associated type from primitive, and in
many type signatures you'll see an equality constraint along the lines of:

```haskell
PrimState m ~ MCState c
```

For those who are wondering, `MCState` stands for "mutable container state."

All actions are part of a typeclass, which allows for generic access to
different types of structures quite easily. In addition, we provide type hint
functions, such as `asIORef`, which can help specify types when using such
generic functions. For example, a common idiom might be:

```haskell
ioref <- fmap asIORef $ newRef someVal
```

Wherever possible, we stick to well accepted naming and type signature
standards. For example, note how closely `modifyRef` and `modifyRef'` match
`modifyIORef` and `modifyIORef'`.

## Single cell references

The base package provides both `IORef` and `STRef` as boxed mutable references,
for storing a single value. The primitive package also provides `MutVar`, which
generalizes over both of those and works for any `PrimMonad` instance. The
`MutableRef` typeclass in this package abstracts over all three of those. It
has two associated types: `MCState` for the primitive state, and `RefElement`
to specify what is contained by the reference.

You may be wondering: why not just take the reference as a type parameter? That
wouldn't allow us to have monomorphic reference types, which may be useful
under some circumstances. This is a similar motivation to how the
`mono-traversable` package works.

In addition to providing an abstraction over `IORef`, `STRef`, and `MutVar`,
this package provides four addition single-cell mutable references. `URef`,
`SRef`, and `BRef` all contain a 1-length mutable vector under the surface,
which is unboxed, storable, and boxed, respectively. The advantage of the first
two over boxed standard boxed references is that it can avoid a significant
amount of allocation overhead. See [the relevant Stack Overflow
discussion](http://stackoverflow.com/questions/27261813/why-is-my-little-stref-int-require-allocating-gigabytes)
and the benchmarks below.

While `BRef` doesn't give this same advantage (since the values are still
boxed), it was trivial to include it along with the other two, and does
actually demonstrate a performance advantage. Unlike `URef` and `SRef`, there
is no restriction on the type of value it can store.

The finally reference type is `PRef`. Unlike the other three mentioned, it
doesn't use vectors at all, but instead drops down directly to a mutable
bytearray to store values. This means it has slightly less overhead (no need to
store the size of the vector), but also restricts the types of things that can
be stored (only instances of `Prim`).

You should benchmark your program to determine the most efficient reference
type, but generally speaking `PRef` will be most performant, followed by `URef`
and `SRef`, and finally `BRef`.

## Collections

Collections allow you to push and pop values to the beginning and end of
themselves. Since different data structures allow different operations, each
operation goes into its own typeclass, appropriately named `MutablePushFront`,
`MutablePushBack`, `MutablePopFront`, and `MutablePopBack`. There is also a
parent typeclass `MutableCollection` which provides:

1. The `CollElement` associated type to indicate what kinds of values are in the collection.
2. The `newColl` function to create a new, empty collection.

The `mono-traversable` package provides a typeclass `IsSequence` which
abstracts over sequence-like things. In particular, it provides operations for
`cons`, `snoc`, `uncons`, and `unsnoc`. Using this abstraction, we can provide
an instance for all of the typeclasses listed above for any mutable reference
containing an instance of `IsSequence`, e.g. `IORef [Int]` or `BRef s (Seq
Double)`.

Note that the performance of some of these combinations is *terrible*. In
particular, `pushBack` or `popBack` on a list requires traversing the entire
list, and any push operations on a `Vector` requires copying the entire
contents of the vector. Caveat emptor! If you *must* use one of these
structures, it's highly recommended to use `Seq`, which gives the best overall
performance.

However, in addition to these instances, this package also provides two
additional data structures: double-ended queues and doubly-linked lists. The
former is based around mutable vectors, and therefore as unboxed (`UDeque`),
storable (`SDeque`), and boxed (`BDeque`) variants. Doubly-linked lists have no
such variety, and are simply `DList`s.

For general purpose queue-like structures, `UDeque` or `SDeque` is likely to
give you best performance. As usual, benchmark your own program to be certain,
and see the benchmark results below.

## Benchmark results

The following benchmarks were performed on January 7, 2015, against version 0.2.0.

### Ref benchmark

```
```

### Deque benchmark

```
```

## Test coverage

As of version 0.2.0, this package has 100% test coverage. If you look at the
report yourself, you'll see some uncovered code; it's just the automatically
derived `Show` instance needed for QuickCheck inside the test suite itself.

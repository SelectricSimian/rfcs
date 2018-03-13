- Design proposal name: Factor Out VCS
- Start Date: March 13, 2018

# Summary

This proposal suggests splitting Attaca into two libraries, `attaca-core` and
`attaca-vcs`, with `attaca-core` exposing a general-purpose API for interacting
with purely content-addressable stores, while `attaca-vcs` builds on top of this
to provide version control abstractions like commits and branches.

# Motivation

The most immediate benefit of this change is that it frees Attaca from needing
to store a "branch set," which is currently the only source of
location-addressed mutable data in Attaca's otherwise purely content-addressed,
immutable data model.  Currently, the branch set is responsible both for
tracking the mutable address of the most recent commit to each branch, and for
specifying the set of root addresses to use in garbage collection.  This
proposal separates these concerns: any semantically meaningful mutable data
associated with an Attaca store, such as a "branch set," is maintained by
higher-level external tools like `attaca-vcs`, while `attaca-core` retains a
low-level garbage collection API intended soley for managing resource usage.

Under this proposal, `attaca-core` becomes more flexible, general, and
mathematically pure, opening up the possibility of using it any time a
content-addressable store is required, not only for git-like version control.

# Design and Implementation

This proposal involves creating a new Rust crate, `attaca-vcs`, renaming the
current `attaca` crate to `attaca-core`, and factoring out abstractions related
to version control from `attaca-core` to `attaca-vcs`.

The most important changes which must be made to `attaca-core` involve the
following three areas:

- Functionality related to branches and branch names is factored out into
  `attaca-vcs`
- Functionality which treats commits, trees, and other version control data
  differently from other objects is factored out into `attaca-vcs`
- The "retained set" used for garbage collection becomes its own entity in
  `attaca-core`, separate from the old branch set

Concretely, we propose the following API changes as a first step towards an
implementation.

## Modifications to the `Backend` Trait

We remove the `load_branches` and `swap_branches` methods from the `Backend`
trait, as well as from the `Store` struct that wraps it.

In their place, we introduce an API for managing garbage collection.  This
involves establishing two new concepts: the "retained set," and the "root set."

The "retained set" is the abstract set of *all* objects which can be accessed in
the store.  It must be closed under pointer-chasing; if an object is in the
retained set, its dependencies must be as well.  The retained set is necessarily
mutable, and the removal of objects from the retained set constitutes their
deletion from the store via garbage collection.

The "retained set" is an abstract logical entity, and for the purposes of
managing garbage collection one interacts with it through an API in terms of
concrete "root sets."  A "root set" is a *non-unique* representation of a
retained set, given by a sequence of object addresses; the retained set it
represents is given by its transitive closure under pointer-chasing.

As much as possible, `attaca-core` and any software built on it should strive to
respect the equivalence relation on root sets induced by their projections to
retained sets.  That is, most Attaca software should not behave differently
based on which root set is chosen to represent a given retained set.

The following new methods are added to the `Backend` trait to provide control
over retained sets:

- `swap_retained`: Takes two root sets, `old` and `new`, and replaces the
  current retained set by the transitive closure of `new` if the current
  retained set is given by the transitive closure of `old`. Fails if there is
  any object in the current retained set not reachable from `old`, or
  vice-versa.  The new retained set must necessarily be a subset of the old
  retained set, because the garbage-collection API cannot create new objects
  from thin air!
- `report_retained`: Returns a representation of the current retained set as a
  root set.  The representation is not guaranteed to be unique and may be
  different than the last root set provided to `swap_retained` (although it must
  have the same transitive closure!)

Software using `attaca-core` should rarely use `report_retained`, and one should
really only need it if something goes slightly wrong.  One might use it, for
example, in trying to identify and manually delete an unwanted temporary object
whose address has been accidentally forgotten.  Under most circumstances, calls
to the Attaca garbage-collection API should be wrapped in a higher-level tool
which can anticipate the state of the retained set at all times and so does not
need to query it.

In addition to these new methods, we clarify the following semantics of existing
`Backend` API methods as they relate to the retained set:

- `finish` adds the object it creates to the retained set immediately upon
  creation.  This can in fact be regarded as the *definition* of `finish`.
- `resolve_id`, `resolve_digest`, and `load` should succeed *only if* the
  identified object is in the retained set.  Again, this in some sense *by
  definition*, because the retained set is exactly the set of objects which are
  queryable. We emphasize that the implementations of `swap_retained` and
  `report_retained` must be compatible with this definition; that is, they must
  consider the retained set to be *all* objects for which a query would succeed,
  not some conservative subset of queryable objects.

Concretely, this leads to the following modified Backend API:

```rust
pub trait Backend: Send + Sync + 'static {
    ///////////////
    // unchanged //
    ///////////////

    fn uuid(&self) -> [u8; 16];

    type Builder: Write + Extend<RawHandle> + 'static;
    type FutureFinish: Future<Item = RawHandle, Error = Error>;
    fn builder(&self) -> Self::Builder;
    fn finish(&self, Self::Builder) -> Self::FutureFinish;

    type Content: Read + Iterator<Item = RawHandle> + 'static;
    type FutureContent: Future<Item = Self::Content, Error = Error>;
    fn load(&self, id: RawHandle) -> Self::FutureContent;

    type Id: Id + ToOwned + ?Sized;
    type FutureId: Future<Item = <Self::Id as ToOwned>::Owned, Error = Error>;
    fn id(&self, id: RawHandle) -> Self::FutureId;

    type Digest: ErasedDigest;
    type FutureDigest: Future<Item = Self::Digest, Error = Error>;
    fn digest(&self, signature: DigestSignature, id: RawHandle) -> Self::FutureDigest;

    type FutureResolveId: Future<Item = Option<RawHandle>, Error = Error>;
    fn resolve_id(&self, bytes: &Self::Id) -> Self::FutureResolveId;

    type FutureResolveDigest: Future<Item = Option<RawHandle>, Error = Error>;
    fn resolve_digest(&self, signature: DigestSignature, bytes: &[u8])
        -> Self::FutureResolveDigest;

    ///////////
    // added //
    ///////////

    type FutureSwapRetained: Future<Item = (), Error = Error>;
    fn swap_retained(
        &self,
        old: Vec<RawHandle>,
        new: Vec<RawHandle>,
    ) -> Self::FutureSwapRetained;

    type FutureReportRetained: Future<Item = Vec<RawHandle>, Error = Error>;
    fn report_retained(&self) -> Self::FutureReportRetained;

    /////////////
    // removed //
    /////////////

    // type FutureLoadBranches: Future<Item = HashMap<String, RawHandle>, Error = Error>;
    // fn load_branches(&self) -> Self::FutureLoadBranches;

    // type FutureSwapBranches: Future<Item = (), Error = Error>;
    // fn swap_branches(
    //     &self,
    //     previous: HashMap<String, RawHandle>,
    //     new: HashMap<String, RawHandle>,
    // ) -> Self::FutureSwapBranches;
}
```

Similar changes are to be made to the `Store` struct and other APIs closely
related to the `Backend` trait.

## VCS-Specific Functionality to be Directly Split Into `attaca-vcs`

Variants related to trees and commits, as defined in the `object` module, should
be removed from `attaca-core` wherever they appear, leaving only `Small` and
`Large` blobs.  Trees and commits will be implemented in `attaca-vcs` as
higher-level abstractions represented internally as ordinary `attaca-core`
objects.

The `batch`, `hierarchy`, and `path` modules can be removed in their entirety
from `attaca-core` and implemented in `attaca-vcs` in terms of public
`attaca-core` APIs.

## New `attaca-vcs` APIs

Version-control-related functionality should be generic over content-addressable
store backends, and should not require the backend to specifically support
functionality related to version control.  We propose separating
location-addressable branch set storage from content-addressable object storage
with the following API:

```rust
pub trait BranchBackend: Send + Sync + 'static {
  type FutureLoadBranches: Future<Item = HashMap<String, DigestSignature>, Error = Error>;
  fn load_branches(&self) -> Self::FutureLoadBranches;

  type FutureSwapBranches: Future<Item = (), Error = Error>;
  fn swap_branches(
      &self,
      old: HashMap<String, DigestSignature>,
      new: HashMap<String, DigestSignature>,
    ) -> Self::FutureSwapBranches;
}
```

`BranchBackend` and `Backend` should not be implemented by the same types.  A `BranchBackend` is a local or remote mutable store of branches.

---

*This RFC is a work in progress.  More detail on the `attaca-vcs` APIS will be added soon.*

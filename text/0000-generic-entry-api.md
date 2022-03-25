- Feature Name: `generic-entry-api`
- Start Date: 2022-03-22
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

There are parallel features for “entries” in `std::collections::HashMap` and `std::collections::BTreeMap`, but as of now they are both incomplete and inconsistent. This RFC is an effort to make the two consistent, and to extend the pattern other collections by adding two new traits: `OccupiedEntry` and `VacantEntry`, and many more subtraits providing additional specific behaviors that are common among collections.

# Motivation
[motivation]: #motivation

When working with collections, there are times when you want to see if a value is present, and create a value to insert if it isn’t. Without the Entry API, this would require something like this:

```rust
match my_hash_map.get_mut(“test”) {
    Some(mut_ref) => { … }
    None => {
        my_hash_map.insert(“test”, MyValue::new())
    }
}
```

While this is fine, you end up traversing the hashmap twice for the same key, and more importantly you lower information from the type system to runtime; insert returns an option, but we know it is always `None`. This means that the `Drop` impl of `Option` will branch on a boolean value that is known to be `false`, when it is returned from insert`. The current entry API is resolves the main concern here, by replacing the code example with the following:

```rust
match my_hash_map.entry(“test”) {
    Entry::Occupied(occupied) => { … }
    Entry::Vacant(vacant) => {
        vacant.insert(MyValue::new())
    }
}
```

But there are problems with the current implementation that stem from imprecise abstractions for what an entry actually is. For a HashMap or a BTreeMap, an occupied entry is a type which allows you to get the key by reference, get the value by mutable reference, and remove the corresponding key/value pair from the collection (by consuming the entry). A vacant entry is a type which owns a key, and allows you to insert into the collection (again by consuming the entry).

Because these behaviors are very similar for many collections, both in the standard library and in outside crates, they are good candidates for abstracting via traits. The value of having all std::collection collections provide entry types which implement these traits is that complex operations can be defined generically over the underlying data structure, making switching data structures painless.

Limitations of the current api, which this approach solves are:
- The inability to move back and forth from occupied to vacant entry
- The ambiguity around raw and normal entries for HashMaps
- Similar apis for HashMap and BTreeMap cannot be used generically.

The concepts of Occupied and Vacant entries are useful in other contexts as well, such as skipping bounds checks that have already been verified on slices, or representing the init state of a `MaybeUninit<T>` in the type system after calling something like `assume_init_entry`


# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

If you are trying to squeeze every ounce of performance out of a collection in rust, or want to eliminate as many unwrap calls as possible, the entry api may be useful.

The “entry api” refers to a set of traits and structs which model partially completed mutable operations on collections. 

### Basic usage for `HashMap`
One scenario where the new entry API is functionally identical to the old one is when inserting into a collection. Say we know what key we would like to insert into a `HashMap`, but don’t want to create a value until we know there isn’t one already present for that key. In this scenario we can use the `get_entry` method from the `GetEntry` trait, which is implemented for `HashMap`. This will give us an `Entry`, which is either occupied or vacant. If occupied, we can mutate the present value; otherwise we can use the `VacantEntry` trait’s `insert` method, as implemented by `VacantHashMapEntry`, to fill the vacant slot, all without re-hashing the key.

```rust
let hash_map = HashMap::new();
let entry = hash_map.get_entry();
match entry {
    Entry::Occupied(occupied_entry) => { todo!() }
    Entry::Vacant(vacant_entry) => {
        // now we know the entry is vacant.
        let new_value = todo!();
        
        // there's no unwrapping because we know we can insert here.
        vacant_entry.insert(new_value);
    }
}
```

### Chaining entries
Many methods on Entry, OccupiedEntry, and VacantEntry consume `self` and return a new entry of some sort. In general, any method ending in `_entry` will return an `Entry`, any method ending in `_occupy` will return an `OccupiedEntry`, and any method ending in `_vacate` will return a `VacantEntry`. These will all have the same lifetime parameter as self, and can be used to preserve information in the type system about the presence or absence of values in the collection.

```rust
let hash_map = HashMap::new();
let entry = hash_map.get_entry();
let occupied_entry = match entry {
    Entry::Occupied(occupied_entry) => occupied_entry,
    Entry::Vacant(vacant_entry) => {
        let new_value = todo!();
        vacant_entry.occupy(new_value)
    }
}
// do some logic
occupied_entry.remove();
```

## Entry traits
Traits implemented by the entries of various collections.

### OccupiedEntry
A type implementing `OccupiedEntry` represents a value in a collection. It can be used to get or mutate a value in place.

### VacantEntry
A type implementing `VacantEntry` represents a vacant part of a collection which can be occupied when provided with a `Self::Value`. In this case “occupied” means converted into a `Self::Occupied`. Note that `Self::Value` might not be the same as `Self::Occupied::Value`, for example `RawVacantHashMapEntry` has value `(K, V)`, as it needs both `K` and `V` to insert into the collection, but once occupied the value is `V`

### KeyedOccupiedEntry
A type implementing `KeyedOccupiedEntry` represents an occupied entry which is identified by key. It can be used to immutably access the key, and mutably access the value.

### KeyedVacantEntry
A type implementing `KeyedVacantEntry` represents a vacant part of a keyed collection which can be occupied when provided with a `Self::Value`. This type should be thought of as owning a `Self::Key`, which will be matched with the value when provided.

### RemovableOccupiedEntry
A type implementing `RemovableOccupiedEntry` represents an occupied entry which can be removed. The precise meaning of "removed" depends on the collection. For `HashMap` and `BTreeMap`, removal means removing the item and returning a value and a vacant entry. For `LinkedList`, removal means removing the item and returning a valve the entry that was next after the removed item.

### EntryRemovableOccupiedEntry
A type implementing `EntryRemovableOccupiedEntry` represents a `RemovableOccupiedEntry` which can convert its `Removed` type into an `Entry`. This is important for implementations of methods on entry, but is not intended to be called by consumers of the API.

### InsertableOccupiedEntry
A type implementing `InsertableOccupiedEntry` represents an occupied entry which can be displaced without being removed from the collection. The canonical example here is `LinkedList`, where inserting on top of an occupied entry shifts that entry and the rest of the collection to the right. This is not intended to be implemented on key-value stores.

### KeyMutableOccupiedEntry
A type implementing `KeyMutableOccupiedEntry` represents a `KeyedOccupiedEntry` which can have the key mutated. The provided key must uphold the invariants of the collection (hashes to the same value for `HashMap`, sorts the same for `BTreeMap`

### IntoCollectionMut
A type implementing `IntoCollectionMut` can be converted back into a mutable reference to the original collection. This can be used as an additional chaining mechanism, though you need to re-traverse your data structure.

## Entry enums
Enums representing common combinations of entries.

### Entry
An entry is an enum which is either Occupied or Vacant. While it isn't required by `Entry`, most of the methods on `Entry` require `Vac: VacantEntry<'c, Occupied = Occ>`. `Entry` provides many convenience methods for inserting, removing, mutating, and chaining more entry methods.

### EntryWithSearchKey
If you acquire an entry in a keyed collection by searching with an owned key, that entry may have already been occupied. In this case you may not want to drop your key. `EntryWithSearchKey` can be matched to get the key out, or converted into a regular `Entry` for further use.

## Collection traits
Traits for acquiring entries from mutable references to collections. These are the only traits in this rfc which rely on `generic-associated-types`.

### GetEntryFromKey
A collection implementing `GetEntryFromKey` can take an owned key and return an `Entry`, `EntryWithSearchKey`, or `OccupiedEntry` depending on what method you use.

### GetEntryByKey
A collection implementing `GetEntryByKey` can take any borrow of a key and return an `Entry` or `VacantEntry` depending on the method you use. If an owned key is needed to produce an entry, the key will be cloned.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

As this is an rfc for the standard library, I think the reference-level explanation is best conveyed via documentation, [which is hosted here.](https://maboesanman.github.io/InPlace/in_place/) and [repo here.](https://github.com/maboesanman/InPlace)

As far as how this meshes with the existing entry api, I suggest leaving it as is, and making new methods. `GetEntryByKey::get_entry` was deliberately named to avoid conflicting with `HashMap::entry` or `BTreeMap::entry`. The only name collision with any collection in `std` is `GetEntryByKey::remove_entry`, which can be renamed, though I haven't thought of a good candidate for it. "Entry" could be replaced with "Slot" in all methods, traits, and enums without losing much intuitive meaning, then all names could be consistent.

I propose deprecating the following stable items:
- `std::collections::hash_map::HashMap::entry`
- `std::collections::hash_map::Entry`
- `std::collections::hash_map::OccupiedEntry`
- `std::collections::hash_map::VacantEntry`
- `std::collections::btree_map::BTreeMap::entry`
- `std::collections::btree_map::Entry`
- `std::collections::btree_map::OccupiedEntry`
- `std::collections::btree_map::VacantEntry`

And the following experimental items:
- `std::collections::hash_map::HashMap::try_insert`
- `std::collections::hash_map::OccupiedError`
- `std::collections::hash_map::RawEntryBuilder`
- `std::collections::hash_map::RawEntryBuilderMut`
- `std::collections::hash_map::RawOccupiedEntryMut`
- `std::collections::hash_map::RawVacantEntryMut`
- `std::collections::btree_map::BTreeMap::try_insert`
- `std::collections::btree_map::BTreeMap::first_entry`
- `std::collections::btree_map::BTreeMap::last_entry`
- `std::collections::btree_map::OccupiedError`

Then adding all the traits and enums in a new `std::collections::entry` module. The structs/enums that implement `OccupiedEntry` and `VacantEntry` will be named `HashMapOccupiedEntry` or `HashMapVacantEntry`, and placed in `std::collections::hash_map` (or the corresponding collection's module). If these names are too cumbersome we can revisit renaming "entry" to "slot" or "cursor" to avoid naming conflicts.

This should also block stabilization of linked list cursors, as there is a large amount of overlap in their functionality with this rfc, which could be powerful if implemented generically via the proposed traits.

# Drawbacks
[drawbacks]: #drawbacks

- This hierarchy of traits is complex, and while this is intended to be an advanced feature, it may still lead to frustration. The traits were designed to be maximally flexible while still being useful, but they may be a little too all encompassing. A more modest approach that focuses solely on keyed-value collections may be more digestible.
- The traits on collections will remain unstable while generic associated types are unstable.
- Having many traits can be frustrating when trying to keep them all in scope.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- This design unifies a few currently available/experimental features. Those features could continue independently, but this API gives them a common interface to strive towards.
- If this is not done, the entry api (particularly for `HashMap`, with their raw entries) may continue to get more distant from `BTreeMap`, despite the two of them being nearly identical, just with different trait bounds on their keys.
- Any collection can make use of some part of this api, even if they can't implement one of the collection traits directly. Even things not generally thought of as collections can get value out of the vacant/occupied model; `MaybeUninit<T>` for example.

# Prior art
[prior-art]: #prior-art

As far as I know this is the first attempt to make the entry api generic across collections. The prior art here is mostly merged into the standard library already. The various entry related apis are already usable in `std` today, but they could be more generic and more useful.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- How should the `remove_entry` name collision be resolved?
- How should the various entry structs be named?
- What helper methods should be provided on the various traits for convenience?
- What types (collections or otherwise) should implement these traits?
- Can the ergonomics of the many different entry traits be improved by something like an `OccupiedEntryExt` trait? Is there a better way to improve ergonomics?
- What should be `#[inline]`'d?
- are there any unforeseen implementation gotchas?

# Future possibilities
[future-possibilities]: #future-possibilities

- `first_occupied_entry`, `last_occupied_entry`, `next_occupied_entry` etc as traits. These could apply to both `LinkedList` and `BTreeMap`.
- Maybe there could be special entries that allocate memory for vacant entries, so that you have a guarantee that you are not going to panic when inserting into your vacant entry. This could be a very powerful tool for scenarios where allocation panics are not acceptable.
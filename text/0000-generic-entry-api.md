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

While this is fine, you end up traversing the hashmap twice for the same key, and more importantly you lower some information from the type system to runtime; insert returns an option, but we know it is always `None`. This means that the `Drop` impl of `Option` will branch on a boolean value that is known to be `false`, when it is returned from insert`. The current entry API is resolves the main concern here, by replacing the code example with the following:

```rust
match my_hash_map.entry(“test”) {
    Entry::Occupied(occupied) => { … }
    Entry::Vacant(vacant) => {
        vacant.insert(MyValue::new())
    }
}
```

But there are some problems with the current implementation that stem from imprecise abstractions for what an entry actually is. For a HashMap or a BTreeMap, an occupied entry is a type which allows you to get the key by reference, get the value by mutable reference, and remove the corresponding key/value pair from the collection (by consuming the entry). A vacant entry is a type which owns a key, and allows you to insert into the collection (again by consuming the entry).

Because these behaviors are very similar for many collections, both in the standard library and in outside crates, they are good candidates for abstracting via traits. The value of having all std::collection collections provide entry types which implement these traits is that complex operations can be defined generically over the underlying data structure, making switching data structures relatively painless.

Some limitations of the current api, which this approach solves are:
- the inability to move back and forth from occupied to vacant entry easily
- the very separate raw_entry api for HashMaps
- Very similar apis for HashMap and BTreeMap are not able to be worked with generically.

The concepts of Occupied and Vacant entries are useful in other contexts as well, such as skipping bounds checks that have already been verified on slices, or representing the init state of a MaybeUninit<T> in the type system after calling something like `assume_init_entry`


# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

Explain the proposal as if it was already included in the language and you were teaching it to another Rust programmer. That generally means:

- Introducing new named concepts.
- Explaining the feature largely in terms of examples.
- Explaining how Rust programmers should *think* about the feature, and how it should impact the way they use Rust. It should explain the impact as concretely as possible.
- If applicable, provide sample error messages, deprecation warnings, or migration guidance.
- If applicable, describe the differences between teaching this to existing Rust programmers and new Rust programmers.

If you are trying to squeeze every ounce of performance out of a collection in rust, or want to eliminate as many unwrap calls as possible, the entry api may be useful.

The “entry api” refers to a set of traits and structs which model partially completed mutable operations on collections. 

### Example
We know what key we would like to insert into a `HashMap`, but don’t want to produce a value until we know there isn’t already one present for that key, we can use the `get_entry` method from the `GetEntry` trait, which is implemented for `HashMap`. This will give us an `Entry`, which is either occupied or vacant. If occupied, we can mutate the present value; otherwise we can use the `VacantEntry` trait’s `insert` method, as implemented by `VacantHashMapEntry`, to fill the vacant slot, all without re-hashing the key.

### Chaining Entries
Many methods on Entry, OccupiedEntry, and VacantEntry consume `self` and return a new entry of some sort. In general, any method ending in `_entry` will return an `Entry`, any method ending in `occupy` will return an `OccupiedEntry`, and any method ending in `vacate` will return a `VacantEntry`. These will all have the same lifetime parameter as self, and can be used to preserve information in the type system about the presence or absence of values in the collection.

## Trait overview

### OccupiedEntry
A type implementing `OccupiedEntry` represents a value in a collection. It can only be used to get or mutate a value in place.

### VacantEntry
A type implementing `VacantEntry` represents a currently vacant part of a collection which can be occupied when provided with a `Self::Value`. In this case “occupied” means converted into a `Self::Occupied`. Note that `Self::Value` might not be the same as `Self::Occupied::Value`, for example `RawVacantHashMapEntry` has value `(K, V)`, as it needs both `K` and `V` to insert into the collection, but once occupied the value is simply `V`

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

This is the technical portion of the RFC. Explain the design in sufficient detail that:

- Its interaction with other features is clear.
- It is reasonably clear how the feature would be implemented.
- Corner cases are dissected by example.

The section should return to the examples given in the previous section, and explain more fully how the detailed proposal makes those examples work.

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

# Prior art
[prior-art]: #prior-art

Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- For language, library, cargo, tools, and compiler proposals: Does this feature exist in other programming languages and what experience have their community had?
- For community proposals: Is this done by some other community and what were their experiences with it?
- For other teams: What lessons can we learn from what other communities have done here?
- Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.

This section is intended to encourage you as an author to think about the lessons from other languages, provide readers of your RFC with a fuller picture.
If there is no prior art, that is fine - your ideas are interesting to us whether they are brand new or if it is an adaptation from other languages.

Note that while precedent set by other languages is some motivation, it does not on its own motivate an RFC.
Please also take into consideration that rust sometimes intentionally diverges from common language features.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- What parts of the design do you expect to resolve through the RFC process before this gets merged?
- What parts of the design do you expect to resolve through the implementation of this feature before stabilization?
- What related issues do you consider out of scope for this RFC that could be addressed in the future independently of the solution that comes out of this RFC?

# Future possibilities
[future-possibilities]: #future-possibilities

Think about what the natural extension and evolution of your proposal would
be and how it would affect the language and project as a whole in a holistic
way. Try to use this section as a tool to more fully consider all possible
interactions with the project and language in your proposal.
Also consider how this all fits into the roadmap for the project
and of the relevant sub-team.

This is also a good place to "dump ideas", if they are out of scope for the
RFC you are writing but otherwise related.

If you have tried and cannot think of any future possibilities,
you may simply state that you cannot think of anything.

Note that having something written down in the future-possibilities section
is not a reason to accept the current or a future RFC; such notes should be
in the section on motivation or rationale in this or subsequent RFCs.
The section merely provides additional information.
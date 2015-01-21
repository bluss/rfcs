- Start Date: 2015-01-21
- RFC PR:
- Rust Issue:

# Summary

Add the syntax `..` for `std::ops::FullRange`.

# Motivation

Range expressions `a..b`, `a..` and `..b` all have dedicated syntax and
produce first-class values. This means that they will be usable and
useful in custom APIs, so for consistency, the fourth slicing range,
`FullRange`, could have its own syntax `..`

# Detailed design

`..` will produce a `std::ops::FullRange` value when it is used in an
expression. This means that slicing the whole range of a slicable
container is written `&foo[..]`.

We should remove the old `&foo[]` syntax for consistency. Because of
this breaking change, it would be best to change this before Rust 1.0.

As previously stated, when we have range expressions in the language,
they become convenient to use when stating ranges in an API.

@Gankro fielded ideas where
methods like for example `.remove(index) -> element` on a collection
could be generalized by accepting either indices or ranges. Today's `.drain()`
could be expressed as `.remove(..)`.

This is just one example of API design that a complete set of all four range
expressions allows.

Because of deref coercions, the very common String and Vec to slices
conversions don't need to use slicing syntax at all, so the change in
verbosity from `[]` to `[..]` is not a concern.

# Drawbacks

Removing the slicing syntax `&foo[]` is a breaking change.

# Alternatives

* We could add this syntax later, but we would end up with duplicate
  slicing functionality using `&foo[]` and `&foo[..]`.

* `0..` could replace `..` in many use cases (but not for ranges in
  ordered maps).

# Unresolved questions

Any parsing questions should already be mostly solved because of `a..` and
`..b` cases.

- Feature Name: `fold_while`
- Start Date: 2016-11-24
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

**DRAFT**

# Summary
[summary]: #summary

Add the iterator method `fold_while` that generalizes the existing methods
`all`, `any`, `find`, `position` and `fold` methods. Iterators can provide one
specific traversal implementation for all of them. Iterators can additionally
implement `rfold_while` to have the same search and fold methods improved
through their reversed iterator as well.

# Motivation
[motivation]: #motivation

*External* iteration has the consumer control when the next iterator element is
consumed, and this is the usual Rust model based around the iterator's
`.next()` method.
*Internal* iteration inverts control and the iterator “pushes” the elements to
the consumer. This is already used by the searching or folding iterator methods
(`all` and the others).

`fold_while` and `rfold_while` allow iterators to special case just one (or two)
iterator methods, and having all the listed iterator methods gain from that
implementation by default.

The existance of both forward and reverse methods mean that reversed iterators
(the `Rev` adaptor) can use the improved implementations as well, so that
`iter.rev().find()` can be as efficient as `iter.find()` is.

## Why Internal Iteration

When the iterator is in control, it can unravel its nested structure directly.
An example of this is [PR #37315][prfold]. In internal iteration, traversing
the `VecDeque` means splitting it into two slices, then a for loop over each
slice; a reduction to simpler and more efficient pieces.

[prfold]: https://github.com/rust-lang/rust/pull/37315

Other data structures with segmented memory layout [can benefit][rulinalg] the
same way as `VecDeque`.

[rulinalg]: https://github.com/AtheMathmo/rulinalg/issues/86

Iterators can be composed together; `a.chain(b)` creates a new iterator that
is the concatenation of `a` and `b`. Chain is already implementing
`Iterator::find` to first run find on the first iterator, then the second.
This is explicitly unraveling the nested structure, instead of going through
Chain's own `next` method; with `fold_while` it only needs to define this
once instead of for each of the searching and folding methods.

## Why Composable

The fold while methods should be self-composable so that it is easy for
composite iterators like `chain` and `flat_map` to use them.

# Detailed design
[design]: #detailed-design

+ `Iterator` gains a new method `fold_while` with a provided implementation.
  `DoubleEndedIterator` gains a new method `rfold_while` with a provided
  implementation.  The two implementations are listed below.

```rust
pub trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item> { unimplemented!() }
    
    /// Starting with initial accumulator `init`, combine the accumulator
    /// with each iterator element using the `g` closure until it returns
    /// `FoldWhile::Done` or the iterator's end is reached.
    /// The last `FoldWhile` value is returned.
    fn fold_while<Acc, G>(&mut self, init: Acc, mut g: G) -> FoldWhile<Acc>
        where Self: Sized,
              G: FnMut(Acc, Self::Item) -> FoldWhile<Acc>
    {
        let mut accum = init;
        while let Some(element) = self.next() {
            match g(accum, element) {
                FoldWhile::Continue(res) => accum = res,
                done @ FoldWhile::Done(_) => return done,
            }
        }
        FoldWhile::Continue(accum)
    }
}

pub trait DoubleEndedIterator : Iterator {
    fn next_back(&mut self) -> Option<Self::Item> { unimplemented!() }
    
    fn rfold_while<Acc, G>(&mut self, init: Acc, mut g: G) -> FoldWhile<Acc>
        where Self: Sized,
              G: FnMut(Acc, Self::Item) -> FoldWhile<Acc>
    {
        let mut accum = init;
        while let Some(element) = self.next_back() {
            match g(accum, element) {
                FoldWhile::Continue(res) => accum = res,
                done @ FoldWhile::Done(_) => return done,
            }
        }
        FoldWhile::Continue(accum)
    }
}
```

The control enum `FoldWhile` holds the value field inside both of its variants.
This design means that the user can't accidentally forget to handle control flow.

```rust
/// An enum used for controlling the execution of `.fold_while()`.
pub enum FoldWhile<T> {
    /// Continue folding with this value
    Continue(T),
    /// Fold is complete and will return this value
    Done(T),
}

impl<T> FoldWhile<T> {
    /// Return the inner value.
    pub fn into_inner(self) -> T {
        match self {
            FoldWhile::Continue(t) => t,
            FoldWhile::Done(t) => t,
        }
    }
}

```


+ Iterator methods `all`, `any`, `find`, `position`, `fold` are all have their
default implementation changed to use `fold_while`.

+ Iterator method `rposition` changes its default implementation to use `rfold_while`.

+ The `Rev` adaptor changes its iterator methods to make use of `fold_while` and
`rfold_while` on the base iterator when possible. This enables implementation
specific improvements to be reachable through the reversed iterator.

+ The `Iterator for &mut I` blanket implementation will gain a specialization
for the `I: Sized` case (when that is possible) and it will forward
the fold while methods, and will implement `fold` by calling `fold_while` on `I`.
This makes an implementation specific `fold_while` reachable from `&mut I` even
when a specific `fold` was not (because `fold` uses a `self` receiver).

+ Iterator documentation will recommend providing a implementation specific
`fold_while` and `rfold_while` in favor of any of the methods that use it
(while of course not insisting on such implementations, most iterators don't
need to implement them).

+ The iterator adaptors will forward `fold_while` and `rfold_while` if
applicable.

## Example: Chain

This is the implementation of `.fold_while` for `Chain`, which shows the need
for returning the `FoldWhile` enum for composability.

```rust
// helper macro for fold_while's control flow (internal use only)
macro_rules! fold_while {
    ($e:expr) => {
        match $e {
            FoldWhile::Continue(t) => t,
            done @ FoldWhile::Done(_) => return done,
        }
    }
}

fn fold_while<Acc, G>(&mut self, init: Acc, mut g: G) -> FoldWhile<Acc>
    where G: FnMut(Acc, Self::Item) -> FoldWhile<Acc>
{
    let mut accum = init;
    match self.state {
        ChainState::Both | ChainState::Front => {
            accum = fold_while!(self.a.fold_while(accum, &mut g));
        }
        _ => { }
    }
    match self.state {
        ChainState::Both | ChainState::Back => {
            self.state = ChainState::Back;
            accum = fold_while!(self.b.fold_while(accum, &mut g));
        }
        _ => { }
    }
    FoldWhile::Continue(accum)
}
```

## Example: Slice Iterator

[PR 37972][prslice] tunes the implementations of `Iterator::find` and similar
methods for the slice iterators in particular. With this RFC, only the methods
`fold_while` and `rfold_while` need to be implemented instead, and the benefit
to `iter.find()` would also apply to `iter.rev().find()`.

[prslice]: https://github.com/rust-lang/rust/pull/37972

# Drawbacks
[drawbacks]: #drawbacks

- `fold_while` and `rfold_while` require threading the state correctly through
  the iterator's parts.
- Adding new methods to `Iterator` and `DoubleEndedIterator` will clash with
  other iterator extension traits that users may have.
- `fold` is much simpler to implement because it consumes the iterator (`self`
  receiver) and doesn't need to save any partial state.
  Implementations can choose to just define `fold` instead if that is
  enough for them, however.

# Alternatives
[alternatives]: #alternatives

## Using an existing method like `.all()`

- None of the existing iterator methods can be made into the “canonical” one
  to override that all the other methods

  1. Changing the default implementation of for example `Iterator::any` to call
     `Iterator::all` is a breaking change for implementations that already
     implement `all` by calling the default `any`. (There's no particular reason
     to do so, but anyway.)
  2. `fold` is general but not short circuiting. It covers many use cases
     on its own, though (sum, min, max, and more).
  3. Strictly, `Iterator::all` already implements general internal iteration,
     but ownership rules make it very awkward to use for searching or folding
     with the result value in as a captured variable, it does not have the
     rigid control flow of fold while, and it does not give us the broader
     benefits of access to improved versions through reversed iterators.

## Use `Result<T, E>` instead of `FoldWhile<T>`

Instead of introducing a new enum, `Result` can be used instead.

- Drawback: Overloading the semantics of `Result` (it would use
  `Ok` for “continue” and `Err` for “done”).
- Drawback: Breaking for a successfully found element is signalled using `Err`.
- Advantage: It can use existing `try!()` or `?` for control flow.
- Advantage: Integrates well with fallible operations

## Add three methods intead of two:

  + `.rfold()`, the reverse version of `fold`. Gives `fold` benefits to `Rev<I>`.
  + `search_while<Res, G>(&mut self, default: Res, g: G) -> SearchWhile<Res>`
    and corresponding `rsearch_while` method.
    Control enum: `enum SearchWhile<T> { Done(T), Continue }`. Control flow is
    simpler.
  + Each of the methods is simpler to implement, but it increases the burden
    from one (two) methods to two (four), going from `fold_while` (`rfold_while`)
    to `fold` and `search_while` (`rfold`, `rsearch_while`).

# Unresolved questions
[unresolved]: #unresolved-questions

- Do the benefits to reversed iterators (`Rev<I>`) materialize? Has not been
  implemented yet.
- Should the helper macro `fold_while!()` be public? Once macro namespacing
  exists, I think it should be exported.

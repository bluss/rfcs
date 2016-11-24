- Feature Name: `fold_while`
- Start Date: 2016-11-24
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary
[summary]: #summary

Add iterator methods `fold_while` and `rfold_while` and generalize the existing
methods `all`, `any`, `find`, `position`, `fold` and `rposition` methods.


# Motivation
[motivation]: #motivation

*External* iteration puts the consumer in control, and this is the usual Rust
model based around the iterator's `.next()` method. *Internal* iteration
inverts control and the iterator “pushes” the elements to the consumer. This
is used by the already listed searching or folding iterator methods.

`fold_while` and `rfold_while` allow iterators to special case just one (or two)
iterator methods, and having all the listed iterator methods gain from that
implementation by default.

The existance of both forward and reverse methods mean that reversed iterators
(the `Rev` adaptor) can use the improved implementations as well.

## Why Internal Iteration

When the iterator is in control, it can unravel its nested structure directly.
An example of this is [PR #37315][prfold]. In internal iteration, traversing
the VecDeque means splitting it into two slices, then a for loop over each
slice.

[prfold]: https://github.com/rust-lang/rust/pull/37315

Iterators can be composed together. `a.chain(b)` creates a new iterator that
is the concatenation of `a` and `b`. Chain is already implementing
`Iterator::find` to first run find on the first iterator, then the second.
This is explicitly unraveling the nested structure, instead of going through
Chain's own `next` method.

## Why Composable

The fold while methods should be self-composable so that it is easy for
composite iterators like `chain` and `flat_map` to use them.

The control enum `FoldWhile` holds the value field inside both of its variants.
This design means that the user can't accidentally forget to handle control flow.

# Detailed design
[design]: #detailed-design

+ `Iterator` gains a new method `fold_while` with a provided implementation.
  `DoubleEndedIterator` gains a new method `rfold_while` with a provided
  implementation.  The two implementations are listed below.

```rust
pub trait Iterator {
    /// Starting with initial accumulator `init`, combine the accumulator
    /// with each iterator element using the `g` closure, until it returns
    /// `FoldWhile::Done` or the iterator's end is reached. The last `FoldWhile`
    /// value is returned.
    fn fold_while<Acc, G>(&mut self, init: Acc, g: G) -> FoldWhile<Acc>
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

pub trait DoubleEndedIterator {
    fn rfold_while<Acc, G>(&mut self, init: Acc, g: G) -> FoldWhile<Acc>
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

// helper macro for fold_while's control flow (internal use only)
macro_rules! fold_while {
    ($e:expr) => {
        match $e {
            FoldWhile::Continue(t) => t,
            done @ FoldWhile::Done(_) => return done,
        }
    }
}
```

This is the implementation of `.fold_while` for `Chain`, which explains
the use of the `fold_while!` macro.

```rust
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
            accum = fold_while!(self.b.fold_while(accum, &mut g));
        }
        _ => { }
    }
    FoldWhile::Continue(accum)
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

# Drawbacks
[drawbacks]: #drawbacks

- `fold_while` and `rfold_while` require threading the state correctly through
  the iterator's parts.
- Adding new methods to `Iterator` and `DoubleEndedIterator` will clash with
  other iterator extension traits that users may have.

# Alternatives
[alternatives]: #alternatives

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

# Unresolved questions
[unresolved]: #unresolved-questions


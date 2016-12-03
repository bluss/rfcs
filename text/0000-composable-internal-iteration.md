- Feature Name: `fold_ok`
- Start Date: 2016-11-24
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

**DRAFT**

# Summary
[summary]: #summary

Add the iterator method `fold_ok` that exends `fold` to be fallible and
short-circuiting. `fold_ok` generalizes the existing methods
`all`, `any`, `find`, `position` and `fold`, and will be their new
common base; iterators can provide one specific traversal implementation for
all of them. Iterators can additionally implement `rfold_ok` to have the same
search and fold methods improved through their reversed iterator as well.

# Motivation
[motivation]: #motivation

*External* iteration has the consumer control when the next iterator element is
consumed, and this is the usual Rust model based around the iterator's
`.next()` method.

*Internal* iteration inverts control and the iterator “pushes” the elements to
the consumer. This is already used by the searching or folding iterator methods
(`all` and the others).

`fold_ok` and `rfold_ok` allow iterators to special case just one (or two)
iterator methods, and having very many iterator methods gain from that by default.

## Why `fold_ok`

`fold_ok` generalizes `fold` to add short-circuiting on error. This integrates
with `Result` and the `?` operator.

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
Chain's own `next` method; with `fold_ok` it only needs to define this
once instead of for each of the searching and folding methods.

## Why Composable

The fold while methods should be self-composable so that it is easy for
composite iterators like `chain` and `flat_map` to use them.

## Why Reversible

The existence of both forward and reverse methods mean that reversed iterators
(the `Rev` adaptor) can use the improved implementations as well, so that
`iter.rev().find()` can be as efficient as `iter.find()` is.

# Detailed design
[design]: #detailed-design

+ `Iterator` gains a new method `fold_ok` with a provided implementation.
  `DoubleEndedIterator` gains a new method `rfold_ok` with a provided
  implementation.  The two implementations are listed below.

```rust
pub trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
    
    /// Starting with initial accumulator `init`, combine the accumulator
    /// with each iterator element using the closure `g` until it returns
    /// `Err` or the iterator’s end is reached.
    /// The last `Result` value is returned.
    fn fold_ok<Acc, E, G>(&mut self, init: Acc, mut g: G) -> Result<Acc, E>
        where Self: Sized,
              G: FnMut(Acc, Self::Item) -> Result<Acc, E>
    {
        let mut accum = init;
        while let Some(elt) = self.next() {
            accum = g(accum, elt)?;
        }
        Ok(accum)
    }
}

pub trait DoubleEndedIterator : Iterator {
    fn next_back(&mut self) -> Option<Self::Item>;
    
    fn rfold_ok<Acc, E, G>(&mut self, init: Acc, mut g: G) -> Result<Acc, E>
        where Self: Sized,
              G: FnMut(Acc, Self::Item) -> Result<Acc, E>
    {
        let mut accum = init;
        while let Some(elt) = self.next_back() {
            accum = g(accum, elt)?;
        }
        Ok(accum)
    }
}
```


+ `fold_ok` and `rfold_ok` use a `&mut self` receiver because they don’t
necessarily consume the iterator fully.

+ Iterator methods `all`, `any`, `find`, `position`, `fold` are all have their
default implementation changed to use `fold_ok`.

+ Iterator method `rposition` changes its default implementation to use `rfold_ok`.

+ sum and product already use `fold`; the max and min methods should change
  their default implementations to use `fold`.

+ The `Rev` adaptor changes its iterator methods to make use of `fold_ok` and
`rfold_ok` on the base iterator when possible. This enables implementation
specific improvements to be reachable through the reversed iterator.

+ The `Iterator for &mut I` blanket implementation will gain a specialization
for the `I: Sized` case (when that is possible) and it will forward
the fold while methods, and will implement `fold` by calling `fold_ok` on `I`.
This makes an implementation specific `fold_ok` reachable from `&mut I` even
when a specific `fold` was not (because `fold` uses a `self` receiver).

+ Iterator documentation will recommend providing a implementation specific
`fold_ok` and `rfold_ok` in favor of any of the methods that use it
(while of course not insisting on implementations, most iterators don't need to
implement them).

+ The iterator adaptors will forward `fold_ok` and `rfold_ok` if applicable.

## Example: Chain

This is the implementation of `.fold_ok` for `Chain`, which shows composability:
it is possible to apply `fold_ok` recursively.

```rust
fn fold_ok<Acc, E, G>(&mut self, init: Acc, mut g: G) -> Result<Acc, E>
    where G: FnMut(Acc, Self::Item) -> Result<Acc, E>
{
    let mut accum = init;
    match self.state {
        ChainState::Both | ChainState::Front => {
            accum = self.a.fold_ok(accum, &mut g)?;
        }
        _ => { }
    }
    match self.state {
        ChainState::Both | ChainState::Back => {
            self.state = ChainState::Back;
            accum = self.b.fold_ok(accum, &mut g)?;
        }
        _ => { }
    }
    Ok(accum)
}
```

## Example: all


How `Iterator::all` can be implemented in terms of `fold_ok`.


```rust
fn all<F>(&mut self, mut predicate: F) -> bool
    where F: FnMut(Self::Item) -> bool,
{
    self.fold_ok(true, move |_, elt| {
        if predicate(elt) {
            Ok(true)
        } else {
            Err(false)
        }
    }).unwrap_or_else(|e| e)
}
```

## Example: Slice Iterator

[PR 37972][prslice] tunes the implementations of `Iterator::find` and similar
methods for the slice iterators in particular. With this RFC, only the methods
`fold_ok` and `rfold_ok` need to be implemented instead, and the benefit
to `iter.find()` would also apply to `iter.rev().find()`.

[prslice]: https://github.com/rust-lang/rust/pull/37972

## Example: Take

How `Take` can implement `fold_ok`.

```rust
pub struct Take<I> {
    n: usize,
    iter: I,
}

// Stopping reason in fold_ok
enum TakeStop<L, R> {
    Their(L),
    Our(R),
}

impl<I> Iterator for Take<I>
    where I: Iterator
{
    type Item = I::Item;
    fn next(&mut self) -> Option<Self::Item> { unimplemented!() }

    fn fold_ok<Acc, E, G>(&mut self, init: Acc, mut g: G) -> Result<Acc, E>
        where G: FnMut(Acc, Self::Item) -> Result<Acc, E>
    {
        if self.n == 0 {
            return Ok(init);
        }
        let n = &mut self.n;
        let result = self.iter.fold_ok(init, move |acc, elt| {
            *n -= 1;
            match g(acc, elt) {
                Err(e) => Err(TakeStop::Their(e)),
                Ok(x) => {
                    if *n == 0 {
                        Err(TakeStop::Our(x))
                    } else {
                        Ok(x)
                    }
                }
            }
        });
        match result {
            Err(TakeStop::Their(e)) => Err(e),
            Err(TakeStop::Our(x)) => Ok(x),
            Ok(x) => Ok(x)
        }
    }
}
```

# Drawbacks
[drawbacks]: #drawbacks

- `fold_ok` and `rfold_ok` require threading the state correctly through the
  iterator's parts.
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
     rigid control flow of fold while, and it does not alone give us the
     broader benefits of access to improved versions through reversed
     iterators.


## Add three methods instead of two:

Add `.rfold()`, the reverse version of `fold`. Gives `fold` benefits to `Rev<I>`.
And add `find_map<X>(&mut self, |Self::Item| -> Option<X>) -> Option<X>`
and corresponding `rfind_map` method.  Control flow is simpler.


+ Advantage: Each of the methods is simpler to implement
+ Drawback: it increases the implementation burden from one (two) methods to
  two (four), going from `fold_ok` (`rfold_ok`) to `fold` and
  `find_map` (`rfold`, `rfind_map`).
+ Advantage: You may want manual unrolling for searches (`find`), but a plain
  loop for unconditional `fold`
+ Drawback: Losing `fold_ok` loses a useful iterator method in itself.


## Use `FoldWhile<T>` instead of `Result`

Use a more specific enum instead of `Result` and reformulate the method(s) to
be `fold_while` and `rfold_while`.

- Advantage: Avoid overloading the semantics of `Result` (it would use
  `Ok` for “continue” and `Err` for “done”).
- Advantage: Better semantic mapping to when the early exit case is the success
  case.
- Disadvantage: Need custom macro instead of `?` operator.

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


# Unresolved questions
[unresolved]: #unresolved-questions

- Are there any existing `fold_ok`, `rfold_ok` methods that extend `Iterator`
  in the common Rust ecosystem?
- Should the more specific method `rfold` exist (corresponding to `fold`) exist
  as well, regardless?

---
layout: post
title: "An overview of Aeneas"
date: 2026-03-18
tags: verification rust
published: true
---

# An Overview of the Aeneas Rust Verification Toolchain

I finally took some time to write a blog-post that presents Aeneas.
This post is essentially the original ICFP 22 paper but with more examples
and up-to-date explanations (the formalism has evolved quite a bit).
The explanations I present here do not cover the latest features supported
by the implementation such as nested (mutable) borrows, iterators, or dyn traits,
but is nonetheless a quite comprehensive overview. Enjoy!

## Introduction

[Aeneas](https://github.com/AeneasVerif/aeneas) is a verification toolchain
for Rust programs. It leverages Rust's ownership discipline to translate Rust
code into pure functional programs, eliminating memory reasoning entirely. The
proof engineer can then verify the resulting code in a theorem prover such as
Lean, focusing on *functional* properties rather than on memory or aliasing.

The key observation is that a large class of safe Rust programs are functional in
essence. References and borrows are an implementation device for performance and memory
layout; coupled with Rust's linear ownership discipline, these programs admit a pure
functional equivalent. Aeneas makes this equivalence concrete.
The key technical device that allows supporting the most subtle patterns,
functions returning mutable borrows, is something I called *backward functions*.

Aeneas is also more than a compiler from Rust to whatever theorem prover you want:
making the translation work required introducing novel semantics for safe Rust, while
the translation itself is performed by means of a symbolic execution (or abstract
interpretation, depending on your way of seeing things) for this semantics.
Importantly, this symbolic execution implements a borrow-checker, a fact that
we have formalized. This means that whenever you run Aeneas to translate a program
you actually also borrow-check it. Interestingly, this borrow-checker can validate
some programs that are not supported by the current borrow-checker or Polonius
(see the examples in the blog post).

Aeneas is being used in practice! Microsoft is
using it to verify implementations of the
[SymCrypt](https://github.com/microsoft/SymCrypt) cryptographic library
([blog post](https://www.microsoft.com/en-us/research/blog/rewriting-symcrypt-in-rust-to-modernize-microsofts-cryptographic-library/)).

## Aeneas and its Functional Translation, by Example

Before jumping into the various facets of our formalism, we keep an eye on the
prize, and immediately showcase how Aeneas translates Rust programs to pure
equivalents. In this section, and for the remainder of this post, we use
Lean syntax for our functional translation; it greatly resembles OCaml and
other ML languages, and as such should be familiar to the reader. A brief note
about terminology: we refer to *regions*, emphasizing that a region encompasses a set of
borrows and loans at a given program point. The Rust compiler and documentation, however,
refer to *lifetimes*, which conveys the idea of a syntactic bracket, and a specific
implementation technique to enforce soundness. In this post, whenever we talk about Rust
specifically, we use "lifetime"; whenever we emphasize our semantic view of ownership, we
use "region".

### Mutable Borrows, Functionally

To warm up, we consider an example that, albeit small, showcases
many of Rust's
features, including its ownership mechanism. In the Rust program below,
`incr` increments a reference, and `test_incr` acts as a
representative caller of the function.

```rust
fn incr(x: &mut i32) {
  *x = *x + 1;
}

fn test_incr() {
  let mut y = 0i32;
  incr(&mut y);
  assert!(y == 1);
}
```

The `incr` function operates by reference; that is, it receives the address of
a 32-bit signed integer `x`, as indicated by the `&` (reference) type. In
addition, `incr` is allowed to modify the contents at address `x`, because
the reference is of the `mut` (mutable) kind, which permits memory
modification. Finally, the Rust type system enforces that mutable references
have a unique owner: the definition of `incr` type-checks, meaning that
the function not only *guarantees* it does not duplicate ownership of
`x`, but also can *rely* on the fact that no one else owns `x`.

In `test_incr`, we allocate a mutable value (`let mut`) on the stack; upon
calling `incr`, we take a mutable reference (`&mut`) to `y`.
Statically, `y` becomes unavailable as long as `&mut y` is active. In Rust
parlance, `y` is *mutably borrowed* and its ownership has been
transferred to the mutable reference.
To type-check the call, the type-checker performs a lifetime analysis: the
`incr` function has type `(&'a mut i32) -> ()`, and the `&mut y`
borrow has type `&'b mut i32`; both `'a` and `'b` are lifetime variables.

For now, suffices to say that the type-checker ascertains that the lifetime
`'b` of the mutable borrow satisfies the lifetime annotation `'a` in the
type of the callee, and deems the call valid.
Immediately after the call, Rust *terminates* the
region `'b`, in effect *relinquishing* ownership of the mutable reference
`&'b mut y` so as to make `y` usable again inside `test_incr`.
This in turn allows the
`assert` to type-check, and thus the whole program.
Undoubtedly, this is a very minimalistic program; yet, there are two properties
of interest that we may want to establish already. The obvious one: the
assertion always succeeds. More subtly, doing so requires us to prove an
additional property, namely that the addition at line 2 does not overflow.

The key insight of Aeneas is that even though the program manipulates
references and borrows, none of this is informative when it comes to reasoning
about the program. More precisely: `x` and `y` are uniquely owned, meaning
that there are no stray aliases through which `x` or `y` may be modified;
in other words, to understand what happens to `y`, it suffices to track what
happens to `&mut y`, and therefore to `x`.
Feeding this program to Aeneas generates the following translation, where
`+` is syntactic sugar for an Aeneas primitive that captures the semantics of
error-on-overflow in Rust.

```lean
def incr (x : I32) : Result I32 :=
  x + 1#i32 -- evaluates to `fail` in case of overflow

def test_incr : Result Unit := do
  let y <- incr 0#i32 -- monadic bind
  massert (y = 1#i32) -- monadic assert

#assert (test_incr = ok ())
```

This program is semantically equivalent to the original Rust code, but does not
rely on the memory: we have leveraged the precise ownership discipline of Rust
to generate a functional, pure version of the program. In hindsight, the usage
of references in Rust was merely an implementation detail, which is why
`incr` becomes a simple (possibly-overflowing) addition. Should the
call to `incr` succeed, its result is bound to `y`;
the `assert` simply becomes a boolean test that may generate a failure
in the error monad.

For the purposes of unit-testing, we can use Lean's normalize to evaluate that
`test_incr` always succeeds, leveraging the fact that Aeneas produces an
executable translation, not a logical encoding.
In the remainder of this section, we use `<-`, Lean's bind
operator in the error monad.

### Returning a Mutable Borrow, and Backward Functions

Rust programs, however, rarely admit such immediate translations. To see why,
consider the following example, where the `choose` function returns a borrow,
as indicated by its return type `&'a mut`.

```rust
fn choose<'a, T>(b: bool, x: &'a mut T, y: &'a mut T) -> &'a mut T {
  if b { return x; } else { return y; }
}

fn test_choose() {
  let mut x = 0i32;
  let mut y = 0i32;
  let z = choose(true, &mut x, &mut y);
  *z = *z + 1;
  assert!(*z == 1);
  assert!(x == 1);
  assert!(y == 0);
}
```

The `choose` function is polymorphic over type `T` and lifetime `'a`; the
lifetime annotation captures the expectation that both `x` and `y` be in the
same region.
At call site, `x` and `y` are borrowed (line 7): they become unusable,
and give birth to two intermediary values `&mut x` and `&mut y` of type
`&'a mut i32`. The value returned by `choose` also lives in region
`'a`, i.e., `z` also has type `&'a mut i32`.
The usage of `z` (lines 8-9) is valid because the region `'a` still
exists; the Rust type-checker infers that region `'a` ought to be terminated
after line 9, which ends the borrows and therefore allows the caller to regain
full ownership of `x` and `y`, so that the `assert`s at lines 10-11 are
well-formed.

At first glance, it appears we can translate `choose` to an obvious
conditional.
But if we reason about the semantics of `choose` from the
caller's perspective, it turns out that the intuitive translation is not
sufficient to capture what happens, e.g., the fact that `x` is updated while
`y` is left unchanged, as we observe at lines 10 and 11.
To solve this issue we observe that at call site, `choose` is an opaque,
separate function, meaning the caller
cannot reason about its precise definition — all that is
available is the function type. This type, however, contains precise region
information, which one can use to summarize its behavior.
When performing the function call, the ownership of `x` and
`y` is transferred to region `'a` in exchange for `z`; symmetrically,
when the lifetime
`'a` terminates, `z` is relinquished to region
`'a` in exchange for regaining ownership of `x` and `y`. The former
operation flows *forward*; the latter flows *backward*. Using a
separation-logic oriented analogy: borrows and regions encode a magic wand that
is introduced in a function call and eliminated when the corresponding region
terminates.

Our point is: both function call and region termination are semantically
meaningful. With the `choose` example,
Aeneas emits a *forward* function which itself returns a *backward* continuation.
The forward function is used to model the function call at line 7,
while the backward continuation is used to propagate changes back into `x` and `y`
when lifetime `'a` terminates.

```lean
def choose {T : Type} (b : Bool) (x : T) (y : T) :
  Result (T × (T → T × T)) :=
  if b
  then ok (x, fun z => (z, y))
  else ok (y, fun z => (x, z))

def test_choose : Result Unit := do
  let (z, choose_back) <- choose true 0#i32 0#i32
  let z1 <- z + 1#i32
  massert (z1 = 1#i32) -- monadic assert
  let (x, y) := choose_back z1
  massert (x = 1#i32)
  massert (y = 0#i32)
  ok ()

#assert (test_choose = ok ())
```

The call to `choose` returns a pair: the selected value, and a backward
continuation `choose_back`. We bind the result of the addition
(provided no overflow occurs) to `z1`; then, per the rules of Rust's type-checker,
region `'a` terminates which compels us to call the backward continuation
`choose_back`.
The intuitive effect of calling `choose_back` is as follows: we relinquish
`z1`, which was in region `'a`; doing so, we propagate any updates
that may have been performed through `z` onto the
variables whose ownership was
transferred to `'a` in the first place, namely `x` and `y`.
This bidirectional approach is akin to lenses, except we propagate
the output back to possibly-many inputs; in this case, `z` is a view
over either `x` or `y`, and the backward function reflects the update to
`z` onto the original variables.
Thus, both variables are re-bound,
before chaining the two asserts.

The key advantage of returning the backward function as a continuation is that
it captures the choice made by `choose` (i.e., whether `b` was true or false)
in a closure, so the caller does not need to pass back the original arguments.

From the caller's perspective, the computational content of `choose` is
unknown; but the signature of `choose` reveals the effect it may have onto its
inputs `x` and `y`, which in turns allows us to derive the type of the
forward and backward functions from the signature of `choose` itself.
The result is a modular, functional
translation that does not rely on any sort of cross-function inlining or
whole-program analysis.
To synthesize the backward continuation, it suffices to invert the direction
of assignments; in one case, `z` flows to `x` and `y` remains
unchanged; the other case is symmetrical.

### Recursion and Data Structures

It might not be immediately obvious that this translation technique scales up
beyond toy examples; we now crank up the complexity and
show how Aeneas can handle a wide variety of idioms while still delivering on the
original promise of a lightweight functional translation.
Our next example is
`list_nth_mut`, which allows taking a mutable reference to the *n*-th
element of a list, mutating it, and regaining ownership of the list.

```rust
enum List<T> {
  Cons(T, Box<List<T>>),
  Nil,
}

fn list_nth_mut<'a, T>(l: &'a mut List<T>, i: usize) -> &'a mut T {
  match l {
    Nil => { panic!() }
    Cons(x, tl) => {
      if i == 0 { x }
      else { list_nth_mut(tl, i - 1) }
    }
  }
}

fn sum(l: & List<i32>) -> i32 {
  match l {
    Nil => { return 0; }
    Cons (x, tl) => { return *x + sum(tl); }
  }
}

fn test_nth() {
  let mut l = Cons (1, Box::new(Cons (2, Box::new(Nil))));
  let x = list_nth_mut(&mut l, 1);
  *x = *x + 1;
  assert!(sum(&l) == 4);
}
```

This example relies on several new concepts. Parametric data type declarations
resemble those in any functional programming language such as OCaml
or SML. The `Box` type denotes a heap-allocated, uniquely-owned piece of
data. Without the `Box` indirection, `List` would describe a type of
infinite size and would be rejected. Immutable (or shared) borrows
do not sport a
`mut` keyword; they do not permit mutation, but the programmer may create
infinitely many of them. Only when all shared borrows have been relinquished
does full ownership return to the borrowed value.

The complete translation is as follows:

```lean
inductive List (T : Type) :=
| Cons : T -> List T -> List T
| Nil : List T

def list_nth_mut {T : Type} (l : List T) (i : Usize) :
  Result (T × (T -> List T)) :=
  match l with
  | Cons x tl =>
    if i = 0#usize
    then ok (x, fun x' => Cons x' tl)
    else do
      let i1 <- i - 1#usize
      let (v, list_nth_mut_back) <- list_nth_mut tl i1
      ok (v, fun v' => Cons x (list_nth_mut_back v'))
  | Nil => fail

def sum (l : List I32) : Result I32 :=
  match l with
  | Cons x tl => do
    let i <- sum tl
    x + i
  | Nil => ok 0#i32

def test_nth : Result Unit := do
  let l := Cons 1 (Cons 2 Nil)
  let (x, list_nth_mut_back) <- list_nth_mut l 1
  let x1 <- x + 1#i32
  let l1 := list_nth_mut_back x1
  let i <- sum l1
  massert (i = 4#i32)

#assert (test_nth = ok ())
```

We first focus on the caller's point of view.
Continuing with the lens analogy, we focus on (or "get") the *n*-th element of the list via
a call to `list_nth_mut`; modify the element; then
close (or "put" back) the lens, and propagate the modification back to the
list via a call to `list_nth_mut_back`.
The backward continuation is of particular interest.
In the `Nil` case (i.e., when `i = 0`), it simply updates the list to replace the head (`x`) with its
new value (`x'`).
In the `Cons` case, it updates the tail of the list by using the backward continuation
returned by the recursive call,
and reconstructs the complete list by consing the (unchanged) head value.

## An Ownership-Centric Semantics for Rust

Before explaining the functional translation above,
we must first present our
input language and its operational semantics. We now present a series of short
Rust snippets, and show in comments how our execution environments model the
effect of each statement.
The language we use is called LLBC (Low Level Borrow Calculus). It is essentially MIR
but with a reconstructed control-flow and a bit of cleaning. In effect, it looks like
Rust's surface syntax, but where everything is explicit, in particular moves and copies.

#### Mutable borrows

After line 1, `x` points to `0`, which we write `x ↦ 0`. At line 2, `px` *mutably* borrows `x` ("MB" stands for "mutable borrow"). As we
mentioned earlier, a mutable borrow grants exclusive ownership of a value, and
renders the borrowed value unusable for the duration of the borrow. We reflect this
fact in our execution environment as follows: `x` is marked as
"loaned-out" ("ML" stands for "mutable loan"), in a mutable fashion, and `px` is known to be a
mutable borrow. Furthermore, ownership of the borrowed value now
rests with `px`, so the value within the mutable borrow is 0. Finally, we
need to record that `px` is a borrow *of `x`*: we issue a fresh loan
identifier ℓ that ties `x` and `px` together.
The same operation is repeated at line 3. Value 0 is now held by `ppx`, and
`px`, too, becomes "loaned out".

```rust
let mut x = 0;       // x ↦ 0
let mut px = &mut x; // x ↦ ML ℓ,   px ↦ MB ℓ 0
let ppx = &mut px;   // x ↦ ML ℓ,   px ↦ ML ℓ',     ppx ↦ MB ℓ' (MB ℓ 0)
```

Our environments thus precisely track ownership; doing so, they capture the aliasing
graph in an exact fashion. Another point about our style:
this representation allows us to adopt a *focused* view of the borrowed
value (e.g., 0), solely
through its owner (e.g., `ppx`), without worrying about following indirections
to other variables.
We believe this
approach is unique to our semantics; it has, in our experience, greatly
simplified our reasoning and in particular the functional translation.

We remark that our style
departs from Stacked Borrows, where the modified value
remains with `x`. We also note that our formalism cannot account for unsafe
blocks; allowing unfettered aliasing would lead to potential cycles, which we
cannot represent. This is an intentional design choice for us: we circumbscribe
the problem space in order to achieve an intuitive, natural semantics and a
lightweight functional translation. Aeneas shines on non-unsafe Rust programs,
and can be complemented by more advanced tools such as RustBelt for unsafe
parts.

#### Shared borrows

Shared borrows behave more like traditional pointers. Multiple shared borrows
may be created for the same value; each of them grants read-only access to the
underlying value.
The owner also retains a read-only access to the borrowed value;
regaining full ownership requires
terminating all of the borrows. In the example
below, the value (0, 1) is borrowed in a shared fashion, at line 2. This time,
the value remains with `x`; but taking an immutable reference to
`x` still requires book-keeping. We issue a new loan ℓ, and record that
`px1` is now a shared borrow associated to loan ℓ; to understand which
value `px1` points to, we simply look up in the environment who is the owner
of ℓ, and read the associated value.
Repeated shared borrows are permitted: at line 3,
we create a new shared borrow of `x`; as `x` is already borrowed we do not need to
update its value and simply reuse the loan identifier ℓ.
At line 4, we copy the first component of `x`; remember that moves and copies are explicit
in our language. Values that are loaned
immutably, like `x`, can still be read; in the resulting environment, `y` points to a copy of
the first component, and bears no relationship whatsoever to `x`.
Finally, at line 5, we *reborrow* (through `px1`) the first component of
the pair only. First, to dereference `px1`, we perform a lookup and find that
`x` owns ℓ. Then, we perform book-keeping and update the value loaned by
`x`, so as to reflect that its first component has been loaned out, introducing a fresh
loan identifier ℓ' at the same time.

```rust
let x = (0, 1);    // x ↦ (0, 1)
let px1 = &x;      // x ↦ SL ℓ (0,1),       px1 ↦ SB ℓ
let px2 = &x;      // x ↦ SL ℓ (0, 1),      px1 ↦ SB ℓ,   px2 ↦ SB ℓ
let y = copy x.0;  // x ↦ SL ℓ (0, 1),      px1 ↦ SB ℓ,   px2 ↦ SB ℓ,   y ↦ 0
let z = &(*px1.0); // x ↦ SL ℓ ((SL ℓ' 0), 1), px1 ↦…, px2 ↦ …, y ↦ 0, z ↦ SB ℓ'
```

Note that `px1` and `px2` share the *same* loan identifier ℓ: shared
borrow identifiers are not unique. This is in contrast with mutable borrows, where
each borrow has a unique loan identifier to enforce exclusive ownership.
For shared borrows, since no mutation is allowed, there is no need to distinguish
between different borrows of the same value — they all grant the same read-only
access. As a consequence, ending a shared loan simply requires checking that there
are no more live borrows referencing that loan.

In our presentation, shared borrows behave like pointers, and every one of them
is statically accounted for via the loan identifier attached to the borrowed value.
The reader might find this design choice surprising: indeed, in Rust, shared
borrows behave like immutable values and we ought to be able to treat them as
such. Recall, however, that one of our key design goals is to give a
*semantic* explanation of borrows; as such, our precise tracking of
shared borrows allows us to know precisely when all aliases
have been relinquished, and full ownership of the borrowed value has been
regained. This allows us to justify why, in the example below, the update to
`x` is sound; without our exact alias tracking, we would have to trust the
borrow checker, something we explicitly do not want to do.

```rust
let mut x = 0;  // x ↦ 0
let px1 = &x;   // x ↦ SL ℓ 0,       px1 ↦ SB ℓ
let px2 = &x;   // x ↦ SL ℓ 0,       px1 ↦ SB ℓ,   px2 ↦ SB ℓ
                // After ending the loan ℓ:
                // x ↦ 0,   px1 ↦ ⊥,   px2 ↦ ⊥
x = 1;          // x ↦ 1
```

Finally, we reiterate our remark that our formalism allows keeping track of the aliasing
graph in a precise fashion; the discipline of Rust bans cycles, meaning that the
aliasing graph is always a tree. This style of representation resembles
Mezzo, where loan identifiers are akin to singleton
types, and entries in the environment are akin to permissions.

#### Rewriting an Old Value, a.k.a. Reborrowing

We now consider a particularly twisted example accepted by the Rust compiler. While the
complexity seems at first gratuitous, it turns out that the pattern of borrowing
a dereference (i.e., `&mut (*px)`) is particularly common in Rust. The reason
is subtle: in the post-desugaring MIR internal Rust representation, moves and
copies are explicit, meaning function calls of the form `f(move px)` abound.
Such function calls *consume* their argument, and render the
user-declared reference `px` unusable past the function call.
To offer a better user experience, Rust automatically "reborrows" the
contents pointed to by `px`, and rewrites the call into `f(move (&mut (*px)))` at desugaring-time.
Thus, only the intermediary value is "lost" to the function call;
relying on its lifetime analysis, the Rust compiler concludes that the
user-declared reference `px` remains valid past the function call, hence
making the programmer's life easier.

Another common pattern is to directly mutate a borrow, i.e., assign a fresh
borrow into a variable `x` of type `&mut t` that was *itself*
declared as `let mut`.
Capturing the semantics of such an update must be done with great care, in order
to preserve precise aliasing information.

We propose an example that combines both patterns; the fact that we make
`px` reborrow itself is what makes the example "twisted".
Rust accepts this
program; we now explain with our semantics *why* it is sound.
In the example below, after line
2, the environment offers no surprises. Justifying the write at line 3 requires
care. We borrow `*px`, which modifies `px` to point to `MB ℓ' 0`,
and returns `ML ℓ'`; the value about to be overwritten is stored in a
fresh variable `px_old`, and `ML ℓ'`
gets written to `px`.

```rust
let mut x = 0;       // x ↦ 0
let mut px = &mut x; // x ↦ ML ℓ,   px ↦ MB ℓ 0
px = &mut (*px);     // x ↦ ML ℓ,   px_old ↦ MB ℓ (ML ℓ'),   px ↦ MB ℓ' 0
                     // x ↦ ML ℓ,   px_old ↦ MB ℓ 0, px ↦⊥
assert!(x==0);       // x ↦ 0, px_old ↦ ⊥, px ↦ ⊥
```

Saving the old value is crucial for line 4.
For the assertion, we need to regain full ownership of `x`. To do so,
we first terminate ℓ'.
This *reorganizes* the environment, with two consequences.
First, `px` becomes unusable, which we write `px ↦ ⊥`. Second, `px_old`, which we had judiciously kept in the
environment, becomes `MB ℓ 0`. We reorganize the environment again, to
terminate ℓ; the effect is similar, and results in `x ↦ 0`, i.e., full ownership of `x`.
This example illustrates a key characteristic of our approach, which is that we
reorganize borrows in a lazy fashion, and don't terminate a borrow until we
need to get the borrowed value back.

#### An Illegal Borrow

We offer an final example which leverages our reorganization rules;
furthermore, the example illustrates how our semantics reaches the same
conclusion as `rustc`, though by different means, on a borrowing error.
At line 2, a borrow is introduced, which results in a fresh loan ℓ.
To make progress, we
terminate borrow ℓ at line 3.
Line 4 then type-checks, with a fresh borrow ℓ'. Then, we error
out at line 5: `px₁` has been terminated, and we cannot dereference ⊥.

The Rust compiler proceeds differently, and implements an analysis which
requires computing borrow constraints for an entire function body.
The compiler notices that lifetime ℓ must go on until line 6 (because of
the assert), which prevents a new borrow from being issued at line 4. Rust
thus ascribes the error to an earlier location than we do, that is, line 4.
We remark that the Rust behavior is semantically equivalent to ours; however, our
lazy approach which terminates borrows only as needed has the advantage that
evaluation can proceed in a purely forward fashion, without requiring a
non-local analysis. This on-demand approach is similar to Stacked
Borrows.

```rust
let mut x = 0;    // x ↦ 0
let px1 = &mut x; // x ↦ ML ℓ,   px1 ↦ MB ℓ 0
                  // x ↦ 0,   px1 ↦ ⊥
let px2 = &mut x; // aeneas: x ↦ ML ℓ',   px1 ↦ ⊥,    px2 ↦ MB ℓ' 0
                  // rustc: error: cannot borrow `px1` as mutable more than once at a time
assert!(*px1 == 0); // aeneas: error, attempt to deference unusable variable px1
```

#### Two-Phase Borrows

We finally review *two-phase* borrows, which are introduced by the Rust compiler
at desugaring time. A two-phase borrow starts as a shared borrow in a
"reservation phase", and later gets activated into a full mutable borrow.
Reserved borrows enable a variety of very common idioms without resorting to
more advanced desugarings.

Below, we create a two-phase borrow at line 3. From the point of view of the
lender, a two-phase borrow acts as a shared borrow; the value of `x`, which
is already immutably borrowed, remains unchanged. However, `px2` maps to the
*reserved* borrow `RB ℓ`.
At line 4, we evaluate the assertion as before.
However, at line 5 we need to use the two-phase borrow to perform an in-place
update. We thus reorganize the environment by ending the shared borrow ℓ
contained by `px1`. There now only remains a single borrow pointing to `x`,
the reserved borrow of `px2`, that we can promote to a mutable borrow. This
allows us to evaluate the in-place update.

```rust
let mut x = 0;          // x ↦ 0
let px1 = &x;           // x ↦ SL ℓ 0,       px1 ↦ SB ℓ
let px2 = &two-phase x; // x ↦ SL ℓ 0,       px1 ↦ SB ℓ,   px2 ↦ RB ℓ
assert!(*px1 == 0);     // x ↦ SL ℓ 0,       px1 ↦ SB ℓ,   px2 ↦ RB ℓ
                        // After ending the shared borrow of px1:
                        // x ↦ SL ℓ 0,       px1 ↦ ⊥,      px2 ↦ RB ℓ
                        // After promoting the reserved borrow of px2:
                        // x ↦ ML ℓ,         px1 ↦ ⊥,      px2 ↦ MB ℓ 0
*px2 = 1;               // x ↦ ML ℓ,         px1 ↦ ⊥,      px2 ↦ MB ℓ 1
```

## From Symbolic Semantics to Functional Code

We now explain how Aeneas, using the symbolic semantics, generates a pure translation
of the original LLBC program. We show this through two detailed examples:
first the translation of a *caller* (`call_choose`), then the translation
of a *callee* (`choose`) which requires synthesizing a backward function.

The translation is carried out by performing a symbolic execution on the source
program and synthesizing a functional AST in parallel. At each step, we show
the symbolic environment alongside the progressively-built Lean translation.

A key insight is that the synthesis rules are exclusively concerned with
*symbolic values*; the actual variables from the source program (`x ↦ ...`)
are mere bookkeeping devices and have no relevance to the translated program.
Symbolic values σ, however, cannot be determined statically; they thus compute
at run-time, and as such are let-bound in the target program.

### Translation Example: `call_choose`

As a starting example, we consider the translation of the `call_choose` function
below, presented in LLBC syntax with explicit writes and moves, along with a
fully-explicit return variable `x_ret`.

```rust
fn call_choose(mut p : (u32, u32)) -> u32 {
  let px = &mut p.0;                       // line 2
  let py = &mut p.1;                       // line 3
  let pz = choose(true, move px, move py); // line 4
  *pz = *pz + 1;                           // line 5
  x_ret = move p.0;                        // line 6
  return;
}
```

We start the translation by initializing an environment where `p` maps to a
symbolic value σ₀. In parallel, we synthesize the function as an AST with a
hole `[.]` to be progressively filled.

**Environment:**
```
p ↦ (σ₀ : (u32, u32))
```

**Synthesized code:**
```lean
def call_choose (s0 : (U32 × U32)) :
  Result U32 := do
  [.]
```

At line 2, accessing field 0 requires us to expand σ₀; doing so introduces a
let-binding in the translation.

**Environment:**
```
p ↦ (σ₁ : u32, σ₂ : u32)
```

**Synthesized code:**
```lean
def call_choose (s0 : (U32 × U32)) :
  Result U32 := do
  let (s1, s2) := s0
  [.]
```

The expansion allows us to evaluate the two mutable borrows on lines 2–3.
Importantly, borrows and assignments just lead to bookkeeping in the
environment: the synthesized translation is left unchanged.

**Environment:**
```
p  ↦ (ML ℓ₁, ML ℓ₂)
px ↦ MB ℓ₁ (σ₁ : u32)
py ↦ MB ℓ₂ (σ₂ : u32)
```

**Synthesized code:**
```lean
def call_choose (s0 : (U32 × U32)) :
  Result U32 := do
  let (s1, s2) := s0
  [.]
```

We then reach the function call at line 4. To account for the call, we introduce
a region abstraction to account for `'a`, transfer ownership of the effective
arguments to the abstraction, then introduce a fresh mutably borrowed value
`MB ℓ₃ (σ₃ : u32)` to account for the returned value stored in `pz`.
We also attach a *continuation* to the region abstraction A(α) that we
will use upon ending A(α). This continuation is simply the backward function
of `choose`; it consumes the value retrieved upon ending ℓ₃ and produces
the values to give back to ℓ₁ and ℓ₂.

In parallel, we introduce a call to `choose` in the synthesized translation. The
borrow types are translated to the identity; the input arguments are thus simply
`s1` and `s2`, while `choose` outputs `s3`, together with the backward
function `back`.

**Environment:**
```
p  ↦ (ML ℓ₁, ML ℓ₂)
px ↦ ⊥
py ↦ ⊥

A(α) {
    _,
    MB ℓ₁ (σ₁ : u32),
    MB ℓ₂ (σ₂ : u32),
    ML ℓ₃,
} ⟦ (ℓ₁, ℓ₂): λ ℓ₃ ⇒ back ℓ₃ ⟧

pz ↦ MB ℓ₃ (σ₃ : u32)
```

**Synthesized code:**
```lean
def call_choose (s0 : (U32 × U32)) :
  Result U32 := do
  let (s1, s2) := s0
  let (s3, back) ← choose true s1 s2
  [.]
```

We can now symbolically execute the increment, which merely introduces an
addition and generates a fresh variable σ₄:

**Environment:**
```
p  ↦ (ML ℓ₁, ML ℓ₂)
px ↦ ⊥
py ↦ ⊥

A(α) {
    _,
    MB ℓ₁ (σ₁ : u32),
    MB ℓ₂ (σ₂ : u32),
    ML ℓ₃,
} ⟦ (ℓ₁, ℓ₂): λ ℓ₃ ⇒ back ℓ₃ ⟧

pz ↦ MB ℓ₃ (σ₄ : u32)
```

**Synthesized code:**
```lean
def call_choose (s0 : (U32 × U32)) :
  Result U32 := do
  let (s1, s2) := s0
  let (s3, back) ← choose true s1 s2
  let s4 ← s3 + 1
  [.]
```

Finally, the move at line 6 requires retrieving the ownership of `p.0`. Doing
so requires ending the region abstraction A(α) (introduced by the call to
`choose`), which in turn requires ending the loan inside the abstraction.
Accordingly, we first end ℓ₃ which leads to the environment below.
Importantly, we update the continuation associated with A(α) to account for
the fact that it consumes the value given back upon ending ℓ₃.
Ending a loan (or a borrow, depending on the point of view) leaves the
synthesized code unchanged.

**Environment:**
```
p  ↦ (ML ℓ₁, ML ℓ₂)
px ↦ ⊥
py ↦ ⊥

A(α) {
    _,
    MB ℓ₁ (σ₁ : u32),
    MB ℓ₂ (σ₂ : u32),
    (σ₄ : u32)
} ⟦ (ℓ₁, ℓ₂): back σ₄ ⟧

pz ↦ ⊥
```

Then, we actually end the region abstraction A(α) by moving back the borrows
ℓ₁ and ℓ₂ in the environment, with fresh symbolic values σ₅ and σ₆. Those
are the values given back *by* `choose`. On the side of the translated code,
we materialize the end of A(α) by introducing a call to its continuation. This
continuation consumes the value given back to the loan ℓ₃ upon ending it (that
is, σ₄); it also outputs the values given back to ℓ₁ and ℓ₂ (that is, the
pair (σ₅, σ₆)).

**Environment:**
```
p  ↦ (ML ℓ₁, ML ℓ₂)
px ↦ ⊥
py ↦ ⊥
_  ↦ MB ℓ₁ (σ₅ : u32)
_  ↦ MB ℓ₂ (σ₆ : u32)
pz ↦ ⊥
```

**Synthesized code:**
```lean
def call_choose (s0 : (U32 × U32)) :
  Result U32 := do
  let (s1, s2) := s0
  let (s3, back) ← choose true s1 s2
  let s4 ← s3 + 1
  let (s5, s6) ← back s4
  [.]
```

We can finally end the borrow ℓ₁ and evaluate the `return`, which ends the
translation. As we save meta-information about the assignments to generate
suitable names for the variables, Aeneas actually generates the following
function:

```lean
def call_choose (p : (U32 × U32)) : Result U32 := do
  let (px, py) := p
  let (pz, back) ← choose true px py
  let pz0 ← pz + 1
  let (px0, _) ← back pz0
  ok px0
```

### Translation Example: `choose`

We now proceed with the synthesis of `choose`, which requires synthesizing a
backward function. We recall its definition; like in the previous example,
we make all the statements explicit.

```rust
fn choose<'a, T>(b : bool, x : &'a mut T, y : &'a mut T) -> &'a mut T {
  if b {
    x_ret = move x;
    return;
  }
  else {
    x_ret = move y;
    return;
  }
}
```

Unlike `call_choose`, the `choose` function takes borrows as input parameters.
We thus need to track their provenance, from the point of view of the callee.
As a consequence, we initialize the environment by introducing an abstraction
containing loans so as to model the values owned by the caller and *loaned* to
the function for as long as α lives.
This is the dual of the caller's point of view: the abstraction contains
two mutable borrows ℓ⁰ₓ and ℓ⁰ᵧ, whose loans are not in the environment;
they stand for the values we consumed upon calling the function.
The abstraction also contains two mutable loans ℓₓ and ℓᵧ, associated to the
borrows of the input values `x` and `y`.
Importantly, we attach a continuation that we will use for the synthesis, and
which is currently the identity; intuitively, this means that ℓ⁰ₓ and ℓₓ are
actually the same (and similarly for ℓ⁰ᵧ and ℓᵧ). The link between borrows
and loans inside the input region abstraction will become more complex as we
proceed through the synthesis.

**Environment:**
```
A_input(α) {
    MB ℓ⁰ₓ (_),
    MB ℓ⁰ᵧ (_),
    ML ℓₓ,
    ML ℓᵧ,
} ⟦ (ℓ⁰ₓ, ℓ⁰ᵧ): λ ℓₓ ℓᵧ ⇒ (ℓₓ, ℓᵧ) ⟧

b ↦ σ_b
x ↦ MB ℓₓ (σₓ : T)
y ↦ MB ℓᵧ (σᵧ : T)
```

**Synthesized code:**
```lean
def choose
  {T : Type} (b : Bool) (x y : T) :
  Result (T × (T → (T × T))) := do
  [.]
```

We then evaluate the `if`, branching over the symbolic value σ_b.
We duplicate the environment and substitute σ_b with `true` for the first
branch, and `false` for the second branch.
Below, we show the environment for the first branch of the `if`:

**Environment (first branch):**
```
A_input(α) {
    MB ℓ⁰ₓ (_),
    MB ℓ⁰ᵧ (_),
    ML ℓₓ,
    ML ℓᵧ,
} ⟦ (ℓ⁰ₓ, ℓ⁰ᵧ): λ ℓₓ ℓᵧ ⇒ (ℓₓ, ℓᵧ) ⟧

b ↦ true
x ↦ MB ℓₓ (σₓ : T)
y ↦ MB ℓᵧ (σᵧ : T)
```

**Synthesized code:**
```lean
def choose
  {T : Type} (b : Bool) (x y : T) :
  Result (T × (T → (T × T))) := do
  if b then [.]
  else [.]
```

We proceed with the evaluation of the first branch. Upon reaching `x_ret = move x`,
we move `x` to the return variable; the synthesized code is left unchanged.

**Environment:**
```
A_input(α) {
    MB ℓ⁰ₓ (_),
    MB ℓ⁰ᵧ (_),
    ML ℓₓ,
    ML ℓᵧ,
} ⟦ (ℓ⁰ₓ, ℓ⁰ᵧ): λ ℓₓ ℓᵧ ⇒ (ℓₓ, ℓᵧ) ⟧

b     ↦ true
x     ↦ ⊥
y     ↦ MB ℓᵧ (σᵧ : T)
x_ret ↦ MB ℓₓ (σₓ : T)
```

We then reach the `return`. This is the crucial part of the translation. We
need to transform the environment so that it matches a target environment given
by the signature of `choose`. In the process, we will introduce and merge
region abstractions, thus progressively synthesizing the backward function for
the first branch of the `if`.

We first move the values of `b` and `y` to anonymous values, and eliminate the
value of `b` as it doesn't contain any borrows; this has no effect on the
synthesis. We then transform the borrow moved from `y` into a region
abstraction A₀. This region abstraction has no inputs, as it doesn't contain
mutable loans, and a single output, the value to give back for the mutable
borrow ℓᵧ; we thus attach a continuation stating that upon ending A₀ we
retrieve a value for ℓᵧ, which is actually σᵧ.

**Environment:**
```
A_input(α) {
    MB ℓ⁰ₓ (_),
    MB ℓ⁰ᵧ (_),
    ML ℓₓ,
    ML ℓᵧ,
} ⟦ (ℓ⁰ₓ, ℓ⁰ᵧ): λ ℓₓ ℓᵧ ⇒ (ℓₓ, ℓᵧ) ⟧

b     ↦ ⊥
x     ↦ ⊥
y     ↦ ⊥
x_ret ↦ MB ℓₓ (σₓ : T)

A₀ {
    MB ℓᵧ (_)
} ⟦ ℓᵧ: σᵧ ⟧
```

We perform one last step by merging A_input(α) and A₀ together. Merging the
content of region abstractions is done as before; in particular the borrow and
the loan for ℓᵧ cancel out. Merging two region abstractions also requires
composing their continuations. Importantly, the fact that a borrow and a loan
cancel out is mirrored by the fact that the continuation of A_input(α) consumes
a borrow (ℓᵧ) which is actually an output of A₀. The resulting continuation of
A_input(α) thus only has one input, the value consumed upon ending ℓₓ.

This yields an environment which matches our target environment: `x_ret` maps
to a valid borrow, all the other local variables map to ⊥, and we have a
single region abstraction for α which acts as an interface between the caller
and the callee. We can thus end the translation of the first branch: the
returned value is the value inside `x_ret` (i.e., σₓ) while the backward
function is given by the continuation attached to A_input(α).

**Environment:**
```
A_input(α) {
    MB ℓ⁰ₓ (_),
    MB ℓ⁰ᵧ (_),
    ML ℓₓ,
} ⟦ (ℓ⁰ₓ, ℓ⁰ᵧ): λ ℓₓ ⇒ (ℓₓ, σᵧ) ⟧

b     ↦ ⊥
x     ↦ ⊥
y     ↦ ⊥
x_ret ↦ MB ℓₓ (σₓ : T)
```

**Synthesized code:**
```lean
def choose
  {T : Type} (b : Bool) (x y : T) :
  Result (T × (T → (T × T))) := do
  if b then ok (x, fun x0 => (x0, y))
  else [.]
```

The translation of the second branch is similar; we omit it.

## Soundness

We claimed that a symbolic interpreter for LLBC implements a borrow-checker for
Rust; we now summarize the formal argument that substantiates this claim.
The full details are in the ICFP 2024 paper and my thesis manuscript.

The soundness of LLBC's symbolic semantics (LLBCS) is predicated on the LLBC
model being a sound foundation. LLBC has several unusual features, such as
attaching values to pointers rather than to the underlying memory location, or
not relying on an explicit heap; it therefore requires a formal argument to
establish that this is an acceptable way to model Rust programs. We remark that
this question is orthogonal to the RustBelt line of work: RustBelt establishes
the soundness of Rust's type system with regards to λ_Rust, whose classic,
unsurprising semantics does not warrant scrutiny. Clarifying the link between
LLBC and a standard heap-and-addresses model is not just a matter of theory:
once the Rust compiler emits LLVM bitcode, the heap and addresses become real.

We address this question in two steps:

1. We introduce PL, for "pointer language", which uses a traditional, explicit
   heap inspired by CompCert's C memory model, and show that it refines LLBC.
   This establishes that a low-level model (where values live at a given
   address) refines the Rust model given by LLBC (where borrows hold the value
   they are borrowing).

2. We prove that LLBC itself refines the symbolic version of LLBC (LLBCS).
   Combined with the previous result, this allows us to precisely state why
   LLBCS is a borrow-checker for LLBC: if an LLBCS execution succeeds, then any
   corresponding low-level PL execution is safe for all inputs.

More precisely, a program which is successfully evaluated following the symbolic
semantics is *memory safe*, and its evaluation following the semantics of LLBC
is in *bisimulation* with an evaluation using the heap-based PL semantics.

To conduct these proofs of refinement, we introduce a novel proof technique that
relies on the fact that our languages operate over the same grammar of
expressions, but give it different *meanings*, or *views*. Rather than go
full-throttle with a standard compilation-style proof of simulation, we reason
modularly over local or pointwise transformations that rewrite one piece of
state to another — proofs over those elementary transformations are much easier.
We then show that two states that are related by the transitive closure of these
elementary transformations continue to relate throughout their execution,
ultimately giving a proof of refinement.

We note that the functional translation is currently trusted: its correctness is
not yet formally established, and is left as future work.

Finally, there is an ongoing effort to mechanize these proofs, which are
currently pen-and-paper. The goal is to obtain a machine-checked formalization
of the soundness of LLBCS as a borrow-checker.

## Beyond the Rust Borrow-Checker

Because our approach to borrow-checking is semantic rather than syntactic, Aeneas
can actually borrow-check (and translate) programs which are not accepted by
Rust's current borrow checker.

### A Limitation of Rust's Borrow-Checker

In the process of testing Aeneas on B-epsilon trees, we bumped into a limitation
of the current Rust borrow-checking algorithm. The `get_suffix_at_x` function
below looks for an element in a list and returns a mutable borrow to the suffix
starting at that element. The current Rust borrow-checker is too coarse to
notice that the `Cons` branch is valid. More specifically, it considers that the
reborrows performed through `hd` and `tl` should last until the end of the
lexical scope, that is, until the end of the `Cons` branch.

Instead, one may notice that it is possible to end those borrows earlier, after
evaluating the conditional, in order to retrieve full ownership of the value
borrowed by `ls` in the first branch of the `if`, and make the example
borrow-check. The ongoing replacement of the borrow-checker in Rust, named
Polonius, implements a more refined lifetime analysis and accepts this program.

More interestingly, our semantic approach of borrows makes this program
borrow-check without issues; since our discipline is based on symbolic execution
and a semantic approach to loans, we accept the example without troubles, and
are resilient to further syntactic tweaks of the program.

```rust
fn get_suffix_at_x<'a>(ls: &'a mut List<u32>, x: u32) -> &'a mut List<u32> {
  match ls {
    Nil => { ls }
    Cons(hd, tl) => { // error: first mutable borrow occurs here
      if *hd == x { ls // second mutable borrow occurs here
      } else { get_suffix_at_x(tl, x) } } } }
```

### Precise Reborrows

Because of the way we handle reborrows, there are actually cases of programs
deemed invalid even by Polonius, but supported by Aeneas.

In the example below, we first create a shared borrow that we store in `pp`,
then a reborrow of a subvalue borrowed by `pp` that we store in `px`. Upon
evaluating the assignment `p.1 = 2`, `px` simply maps to a shared borrow of the
first component of `p`. Importantly, even though `px` reborrows part of `pp`,
there are no links between `px` and `pp`: our semantics does not track the
hierarchy between borrows and their subsequent reborrows. In other words,
`let px = &(*pp.0)` is equivalent to `let px = &p.0`, where we borrow directly
from `p` without resorting to `pp`. This implies that, upon ending the borrow
ℓ_p stored in `pp` at the assignment, we do not need to end ℓ_x stored in `px`,
which in turn allows us to legally evaluate the assertion.

```rust
let mut p = (0, 1);
let pp = &p;
// p  ↦ SL ℓ_p (0, 1)
// pp ↦ SB ℓ_p
let px = &(*pp.0);
// p  ↦ SL ℓ_p (SL ℓ_x 0, 1)
// pp ↦ SB ℓ_p
// px ↦ SB ℓ_x
p.1 = 2;
// p  ↦ (SL ℓ_x 0, 2)
// pp ↦ ⊥
// px ↦ SB ℓ_x
assert!(*px == 0);
```

When we attempt to borrow-check this program (with Polonius or the current
implementation of the borrow checker), the Rust borrow checker considers that
`px` reborrows `pp`, and thus needs to end before `pp` ends.

The code snippet below illustrates a similar example with mutable borrows.
We create a borrow `px1` of (the value of) `x`, then reborrow this value
through `px2`. We then update `px1` to borrow `y`. At this point, `px2` still
borrows `x`. The important point to notice is that upon performing this update,
we remember the old value of `px1` in an anonymous variable to not lose
information about the borrow graph. Similarly to the previous example with
shared borrows, the resulting environment doesn't track the fact that `px2` was
created by reborrowing the value initially borrowed by `px1`: there are no links
between those two variables. Consequently, upon ending borrow ℓ_y (stored in
`px1`) to access `y`, we don't need to end ℓ₂ (stored in `px2`). This in
return allows us to legally dereference `px2` at the end.

```rust
let mut x = 0;
let mut px1 = &mut x;
let px2 = &mut (*px1); // Reborrow: px2 now borrows (the value of) x
// x   ↦ ML ℓ₁
// px1 ↦ MB ℓ₁ (ML ℓ₂)
// px2 ↦ MB ℓ₂ 0
let mut y = 1;
px1 = &mut y; // Update px1 to borrow y instead of x
// x   ↦ ML ℓ₁
// _   ↦ MB ℓ₁ (ML ℓ₂)
// px2 ↦ MB ℓ₂ 0
// y   ↦ ML ℓ_y
// px1 ↦ MB ℓ_y 1
assert!(*px1 == 1);
assert!(*px2 == 0);
assert!(y == 1); // End the borrow of y through px1 (shouldn't impact px2!)
// x   ↦ ML ℓ₁
// _   ↦ MB ℓ₁ (ML ℓ₂)
// px2 ↦ MB ℓ₂ 0
// y   ↦ 1
// px1 ↦ ⊥
assert!(*px2 == 0); // Considered invalid by rustc, but accepted by Aeneas
```

The two examples above exemplify cases where both the Rust borrow checker and
Polonius deem a program as invalid, while Aeneas accepts it. We do not claim
that this is a strong limitation of the Rust borrow checker: these use cases
seem quite anecdotal and are probably useless in practice. However, we believe
the ability of Aeneas to precisely capture the behavior of such use cases
supports our claim that our semantics really captures the essence of the borrow
mechanism.

More examples of programs rejected by the Rust compiler but accepted by Aeneas
can be found in the
[test suite](https://github.com/AeneasVerif/aeneas/blob/main/tests/src/rust-borrow-check-issues.rs).

## Verifying Rust Programs with Aeneas

Aeneas is being used in practice to verify real-world Rust code. In particular,
Microsoft is using Aeneas to verify implementations of the
[SymCrypt](https://github.com/microsoft/SymCrypt) cryptographic library, as
described in their
[blog post](https://www.microsoft.com/en-us/research/blog/rewriting-symcrypt-in-rust-to-modernize-microsofts-cryptographic-library/).
We now showcase the verification workflow through a few examples.

### A Simple Arithmetic Example

Consider `mul2_add1`, which Aeneas translates as follows:

```rust
pub fn mul2_add1(x: u32) -> u32 {
    (x + x) + 1
}
```

```lean
def mul2_add1 (x : U32) : Result U32 := do
  let x1 ← x + x
  let x2 ← x1 + 1#u32
  ok x2
```

Le us show how to prove that `mul2_add1` evaluates to `2 * x + 1` provided there is no overflow.
We write specifications in the Hoare-logic style: `mul2_add1 x ⦃ y => y.val = 2 * x.val + (1 : Nat) ⦄`
below means that `mul2_add1 x` successfully evaluates to some value `y` (in particular, it doesn't panic)
that satisfies `y.val = 2 * x.val + 1` (`x.val` means `x` seen as an unbounded mathematical integer).

```lean
theorem mul2_add1_spec (x : U32) (h : 2 * x.val + 1 ≤ U32.max) :
  mul2_add1 x ⦃ y => y.val = 2 * x.val + 1 ⦄ := by
  unfold mul2_add1
  progress as ⟨ i ⟩
  progress as ⟨ i' ⟩
  scalar_tac
```

Initially, Lean shows us the following goal to be proven:

```
x : U32
h : 2 * x.val + 1 ≤ U32.max
⊢ mul2_add1 x ⦃ y => y.val = 2 * x.val + 1 ⦄
```

Proving that this holds looks a lot like a debugging session: the first thing we need to do
is dive into the body of `unfold mul2_add1`. We can do so with the `unfold` tactic ("tactic" is the
name of the instructions that are used to guide the proofs), revealing:

```
x : U32
h : 2 * x.val + 1 ≤ U32.max
⊢ (do
    let x1 ← x + x
    let x2 ← x1 + 1#u32
    ok x2) ⦃
  y => y.val = 2 * x.val + 1 ⦄
```

We now need to step through the function body and provide a dedicated tactic, `progress`.
The `progress` tactic looks at the first monadic operation in the goal, finds a
matching specification and applies it, introducing the result variable. The
`as ⟨ i ⟩` clause names it.
The first `progress as ⟨ i ⟩` processes `x + x`, proving on the fly that there is no overflow
(we have `x.val + x.val ≤ U32.max`) and updating the context accordingly:
```
x : U32
h : 2 * x.val + 1 ≤ U32.max
i : U32
_ : i.val = x.val + x.val
⊢ (do
    let x2 ← i + 1#u32
    ok x2) ⦃
  y => y.val = 2 * x.val + 1 ⦄
```

The second `progress as ⟨ i' ⟩` removes the second addition, leaving a pure arithmetic goal:

```
i : U32
_ : i.val = x.val + x.val
i' : U32
_ : i'.val = i.val + 1
⊢ i'.val = 2 * x.val + 1
```

This goal can trivially be solved by using an arithmetic solver (`scalar_tac` in our case).

### Using Registered Theorems

Consider a caller of `mul2_add1`:

```rust
pub fn mul2_add1_add(x: u32, y: u32) -> u32 {
    mul2_add1(x) + y
}
```

```lean
def use_mul2_add1 (x : U32) (y : U32) : Result U32 := do
  let x1 ← mul2_add1 x
  x1 + y
```

When doing the proof, we want to reuse the specification we just proved for `mul2_add1`.
We can do so by explicitly instructing `progress` to use `mul2_add1_spec` through the
`with` keyword: `progress with mul2_add1_spec`. Even better, if we mark `mul2_add1_spec`
with the `@[progress]` attribute, `progress` can automatically look it up when encountering
`mul2_add1`. This looks like this:

```lean
@[progress]
theorem mul2_add1_spec (x : U32) (h : 2 * x.val + 1 ≤ U32.max) :
  mul2_add1 x ⦃ y => y.val = 2 * x.val + 1 ⦄ := by
  ...

theorem use_mul2_add1_spec (x : U32) (y : U32)
  (h : 2 * x.val + 1 + y.val ≤ U32.max) :
  use_mul2_add1 x y ⦃ z => z.val = 2 * x.val + (1 : Nat) + y.val ⦄ := by
  unfold use_mul2_add1
  progress as ⟨ i ⟩    -- automatically uses mul2_add1_spec
  progress as ⟨ i' ⟩
  scalar_tac
```

### Constant-Time Modular Addition

Let's now look at a real example.

Cryptographic code often requires constant-time implementations to prevent
side-channel attacks: if the execution time of a cryptographic operation depends
on the value of a secret key, an attacker can measure that time across many invocations
and statistically recover the key. For this reason, cryptographic
implementations must avoid branches and data-dependent operations that would
cause the execution time to vary depending on the value of their secret inputs.

The following example appears in the SymCrypt implementation of ML-KEM, where we
need to do operations modulo prime number 3329.
In particular,  given `a` and `b` two numbers smaller than (the prime number) 3329,
the following function computes `(a + b) % 3329`.
It cannot use `%` — the division and conditional branch would produce
variable-time code. Instead, the implementation uses a branchless trick based
on underflow and bitmasking:

```rust
fn mod_add(a: u32, b: u32) -> u32 {
    assert!(a < 3329);
    assert!(b < 3329);
    let sum = a + b;                  // 0 <= a + b <= 2 * 3328
    let res = sum.wrapping_sub(3329);
    let mask = res >> 16;             // mask = 0xffff if a + b < 3329, 0 otherwise
    let q = 3329 & mask;              // q = 3329 if a + b < 3329, 0 otherwise
    res.wrapping_add(q)
}
```

Because of properties of the modulo, either `a + b` is smaller than 3329, in which
case `(a + b) % 3329 = a + b`, or it is greater, in which case we have:
`(a + b) % 3329 = a + b - 3329`.
This means that computing the modulo simply requires conditionally subtracting 3329.
Same as with the modulo operation, we can not branch on `a + b < 3329`, as it would 
lead to non-constant time code revealing information about `a + b` (the branches of the
`if then else` would not require the same number of instructions to execute, not even
mentioning the problem of speculative execution).
The trick is to always subtract 3329, then use the result to conditionally add 3329 back
in case we didn't have to do the subtraction in the first place.
This works because in case the subtraction was not needed (i.e., `a + b < 3329`)
`wrapping_sub` underflows: `res` wraps to a large value
whose upper bits are set, so the right shift yields `0xffff`. Then
`q = 3329 & 0xffff = 3329`, and adding it back wraps around to `sum`.
In the other case (`a + b < 3329`), the shift gives a mask equal to 0, meaning `wrapping_add` adds 0.
In both cases the result is `(a + b) % 3329`, without any branch.

Aeneas translates this to:

```lean
def mod_add (a : Std.U32) (b : Std.U32) : Result Std.U32 := do
  massert (a < 3329#u32)
  massert (b < 3329#u32)
  let sum ← a + b
  let res ← lift (core.num.U32.wrapping_sub sum 3329#u32)
  let mask ← res >>> 16#i32
  let q ← lift (3329#u32 &&& mask)
  ok (core.num.U32.wrapping_add res q)
```

The correctness proof is:

```lean
theorem mod_add.spec (x y : U32) (h : x.val < 3329 ∧ y.val < 3329) :
  mod_add x y ⦃ z => z.val = (x.val + y.val) % 3329 ⦄ := by
  unfold mod_add
  progress*
  bv_tac 32
```

Rather than calling `progress` step by step, `progress*` applies it repeatedly
until no monadic operations remain, leaving a goal containing arithmetic operations
and bit-vector operations. We provide a set of tactics and solvers to discharge
the various proof obligations encountered when verifying such code; in the present
case, it suffices to call `bv_tac`,
a bitvector decision tactic that handles the remaining bit-level reasoning.

One last remark: when developing proofs interactively, one often wants to go fast by means
of, e.g., `progress*`, but also have the possibility of diving into the details of the proof.
For this reason we provide `progress*?` that works like `progress*` but also generates a
detailed proof script that can be inserted in the code via VS Code code actions (one just needs
to click a button). For `mod_add`, we get:

```lean
  let* ⟨ ⟩ ← massert_spec
  let* ⟨ ⟩ ← massert_spec
  let* ⟨ sum, sum_post ⟩ ← U32.add_spec
  let* ⟨ res, res_post ⟩ ← core.num.U32.wrapping_sub.progress_spec
  let* ⟨ mask, mask_post1, mask_post2 ⟩ ← U32.ShiftRight_IScalar_spec
  let* ⟨ q, q_post1, q_post2 ⟩ ← UScalar.and_spec
```

Notation `let* ⟨ x ⟩ ← spec` is syntactic sugar for `progress with spec as ⟨ x ⟩`.
After these steps, only the pure bitvector goal remains.

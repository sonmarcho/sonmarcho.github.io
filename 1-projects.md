---
layout: page
title: "Projects"
---

## Noise\*

My research journey towards program verification at scale started with
[Noise\*](https://github.com/Inria-Prosecco/noise-star), a verified compiler for
the [Noise](https://noiseprotocol.org) family of protocols. Noise defines a
simple, succinct domain-specific language (DSL) to describe key-exchanged (or
handshake) protocols. Handshake protocols are typically used to setup secure
connexions between peers willing to communicate, similarly to TLS. The
description of a protocol in the Noise DSL is called a "pattern".

<p>
<pre style="margin: 0">  IKpsk2:
     <- s
     ...
     -> e, es, s, ss
     <- e, ee, se, psk
</pre>
<div style="text-align: center">
<figcaption>
Description in the Noise DSL of the IKpsk2 protocol (or "pattern"), used by the
<a href="https://www.wireguard.com/">Wireguard VPN</a>.
A very similar protocol, IK, is used by WhatsApp.
</figcaption>
</div>
</p>

Protocol code can be quite low-level, leading to bugs which can compromise their
security (e.g., see [Heartbleed](https://en.wikipedia.org/wiki/Heartbleed)) and
making such code a prime target for program verification.  Verifying one
protocol is however expensive: for instance, miTLS, a verified implementation of
TLS, took 3.5 person-years to verify. At the same time, we need various protocols
for various use cases, leading to obvious scalability issues. This is the reason
why we decided to target a whole family of protocols like Noise, which covers
a wide range of use cases at one.

The result of our work is a compiler which generates fast, specialized C code
for every choice of cipher suite and Noise protocol pattern, and complements
this low-level protocol code with a high-level, specialized, defensive API which
handles state-machines, key lookup and storage, peers, etc. We used the
[Low\*](https://fstarlang.github.io/lowstar/html/LowStar.html) toolchain inside
the [F\*](https://github.com/FStarLang/FStar) theorem prover to prove that the
code is memory safe, functionally correct, and satisfies formal authentication
and confidentiality properties.

We implemented this compiler by using a technique called the Futamura
projection. We wrote and formally verified *one* generic interpreter for the
Noise DSL, that we could later instantiate with a specific pattern. By partially
evaluating this instantiation (i.e., by evaluating subexpressions at compile
time, for instance by unfolding functions, inlining let-bindings, evaluating
matches, etc.) we could turn this instantiation into specialized code. This is a
simple technique which becomes extremely powerful when combined with F\*'s
expressive type system, and more specifically its dependent types. As a
consequence, we were able to generate efficient, idiomatic C code with precise
control-flow, types and function signatures for each of the 59 Noise protocols
(remember, miTLS took 3.5 person-years for a *single* protocol!). If you want to know
more about the application of partial reduction to the verification of
cryptographic code in general, you can read Jonathan's excellent [blog
post](https://jonathan.protzenko.fr/2022/05/22/meta-programming-cryptography.html).

Given that every API instantiation generates between 4k and 6k lines of C code
and that we currently support 8 choices of cipher suites per pattern, not even
mentionning the fact that we can pick various optimized implementations for the
cryptographic primitives, we can generate hundreds of thousands of lines of
verified code for the 59 Noise patterns! For now, we generated 472
instantiatiations which are readily available in the [github
repository](https://github.com/Inria-Prosecco/noise-star) of the project. Those
are of course a limited subset of what we cover: feel free to reach us if you
want a specific instantiation of Noise\*. And if you want to know more about the
specific details, you can have a look at our [S&P'22
paper](https://eprint.iacr.org/2022/607.pdf).

The project hit several difficulties, however, which revealed the limitations of
our toolchain.

First of all, F\* targets *intrinsic* proofs, by which we write code with
annotations such as pre and post-conditions, assertions or lemma calls.  At
type-checking time, F\* generates big weakest preconditions (sets of proof
obligations) which it sends to the Z3 SMT solver to discharge (or fail to
discharge, in which case we need to update the code to fix bugs, or better
explain to Z3 why the code is indeed correct). The issue is that intrinsic
proofs work fine and are very pleasant at first...  until they suddenly become
intractable. More specifically, as the context grows bigger, the proof
obligations become more complex and Z3's response time becomes higher, making
F\* gradually less interactive. This can lead to nightmarish situations when
combined with the fact that the user is blind when carrying their
proofs. Carefully controling the context, for instance by hiding definitions
behind interface files, requires great care and expertise and is not always
possible. Finally, intrinsic proofs are fundamentally non-modular, while
projects require modularity as they grow bigger: for instance, one may want
to verify a single function in several steps.

A second problem we faced was reasoning about memory and aliasing. This was
especially frustrating because memory management is quite simple in Noise\*,
making one expect the memory reasoning to be straightforward. More specifically,
reasoning about the disjointness of various elements in memory makes the context
quickly saturate, especially in the last layers of the API, and acted as a
formidable catalyzer of all the problems we faced with intrinsic
proofs. Overall, the Low\* toolchain is very well suited at verifying
cryptographic primitives and protocols as demonstrated by the
[Hacl\*](https://github.com/hacl-star/hacl-star) project: memory management is
quite simple and in particular memory allocation is almost non-existent, the
control-flow contains few branchings, the SMT solver is of great help when
reasoning about non-linear arithmetic, etc. As a consequence, we had a very
smooth experience when working on the protocol code of Noise\*. In contrast, we
quickly faced an accumulation of problems while moving up the stack through the
various layers of the API, and as a consequence decided to completely rethink
our toolchain, leading us to the second part of my journey.

<a name="Aeneas"></a>
## Aeneas, and his little brother Charon

An appalling issue we faced when reasoning about aliasing in Noise\* is that,
when one looks closer, there is actually barely any aliasing at all in this
code. This made us ask ourselves: isn't there a way of drastically simplifying
the proofs when the memory usage is *disciplined*? And actually, what does a
*disciplined* memory usage mean? Fortunately, Rust and its borrow mechanism
answer this last question in a very interesting manner, by constraining the
programmer in their use of memory to get memory safety while allowing for a very
expressive language. We consequently decided to switch to Rust as our
implementation language, and leverage its type system to simplify the proof
obligations.

This idea of leveraging the Rust type system to simplify memory reasoning is not
new, and has been exploited for several years by other projects like
[Prusti](https://www.pm.inf.ethz.ch/research/prusti.html) or
[Creusot](https://github.com/xldenis/creusot). However, many of those projects
target frontends *à la* Dafny, with intrinsic proofs and weakest preconditions
sent to SMT solvers. As stated above, such proofs work fine in many situations,
but on our side we needed a toolchain targeting extrinsic proofs. We also wanted
to use tactic-based theorem provers, by which one does the proofs incrementally
by interacting with a proof context, while leveraging the possibility of
implementing custom automation with their tactic language. For this reason, we
decided to create a new toolchain.

Our toolchain, [Aeneas](https://github.com/AeneasVerif/aeneas), leverages the
Rust type system to compile Rust programs to pure, executable models (i.e.,
pure, functional versions of the original Rust programs). The key idea behind
Aeneas' compilation is that, under the proper restrictions, a Rust function is
fully characterized by a *forward* function, which computes its return value at
call site, and a set of *backward* functions (one per lifetime), which propagate
changes back into the environment upon ending lifetimes, thus accounting for
side effects. Such forward and backward functions behave similarly to
[lenses](https://www.cis.upenn.edu/~bcpierce/papers/lenses-toplas-final.pdf).
Relying on theorem provers to state and prove lemmas about those models, it is
then possible to enforce guarantees about the original programs. For instance,
one can prove panic freedom and functional correctness (see our [ICFP
paper](https://dl.acm.org/doi/10.1145/3547647), or its [long
version](https://arxiv.org/abs/2206.07185)), but also security guarantees like
authentication and confidentiality as was done with Noise\*, and potentially
more. For now, Aeneas generates pure models for
[F\*](https://www.fstar-lang.org/) and [Coq](https://coq.inria.fr/), and we are
working on additional backends for [Lean](https://leanprover.github.io/) and
[HOL4](https://hol-theorem-prover.org/).

Aeneas relies on [Charon](https://github.com/AeneasVerif/charon) to retrieve
code from the Rust compiler. Charon is a driver which retrieves Rustc's output
(more precisely, the generated MIR)
and translates it to an intermediate language we called LLBC (Low Level Borrow
Calculus - an "easy-to-use" version of MIR in practice). Charon is independent of Aeneas and
should be reusable for other projects: feel free to use it, submit PRs, and
contact us if you have any questions.

## Side Projects

### F\* Extended Mode

The [F\* extended mode](https://github.com/sonmarcho/fstar-extended-mode) is a side
project I did at some point to improve the interaction with F\*, and in
particular make the user less blind when writing proofs by inserting information
about the context directly into their code. The principle is very simple: by
calling dedicated commands, one could introduce the pre and post-conditions of
an (effectful) function call, unfold definitions or split conjunctions in
assertions. I merged the F\* meta code necessary to compute this information to
the F\* main branch, but for some reason I never took the time to merge the
elisp code into the [F\*-mode](https://github.com/FStarLang/fstar-mode.el) repo. I
may find the motivation to do so in the future provided I get enoug traction.


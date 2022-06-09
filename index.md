---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: home
title: ""
---

<img src="profile_picture.jpg"
     alt="Profile picture"
     width="250"
     style="float: left; margin-top: 15px; margin-right: 15px; margin-bottom: 0px" />

# Introduction

I am a Ph.D. candidate working at [INRIA](https://www.inria.fr/fr/centre-inria-de-paris) in
the [Prosecco](https://prosecco.gforge.inria.fr/) team, under the supervision of
[Karthikeyan Barghavan](https://prosecco.gforge.inria.fr/personal/karthik/) and [Jonathan
Protzenko](https://jonathan.protzenko.fr/). My research field is the formal verification
of computer programs, which basically consists in applying mathematical analyses to
programs in order to check that they behave in some well-defined manner, for example that
an embedded system never crashes, or that a cryptographic protocol never reveals secret
data such as private keys or passwords.

# Motivation

My professional experience gave me the conviction that formal verification will become a
necessity in computer security. Any vulnerability left in hardware and software is indeed
of great concern when dealing with the safety of critical systems, the security of
embedded systems, and the protection of our privacy. Considering the ambitious recent
verification projects about compilers, cryptographic protocols, micro-kernels, or hardware
components, I feel this academic field is achieving maturity. However, much work remains
to democratize formal verification, since as of today it is extremely labor-intensive,
mostly relies on experts manipulating advanced tools far remote from what regular
programmers use, and still has many gaps regarding the verification of whole systems. I
thus believe we have yet to create the tools which will be widely adopted tomorrow in the
industry.

# Research Journey (Projects)

## Noise\*

My research journey towards program verification at scale started with
[Noise\*](https://github.com/Inria-Prosecco/noise-star), a verified compiler for the
[Noise](https://noiseprotocol.org) family of protocols. Noise defines a simple, succinct
domain-specific language (DSL) to describe key-exchanged (or handshake) protocols. Handshake
protocols are typically used to setup secure connexions between various peers, in a fashion similar
to TLS. The description of a protocol in the Noise DSL is called a "pattern".

As protocol code is very low-level code, a lot of bugs have been historically found in protocol
implementations such as TLS, which totally compromise their security guarantees. For this reason,
this kind of code is a prime target for program verification. Moreover, protocols can be used in
various use cases, and verifying one protocol is expensive (miTLS, a verified implementation of TLS,
took 3.5 man-years): hence our choice of targetting a whole family of protocols like Noise at once.

The result of our efforts is a compiler which generates fast, specialized C code for every choice
of cipher suite and Noise protocol pattern, and complements this low-level protocol
code with a high-level, specialized, defensive API which handles state-machines, key lookup and
storage, peers, etc. We used the F\* theorem prover to prove that the code is type safe, memory
safe, functionally correct, and satisfies various formal security properties.

The compiler was implemented by using a technique called the Futamura projection: we wrote and
proved correct *one* super generic interpreter for the Noise DSL, instantiated it by applying
it to the various patterns, then partially evaluated those instantiations (i.e., evaluated
subexpressions at compile time, for instance by unfolding functions, inlining let-bindings,
evaluating matches, etc.) to turn them into specialized code. This is a simple technique which
becomes extremely powerful when combined with F\*'s extremely expressive type system, and more
specifically its dependent types. This way, we were able to generate efficient, idiomatic C code
with very precise control-flow, types and function signatures. If you want to know more about the
application of partial reduction to the verification of cryptographic code in general, you can read
this excellent [blog
post](https://jonathan.protzenko.fr/2022/05/22/meta-programming-cryptography.html) written by
Jonathan.

Given that every API instantiation generates between 4k and 6k lines of C code, that there are 59
Noise patterns and that we currently support 8 choices of cipher suites per pattern, not even
mentionning the fact that we can pick various optimized implementations for the cryptographic
primitives, we can generate hundreds of thousands of lines of verified code! For now, we generated
472 instantiatiations which are readily available in the [github
repository](https://github.com/Inria-Prosecco/noise-star) of the project. Those are of course a
limited subset of what we cover, and if you're interested in using a specific choice of
cryptographic primitive implementations for a specific pattern feel free to reach us. And if you
want to know more about how we did all this, you can have a look at the
[paper](https://eprint.iacr.org/2022/607.pdf) we presented at S&P'22.

The project hit several difficulties, however, which revealed the limitations of our toolchain.

First, F\* targets *intrinsic* proofs, by which we write code with annotations such as pre and
post-conditions, assertions or lemma calls, making F\* generate huge weakest preconditions (sets of
proof obligations) which are sent to the Z3 SMT solver to discharge (or fail to discharge, in which
case we need to update the code to fix bugs, or better explain to Z3 why the code is indeed
correct). The issue is that intrinsic proofs work fine and are very pleasant...
until they suddenly become completely intractable. More specifically, as the context grows bigger,
the proof obligations become more complex and Z3's response time becomes higher, making F\*
gradually less interactive. This combined with the fact that the user is blind when carrying their
proofs, at the difference of tactic-based proofs which allow one to inspect an interactive context,
can lead to nightmarish situations. Carefully controling the context, for instance by hiding
definitions behind interface files, requires great care and expertise and is not always
possible. Finally, intrinsic proofs are fundamentally non-modular, while projects require modularity
as they grow bigger.

Second, we faced a lot of issues when reasoning about memory and aliasing. This was especially
frustrating because memory management is quite simple in Noise\*, making one expect the memory
reasoning to be quite straightforward. More specifically, reasoning about the disjointness of
various elements in memory makes the context very quickly saturate, especially in the last layers
of the API, and acted as a formidable catalyzer of all the problems we faced with intrinsic proofs.

All those problems were mostly tractable while we worked on the protocol code of Noise\*,
but quickly worsened as we moved up the stack through the various layers of the API.
As a consequence we decided to completely rethink our toolchain, leading us to the second part of my
journey.

## Aeneas, and his little brother Charon

An appalling issue we faced when reasoning about aliasing in Noise\* is that, when one looks closer,
there is actually barely any aliasing at all in this code. This made us ask ourselves: isn't there a
way of drastically simplifying the proofs when the memory usage is disciplined? And actually, what
does a *disciplined* memory usage mean? Fortunately, Rust and its type system answer this last
question in a very interesting manner, by constraining the user so as not to introduce aliasing,
while allowing for a very expressive language. Consequently, we decided to study the verification of
Rust programs.

This idea of leveraging the Rust type system to simplify reasoning is not new, and has been
exploited by several other projects like [Prusti](https://www.pm.inf.ethz.ch/research/prusti.html)
or [Creusot](https://github.com/xldenis/creusot). However, many of those projects target frontends
*à la* Dafny, with intrinsic proofs and weakest preconditions sent to SMT solvers.  On our side, our
toolchain specification was to have extrinsic proofs, which could be performed with tactic-based
theorem provers so as to have an escape hatch when automation doesn't work. For this reason, we
decided to create a new toolchain, which would work by translating safe Rust programs to pure lambda
calculus, thus removing memory reasoning altogether, then extracted to the backend of
our choice (F\*, Coq, HOL4, etc.).  This in spirit is similar to
[Electrolysis](https://github.com/Kha/electrolysis), an older, very impressive project which
translated Rust to Lean, but suffered from limitations in the subset it targeted. On our side,
our research effort led to the development of [Charon](https://github.com/Kachoc/charon) and
Aeneas. Charon is a compiler pluging for the Rust compiler, which translates Rust programs (starting
from the MIR) to an intermediate language we called LLBC (Low Level Borrow Calculus - a "cleaned-up"
MIR in effect) and is already open-source. Aeneas is a compiler written in OCaml which then
translates LLBC to a pure lambda-calculus. We will open-source it once we have a preprint (coming
soon!).

If you're interested in my work, and in Rust verification in general, feel free to **drop me a mail**!

# Side Projects

## F\* Extended Mode

The [F\* extended mode](https://github.com/Kachoc/fstar-extended-mode) is a side project I did at
some point to improve the interaction with F\*, and in particular make the user less blind when
writing proofs by inserting information about the context directly into their code. The principle is
very simple: by using the proper commands, one could introduce the pre and post-conditions of an
(effectful) function call, unfold definitions or split conjunctions in assertions. I merged the
F\* meta code necessary to compute this information to the F\* main branch, but for some reason I
never merged the elisp code to the [F\*-mode](https://github.com/FStarLang/fstar-mode.el) repo. I
may find the motivation to do so in the future provided I get enoug traction.


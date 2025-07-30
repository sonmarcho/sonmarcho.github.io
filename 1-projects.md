---
layout: page
title: "Projects"
---

## Aeneas

Verify Rust programs in Lean while benefitting from a pure, functional model which
abstracts away the memory! We are currently using Aeneas in the process of porting
Microsoft's open source cryptographic library, used in particular in Windows and Azure
Linux, from trusted C code to verified Rust code - see Microsoft's [blog
post](https://www.microsoft.com/en-us/research/blog/rewriting-symcrypt-in-rust-to-modernize-microsofts-cryptographic-library/).
For more information about this project, including ongoing work
and research topics for potential collaborations and internships, you can go
[here](/assets/documents/aeneas.html) or on the [official website](https://aeneasverif.github.io).

## Charon

Building tools which use the Rust compiler's output, for instance to perform static
analyses, is extremely tedious. If you do not want to waste time in boring engineering effort
you can use
[Charon](https://github.com/AeneasVerif/charon). Essentially, by calling `charon cargo`
from within your crate, you get a file containing your serialized crate, in a high-level,
easy to use AST. We also provide libraries in Rust and OCaml to deserialize this file,
pretty print its components, etc. Charon is being used in several projects, including but
not limited to Aeneas and the [Eurydice
compiler](https://github.com/AeneasVerif/eurydice).
The [tool-paper](https://arxiv.org/abs/2410.18042) provides a high-level overview of
Charon's features.

## Past Projects

### Dafny in Dafny

A partial formalization of the Dafny compiler within Dafny itself ([github
repo](https://github.com/dafny-lang/compiler-bootstrap) and [workshop paper](https://arxiv.org/abs/2401.16233)).

### Noise\*

A verified protocol compiler: give it the description of a protocol as input, and it
produces a specialized, efficient implementation, shipped with a
high-level API inspired by [Wireguard](https://www.wireguard.com), and backed by
functional correctness and security proofs in F\*.
Every call to the compiler generates between 4k and 6k lines of low-level C code,
and the compiler itself is implemented by using what is called a
[Futamura projection](https://en.wikipedia.org/wiki/Partial_evaluation#Futamura_projections) -
a technique I would never have imagined to use in practice, and which is extremely powerful
in combination with dependent types.
More information [here](/assets/documents/noise-star.html) or in the [S&P paper](https://eprint.iacr.org/2022/607.pdf).


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

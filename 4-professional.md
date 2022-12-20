---
layout: page
title: "Past Experience"
---

## Amazon Web Service --- Summer 2022

During this research internship at Amazon Web Service in Seattle, I worked on
formalizing the semantics of the [Dafny](https://dafny.org/) verification
language, and verifying compiler transformations applied on Dafny programs.
This work was carried in Dafny itself. The end goal is to bootstrap the Dafny
compiler, in order to have a verified compilation pipeline from Dafny to its
various target languages (Java, C#, Go, etc.). The result of this work is in the
following [repository](https://github.com/dafny-lang/compiler-bootstrap), and
there is a blog post coming.

## Microsoft Research --- Summer 2021

During this research internship at Microsoft Research, I explored the
verification of Rust programs. This eventually led to my current project,
[Aeneas](https://github.com/AeneasVerif/aeneas), a verification framework which
generates pure models from safe Rust programs, for the purpose of verifying
those programs.  You can learn more [here](2-projects.md#Aeneas), or read our
paper ([ICFP](https://dl.acm.org/doi/10.1145/3547647), [long
version](https://arxiv.org/abs/2206.07185)).

## Carrefour China - 家乐福中国 --- 2018-2019

I worked almost a year in Shanghai for Carrefour China, the
Chinese branch of the [French retailer](https://www.carrefour.com),
on the 
development of data offers and data partnerships.
I supervised the design and implementation of Business
Intelligence products by the IT team, recruited data-analysts, negotiated with
suppliers and partners, and handled contract drafting and signature.


## Prove & Run			---		2017-2018

I worked for 9 months at [Prove & Run](https://www.provenrun.com/) on ProvenTools ®, an
environment to develop mathematically verified programs in an industrial setting.
My task was to design and implement a mechanism in Java to embed the
programs written and verified with ProvenTools ® into the logic of the [Coq proof
assistant](https://coq.inria.fr/).

The goal of the project was to improve the confidence one can have in the programs
verified with ProvenTools ®, by
giving the possibility to use a widely-known and trusted proof assistant to independently
recheck the properties proven about those programs.

## CakeML, CSIRO's Data61			---			2017

During my master internship, I worked on the [CakeML](https://cakeml.org/) project, which aims at developing a
framework to write SML programs verified down to the compiled executables by using the
HOL4 theorem prover. I worked on a mechanism which automatically synthesizes stateful ML
code from functions written in the logic of the HOL4 theorem prover, while generating a proof
that the synthesized ML code correctly implements the functions defined in HOL4. Our
results were published at [IJCAR2018](https://cakeml.org/ijcar18.pdf), and the
slides are available [here](https://easychair.org/smart-slide/slide/vkLp#).

## IDEMIA			---			2016

During my second year at *École polytechnique*, I did an internship at [IDEMA](https://www.idemia.com),
one of the world leaders in biometrics, to develop semi-automated image labelling tools in
Python.

The goal of such tools is to speed-up the work of classifying images (say: giving the
labels "cat", "dog", "horse", etc. to images of animals) in order to generate training datasets
for machine-learning algorithms, by learning the classification while the user is
working on it and using this knowledge to assist him in doing so.

In practice, by using the tools I implemented, one could label tens of thousands of images
per hour without much effort.

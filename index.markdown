---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: home
title: Homepage
---

# Introduction

I am a PhD student working at [INRIA](https://www.inria.fr/fr/centre-inria-de-paris) in
the [Prosecco](https://prosecco.gforge.inria.fr/) team, under the supervision of
[Karthikeyan Barghavan](https://prosecco.gforge.inria.fr/personal/karthik/) and [Jonathan
Protzenko](https://jonathan.protzenko.fr/). My research field is the formal verification
of computer programs, which basically consists in applying mathematical analyses to
programs in order to check that they behave in some well-defined manner, for example that
an embedded system never crashes or that a cryptographic protocol never reveals secret
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

# Past Experience

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
results were published at [IJCAR2018](https://cakeml.org/ijcar18.pdf).

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

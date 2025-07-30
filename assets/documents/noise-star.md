---
layout: default
exclude: true
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
specific details, you can have a look at our
[S&P'22 paper](https://eprint.iacr.org/2022/607.pdf).

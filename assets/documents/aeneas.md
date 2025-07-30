---
layout: default
---

# Aeneas

## Context

One of the main challenges of program verification today consists in finding a way of
efficiently reasoning about low-level, effectful programs. This problem has led to the
development of a large collection of frameworks and logics
[^low_star][^steel][^iris2017][^refinedC][^VST2011][^bedrock][^dafny] which
make various compromises between ease of verification and expressiveness. Despite the
intense research effort in this area, verifying such programs still represents a daunting
task [^haclstar][^noisestar][^VeriBetrKV][^IronFleet].

Following the development of the Rust programming language[^rust], the
recent years have seen the emergence of a new line of research
[^prusti][^rusthorn][^creusot][^flux][^verus][^aeneas] which
attempts to leverage the Rust type system to improve scalability when reasoning about
low-level programs. Rust is indeed a perfect target for program verification, as its
static ownership discipline strikes a good balance between a fine-grained, expressive
control of memory, making it a language of choice for system programming, and strong
memory guarantees, which can be leveraged to ease program verification.

A key insight which drove the design of several frameworks[^rusthorn][^creusot][^verus] is
that a large subset of Rust doesn’t require to explicitly reason about memory, which
typically involves tedious, low-level proofs. Instead, proof engineers can focus on the
functional behavior of their programs. This is what led to the development of the
Aeneas[^aeneas] toolchain, which works essentially as a compiler: given a Rust program,
Aeneas compiles it to a *pure, functional* model that gets extracted to an interactive
theorem prover. By stating and proving theorems about this model, proof engineers can then
prove functional correctness properties about the original code. Aeneas currently has
backends for F\*, Coq and Lean, the latter being our main backend today.

[^low_star]: Jonathan Protzenko, Jean-Karim Zinzindohoué, Aseem Rastogi, Tahina
    Ramananandro, Peng Wang, Santiago Zanella-Béguelin, Antoine Delignat-Lavaud, Cătălin
    Hriţcu, Karthikeyan Bhargavan, Cédric Fournet, and Nikhil Swamy. 2017. Verified low-level
    programming embedded in F\*. Proc. ACM Program. Lang. 1, ICFP, Article 17 (September 2017),
    29 pages. https://doi.org/10.1145/3110261

[^steel]: Aymeric Fromherz, Aseem Rastogi, Nikhil Swamy, Sydney Gibson, Guido Martínez,
    Denis Merigoux, and Tahina Ramananandro. 2021. Steel: proof-oriented programming in a
    dependently typed concurrent separation logic. Proc. ACM Program. Lang. 5, ICFP,
    Article 85 (August 2021), 30 pages. https://doi.org/10.1145/3473590

[^iris2017]: Robbert Krebbers, Ralf Jung, Aleš Bizjak, Jacques-Henri Jourdan, Derek
    Dreyer, and Lars Birkedal. 2017. The Essence of Higher-Order Concurrent Separation
    Logic. In Programming Languages and Systems: 26th European Symposium on Programming,
    ESOP 2017, Held as Part of the European Joint Conferences on Theory and Practice of
    Software, ETAPS 2017, Uppsala, Sweden, April 22–29, 2017,
    Proceedings. Springer-Verlag, Berlin, Heidelberg,
    696–723. https://doi.org/10.1007/978-3-662-54434-1_26

[^refinedC]: Michael Sammler, Rodolphe Lepigre, Robbert Krebbers, Kayvan Memarian, Derek
    Dreyer, and Deepak Garg. 2021. RefinedC: automating the foundational verification of C
    code with refined ownership types. In Proceedings of the 42nd ACM SIGPLAN
    International Conference on Programming Language Design and Implementation (PLDI
    2021). Association for Computing Machinery, New York, NY, USA,
    158–174. https://doi.org/10.1145/3453483.3454036

[^VST2011]: Andrew W. Appel. 2011. Verified software toolchain. In Proceedings of the 20th
    European conference on Programming languages and systems: part of the joint European
    conferences on theory and practice of software (ESOP'11/ETAPS'11). Springer-Verlag,
    Berlin, Heidelberg, 1–17.

[^bedrock]: Adam Chlipala. 2013. The bedrock structured programming system: combining
    generative metaprogramming and hoare logic in an extensible program verifier. In
    Proceedings of the 18th ACM SIGPLAN international conference on Functional programming
    (ICFP '13). Association for Computing Machinery, New York, NY, USA,
    391–402. https://doi.org/10.1145/2500365.2500592

[^dafny]: https://dafny.org

[^haclstar]: Jean-Karim Zinzindohoué, Karthikeyan Bhargavan, Jonathan Protzenko, and
    Benjamin Beurdouche. 2017. HACL*: A Verified Modern Cryptographic Library. In
    Proceedings of the 2017 ACM SIGSAC Conference on Computer and Communications Security
    (CCS '17). Association for Computing Machinery, New York, NY, USA,
    1789–1806. https://doi.org/10.1145/3133956.3134043

[^noisestar]: Ho, Son & Protzenko, Jonathan & Bichhawat, Abhishek & Bhargavan,
    Karthikeyan. (2022). Noise*: A Library of Verified High-Performance Secure Channel
    Protocol Implementations. 107-124. 10.1109/SP46214.2022.9833621.

[^VeriBetrKV]: Travis Hance, Andrea Lattuada, Chris Hawblitzel, Jon Howell, Rob Johnson,
    and Bryan Parno. 2020. Storage systems are distributed systems (so verify them that
    way!). In Proceedings of the 14th USENIX Conference on Operating Systems Design and
    Implementation (OSDI'20). USENIX Association, USA, Article 6, 99–115.

[^IronFleet]: Chris Hawblitzel, Jon Howell, Manos Kapritsos, Jacob R. Lorch, Bryan Parno,
    Michael L. Roberts, Srinath Setty, and Brian Zill. 2015. IronFleet: proving practical
    distributed systems correct. In Proceedings of the 25th Symposium on Operating Systems
    Principles (SOSP '15). Association for Computing Machinery, New York, NY, USA,
    1–17. https://doi.org/10.1145/2815400.2815428

[^rust]: https://www.rust-lang.org

[^prusti]: Vytautas Astrauskas, Peter Müller, Federico Poli, and Alexander
    J. Summers. 2019. Leveraging rust types for modular specification and
    verification. Proc. ACM Program. Lang. 3, OOPSLA, Article 147 (October 2019), 30
    pages. https://doi.org/10.1145/3360573

[^creusot]: Xavier Denis, Jacques-Henri Jourdan, and Claude Marché. 2022. Creusot:
    A&nbsp;Foundry for&nbsp;the&nbsp;Deductive Verification of&nbsp;Rust Programs. In
    Formal Methods  and Software Engineering: 23rd International Conference on Formal
    Engineering Methods, ICFEM 2022, Madrid, Spain, October 24–27, 2022,
    Proceedings. Springer-Verlag, Berlin, Heidelberg,
    90–105. https://doi.org/10.1007/978-3-031-17244-1_6

[^flux]: Nico Lehmann, Adam T. Geller, Niki Vazou, and Ranjit Jhala. 2023. Flux: Liquid
    Types for Rust. Proc. ACM Program. Lang. 7, PLDI, Article 169 (June 2023), 25
    pages. https://doi.org/10.1145/3591283

[^verus]: Andrea Lattuada, Travis Hance, Chanhee Cho, Matthias Brun, Isitha Subasinghe, Yi
    Zhou, Jon Howell, Bryan Parno, and Chris Hawblitzel. 2023. Verus: Verifying Rust
    Programs using Linear Ghost Types. Proc. ACM Program. Lang. 7, OOPSLA1, Article 85
    (April 2023), 30 pages. https://doi.org/10.1145/3586037

[^aeneas]: Son Ho and Jonathan Protzenko. 2022. Aeneas: Rust verification by functional
    translation. Proc. ACM Program. Lang. 6, ICFP, Article 116 (August 2022), 31
    pages. https://doi.org/10.1145/3547647

[^rusthorn]: Yusuke Matsushita, Takeshi Tsukada, and Naoki Kobayashi. 2021. RustHorn:
    CHC-based Verification for Rust Programs. ACM Trans. Program. Lang. Syst. 43, 4,
    Article 15 (December 2021), 54 pages. https://doi.org/10.1145/3462205

## Research Topics

### Use Cases
We are interested in using Aeneas to specify and verify various large use cases, which
include but are not limited to cryptographic applications [^haclstar][^noisestar] or
storage systems[^VeriBetrKV]. Making this verification amenable would imply extending the
subset supported by Aeneas as well as improving its automation.

### Extending the Translation

The translation is a work in progress: we currently support a quite large subset of Rust,
but of course not the whole of Rust. A fundamental limitation is that the pure translation
can only handle a subset of safe Rust: it can't be applied to unsafe code, as for instance
manipulating raw pointers requires modeling memory, and it can also not support (safe)
concurrent code, interior mutability or I/O. We are currently working on connecting Aeneas
to a separation logic framework, so that we can benefit from the pure translation wherever
possible, but can also resort to stateful reasoning wherever necessary. As a simple
example, think about a
[`Mutex`](https://doc.rust-lang.org/std/sync/struct.Mutex.html):
both the `lock` and `unlock` functions are stateful[^mutex_lock], and it is possible to
use separation logic about them. However, what happens between `lock` and `unlock` likely
fits within the subset supported by Aeneas' pure translation. This means that one would
only have to pay the price of using separation logic at a limited number of places, and
would benefit from Aeneas' pure translation speedup elsewhere.

There is still a lot of remaining work to do on this topic:
- connecting the pure translation with a separation logic framework is hard: it both
  requires a bit of thinking to make it work in theory, and a fair amount of engineering
  to make the framework usable in practice.
- a key challenge is to allow having local reasoning by means of separation logic, so the
  use of separation logic is not contaminating. For instance, one might want to formally
  verify a low-level implementation of `Vec` by means of separation logic, but use a
  high-level, pure view of vectors elsewhere (for instance, use lists in the translation).
- proving the soundness of the connection is also an open research problem
- we might want to have a hierarchy of effects, to reason about functions which do not
  modify the state, are total, partial, etc. It is unclear how to make the framework
  usable in that context: we would need a way of automatically lifting theorems from one
  effect to the other, for instance.
- of course, we need to apply this framework to use cases to see whether it really works!

[^mutex_lock]: actually, there is no `unlock` function: unlocking is taken care of by `drop`.

### Soundness

The translation works by doing some sort of a symbolic execution of the Rust programs,
and this symbolic execution actually borrow-checks the program at the same time, which
means we do not need to trust the Rust borrow-checker. In some situations, we can even
borrow-check programs that are rejected by the current Rust borrow-checker, because it is
not precise enough.
We have a pen and paper proof that (a subset of) the symbolic execution acts as a
borrow-checker - essentially, if one successfully symbolic evaluates a Rust program, then
this program is memory safe. We are currently working on mechanizing this pen and paper
proof. Some future work includes extending this proof (the pen and paper proof as a first step, then
the mechanized proof) to prove that the *translation* itself is correct.
Finally, on the longer term, we would be interested in having a proof generation
mechanism: Aeneas would generate a deep-embedding of a program, its functional
translation, together with a proof that the deep-embedding refines the functional
translation.

### Lean Backend: Proof Experience and Automation

A lot of work is going into the Lean backend to make the proof experience pleasant and
scalable. We are always interested in making the backend more user-friendly and more
automated.

## Notes & References

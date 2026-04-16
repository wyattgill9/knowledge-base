These are great questions that touch on some deep foundations of computer science. Let me walk through them.

**What "calculus" means here**

A "calculus" in this context isn't about derivatives and integrals (that's differential/integral calculus). It's the older, broader meaning: a _formal system of symbolic manipulation_ — a set of rules for rewriting expressions. Any time you have a well-defined language of terms plus rules for transforming them, you have a calculus. Lambda calculus is a calculus of _functions_: its terms are variables, abstractions (λx. body), and applications (f x), and its rules describe how to simplify expressions (most importantly, β-reduction: substituting an argument into a function body).

**Lambda calculus and type theory**

Lambda calculus and type theory are deeply intertwined through the **Curry-Howard correspondence** (also called "propositions as types"). The key insight is that typed lambda calculus terms correspond exactly to proofs in a logical system, and types correspond to propositions. For example, a function of type `A → B` is simultaneously a program that takes an A and returns a B, _and_ a proof that "A implies B." This isn't a loose analogy — it's a precise formal isomorphism.

Alonzo Church originally developed the lambda calculus in the 1930s as a foundation for logic, and it turned out to be equivalent in power to Turing machines for defining computability. When people later added type systems to tame it (simply typed lambda calculus, System F, the Calculus of Constructions, etc.), they discovered this correspondence with logic, which is why type theory and lambda calculus grew up together. Proof assistants like Coq and Lean are essentially elaborate typed lambda calculi where writing a program _is_ writing a proof.

**Why languages define their own "calculus"**

Programming languages often define a **core calculus** (sometimes called a "core language" or "formal core") as a minimal, mathematically tractable subset that captures the essential features of the language. They do this because:

- You can _prove properties_ about a small calculus (type safety, termination, confluence) much more easily than about a sprawling real language.
- The full language can then be defined as **syntactic sugar** that desugars into the core. If the core is sound, the full language inherits that soundness.
- It gives language designers a precise way to communicate semantics without ambiguity.

For instance, Haskell's core language is called **Core** (based on System Fω), and GHC compiles Haskell down to it. Rust's type system draws on affine/linear type theory. ML-family languages are rooted in the Hindley-Milner type system, which is a particular typed lambda calculus.

**Derivatives (variants) of lambda calculus and what they're for**

There's a whole zoo of them, each extending the basic lambda calculus to study different phenomena:

- **Untyped lambda calculus** — the original. Models pure computation. Equivalent to Turing machines. Everything is a function, no restrictions.
    
- **Simply typed lambda calculus (STLC)** — adds basic types (like `A → B`). Guarantees termination (all programs halt), but is too weak for general-purpose programming. Corresponds to propositional logic.
    
- **System F (polymorphic lambda calculus)** — adds type variables and universal quantification (∀α. ...), enabling parametric polymorphism (generics). This is the theoretical backbone of ML and Haskell's type systems. Corresponds to second-order logic.
    
- **System Fω** — extends System F with _type operators_ (functions at the type level). Haskell's Core is based on this. Lets you express things like higher-kinded types.
    
- **Calculus of Constructions (CoC)** — the top of the "lambda cube." Unifies types and terms into a single hierarchy, allowing dependent types (types that depend on values). The basis for Coq.
    
- **Lambda cube** — Henk Barendregt's framework organizing eight typed lambda calculi along three axes: polymorphism, type operators, and dependent types. The CoC sits at the corner where all three are enabled.
    
- **Pi-calculus** — not a direct extension but a related calculus for modeling _concurrent_ and _communicating_ processes rather than pure functions.
    
- **Linear lambda calculus** — based on linear logic, where every variable must be used exactly once. This is the theoretical foundation for Rust's ownership/borrowing system and for resource-aware programming.
    
- **Call-by-push-value (CBPV)** — a calculus that cleanly separates values from computations, subsuming both call-by-value and call-by-name evaluation strategies.
    

The pattern is: each variant adds or restricts features to model some specific aspect of programming or logic — polymorphism, concurrency, resource management, dependent types, effects — while keeping the system small enough to reason about formally. Language designers pick the variant (or combination) that matches what they want their type system to guarantee.
Vision
======

The language extensions proposed in these RFCs are meant to develop the Ada
programming language and to extends its adoptabilty and fitness for the kind
of use case where it is the most relevant, that is safety- security-critical
and embedded applications. Evolutions should keep in mind the following axis:

- Improvement of programming paradigms, and addition to potentially missing
  ones. In this category, we'll include thing such as genericity improvements,
  OOP overhall, lambda and coroutines, etc.

- Improvements specific to embedded and low level development, whcih may include
  additional capabilities for record layout, lowering the need for dynamic
  memory, reducing space and time footprint, etc.

- Improvemements to safety and security capabilities, anything that can render
  code "correct by design". For example, borrow-checking is high on the list
  here.

- Improvements to formal proof, constructions that help the development and
  expressiveness of contracts for example

- Improvements to quality of life. Simplifications and additional features
  to the language that do not hinders safety but makes the task of writing
  code easier. A good example of that is the ability to declare variable within
  sequences of statements.

Feature may hit one or several of the above axis - but as much as possible,
they should not contradict one. For example, a feature should not be proposed
if it's not usable in an embedded context, of if it renders the proof
impossible. While there may be rare exceptions to this rule, it should be used
as a strong guideline.

Retro-Compatibilty
==================

Retro-compatibility between these extensions and the current version of the
language are desireable, but not necessary. Ultimately, what will be produced
is two flavors of these extensions:

- A compatible one, which includes only features that can be added to an Ada
  code base with no changes in semantics or additional restrictions.
- A pedantic one, which will not only include features that may modify semantics
  of previously written Ada code, but also adds additional restrictions.

Non-retro compatible changes must be easy to flag by the compiler for analysis.
The compatible version should be as comprehensive and large as possible.

It should also be possible to mix units written in different versions of the
language. This may involve additional semantic clarification and features

Core Feature
============

Object Orientation
------------------

Ada execution model of object orientation is very close to those of languages
of the same generation, such as C++ and Java. However, there are some aspects
that are confusing (absence of structural relationship between primitives
and types, non-dispatching behavior on non-class-wide views) as well as some
missing capabilities (constructors, destructors, "protected" regions...).

This aspect of the roadmaps aims at completing and adjusting OOP concepts to
match closer to those of traditional OOP languages. It will include in
particular:

- Addition of constructors
- Addition of destructions
- Creation of a structural syntax encompassing components and primitives in the
  same type scope.
- Modification of dispatching rules, making dispatching by default
- Finer grain visibility, allowing both public and private components, and
  introduction of a section visible to children but not users (called protected
  in other languages)

Genericity
----------

Genericity in Ada is very powerful, but the full capacity of the feature is
sometimes difficult to gasp because of the heaviness of its usage. The objective
of the enhancement will cover several aspects to increase it useability:

- Structural instanciation, allowing implicit instanciation and type
  compatibilty at least for stateless generic units.
- Inference between generic formal parameters. We can leveregage Ada strong
  typing to automatically detect actual parameters from other actual parameters
  (e.g. in Unchecked_Deallocation, you can infer safely the object type from
  the pointer type)
- In the context of calls of generic subprograms, inference between the types
  of the actual parameters and the generic formal types.
- Improvements in generic parameter declaration. Right now some things can
  be expressed (e.g. this formal is discrete) and some can't (e.g. this formal
  is a number). We could consider revamping that into a more general and
  explicit model, possibly backing-up the introduction of traits.

Closure and Related Capabilities
--------------------------------

Closure is the ability to automatically "detach" a stack of data from a local
stack. It can be used to turn a procedure into an object, such as a lambda,
a generator or a co-routine. In itslef, it may be viewed as a capability
reserved for dynamic or functional languages. However, the abilty to write
generators and co-routines enables single thread concurrency, which can be
extremely interesting on small run-time or deterministic systems. When they
fit the underlying software architecture, they're also more performant as
they don't require context commutation (the confusingly called stack-free
model).

One aspect of closures is that they can capture their environment, either by
copy or by reference. By reference capture can only be done with ownership
semantics, this would need to wait for the borrow checking design (see later).

To some respect, co-routines could be viewed as Ada tasks that don't pre-empt,
scheduling and activation needs to be triggered by the sequential program flow.
From a language design perpective, a non-preemptive task model might be close
to enough to implement the concept. It is however a significant effort from
an implementation standpoint.

Beyond the bare efficiency aspect, this may produce a computational model
that lends itself better to formal proof, and complete the set of paradigms
available to Ada.

Generators and lambda are probably also useful in their own right - although
they may lean themeselves more to the "quality of life" category.

Memory Safety and Borrow-Checking
---------------------------------

This should be considered in the context of an overhall of pointers semantics,
that has grown organically complex with Ada. This complexity arose from the
objective of getting safer dynanic memory support, with argubably mitigated
success.

The borrow checking mechanism popularized by Rust could and should be an
effective and chartered solution for this. In the context of Ada, we would
possibly implement a simpler model (ie more constraining) and defer to SPARK
and formal proof demonstration of more complex cases where we could relax
constraints (similar to initialization).

Another source of inspiration is C++ move semantics, which can be applied
even to objects that don't require absence of aliasing but instead describes
cases where copy are unecessary, and reference passing is enough to ensure
safety. This is potentially of a broader applicability than bare borrow
checking applied to access types, as other types of object may benefit from
what is now an optimization. However, there are links between the two
capabilities that need to be explored.

Smart pointers should come naturally out of this, with the correspoding move
and borrow-checking semantics. This could either be a language feature or
a library.

Last, the concept of accessibilty needs to be reviewed. It may be less
necessary to describe dynamic memory when borrow-checking is present, but
there is still potential need to describe what other languages refer to as
lifetimes. We could consider making that concept explicit in the language.

Other Topics of Interest and Quality of Life
============================================

Besides the 4 main chapters decribed above, a number of smaller aspects of the
language improvement can be considered. To date, the following list has been
developped:

- Anonymous records / tuples
- Traits
- Overlaid record fields
- Explicit static expressions
- Standard library improvements

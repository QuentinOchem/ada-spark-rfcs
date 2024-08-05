- Feature Name: flare
- Start Date: 2024-08-01
- RFC PR: (leave this empty)
- RFC Issue: (leave this empty)

Summary
=======

This RFC introduces a name for the new Ada extensions: Flare (or Ada Flare).
It proposes a pragma to enable the Flare mode of the language, which alters
the semantics of various Ada constructions and enables a number of
pragmas.

Motivation
==========

Guide-level explanation
=======================

A new pragma is introduced, `Flare (<version>)`, which can be used at the
library level. `version` is a number representing the version number of the
language, for example 2025 or 25 for the initial version:

```Ada
pragma Flare (25);

package My_Package is

end My_Package;
```

When present, the whole unit (specification and implementation) is compiled
following the flare 25 specification of the Ada language.

The 25 version of the language enables in particular the following changes:

- All tagged types are by-constructor types (TODO REF)
- All calls to controlling primitives are dispatching by default (TODO REF)
- All record types are scoped (TODO REF)
- Finalization is done following the light model (TODO REF)
- <numberic number>'Image doesn't add a space in front of positive numerical values
- `and` and `or` are always shortcirtuit when dealing with boolean, shortcut to
  `and then` and `or else` which are still accepted as obsolete features.

It also make legal the following features:

- [Fixed lower bounds](https://github.com/AdaCore/ada-spark-rfcs/blob/master/prototyped/rfc-fixed-lower-bound.rst)
- [Local vars without blocks](https://github.com/AdaCore/ada-spark-rfcs/blob/master/prototyped/rfc-local-vars-without-block.md)
- [Prefix untagged](https://github.com/AdaCore/ada-spark-rfcs/blob/master/prototyped/rfc-prefixed-untagged.rst)
- [String interpolation](https://github.com/AdaCore/ada-spark-rfcs/blob/master/prototyped/rfc-string-interpolation.md)
- [Size'Class](https://github.com/AdaCore/ada-spark-rfcs/blob/master/considered/rfc-class-size.md)
- [Constrained indefinite arrays](https://github.com/AdaCore/ada-spark-rfcs/pull/97)
- [continue](https://github.com/AdaCore/ada-spark-rfcs/blob/master/considered/rfc-continue.md)
- [Embedded binary resources](https://github.com/AdaCore/ada-spark-rfcs/blob/master/considered/rfc-embed-binary-resources.rst)
- [Finally](https://github.com/AdaCore/ada-spark-rfcs/blob/master/considered/rfc-finally.md)
- [Generalized finalization](https://github.com/AdaCore/ada-spark-rfcs/blob/master/considered/rfc-generalized-finalization.md)
- [Array slice access](https://github.com/AdaCore/ada-spark-rfcs/pull/102)
- [Deep delta aggregates](https://github.com/AdaCore/ada-spark-rfcs/pull/84)
- [Multiple levels of ghost](TODO REF)
- [Generic - expression default for functions](https://github.com/AdaCore/ada-spark-rfcs/blob/master/prototyped/rfc-expression-functions-as-default-for-generic-formal-function-parameters.rst)
- [Generic - Inference of dependent types](https://github.com/AdaCore/ada-spark-rfcs/blob/master/considered/rfc-inference-of-dependent-types.md)
- [Generic - Constrained attribute for generic objects](https://docs.adacore.com/gnat_rm-docs/html/gnat_rm/gnat_rm/gnat_language_extensions.html#constrained-attribute-for-generic-objects)
- [OOP - Dispatching](https://github.com/AdaCore/ada-spark-rfcs/pull/118)
- [OOP - Constructors](https://github.com/AdaCore/ada-spark-rfcs/pull/115)
- [OOP - Destructors](https://github.com/AdaCore/ada-spark-rfcs/pull/117)
- [OOP - Super](https://github.com/AdaCore/ada-spark-rfcs/pull/116)
- [Static on instrinsic](https://docs.adacore.com/gnat_rm-docs/html/gnat_rm/gnat_rm/gnat_language_extensions.html#static-aspect-on-intrinsic-functions)
- [Conditional when](https://docs.adacore.com/gnat_rm-docs/html/gnat_rm/gnat_rm/gnat_language_extensions.html#conditional-when-constructs)

Note that there are other features currently under development and
experimentation that are not part of the 24 profile. These can be enabled
by ``pragma Flare (0);``. This is meant to be an experimental mode and may
have unstable features.

Reference-level explanation
===========================


Rationale and alternatives
==========================


Drawbacks
=========


Prior art
=========

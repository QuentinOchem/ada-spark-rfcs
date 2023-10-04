- Feature Name: (fill me in with a unique ident, my_awesome_feature)
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- RFC Issue: (leave this empty)

Summary
=======


Motivation
==========

Guide-level explanation
=======================

The evolution of Ada access types ends up making the "generally useful and safe
version" quite verbose. A user would need to write for the majority of its
access types:

```Ada
   type A is limited access all Some_Type;
```

We're introducing a new pragma, Language_Profile, which allows to modify the
default semantics of the Ada syntax:

```Ada
   pragma Language_Profile (Value => [True|False]);
```

This pragma can be applied to a syntactical scope, or the whole application.

In this proposal, we're introducing two values:

- Access_Limited, when enabled, all access types declared in the scope are
  limited. By default, it is False for partitions compiled until Ada_2022, True
  afterwards.
- Access_All, when enabled, all acess types delcared in the scope are general
  access types. By default, it is False for partitions compiled until Ada_2022, True
  afterwards.
- Thread_Safe_Tasking, when enabled, all tasks and protected objects are marked
  Thread safe.

In order to allow access to cancel either default on a specific access
declaration, we provide a `not limited` and `not all` syntax so you can have:

```Ada
   type A is not limited access not all Some_Type;
   -- Equivalent to type A is access Some_Type until Ada 2022

   type A is limited access all Some_Type;
   -- Equivalent to type A is access Some_Type after Ada 2022
```

This allows users of the new version of the Ada language to have memory-safe
access by default, while allowing for backward compatibilty, as non-memory-safe
accesses are still available.

Reference-level explanation
===========================

Rationale and alternatives
==========================

Drawbacks
=========

Prior art
=========

Unresolved questions
====================


Future possibilities
====================

The pragma Language_Profile is meant to allow to control additional backward
incompatibles and simplifications of the language. It could be used for e.g.
control of finalization or short-circuit behavior of boolean operators.

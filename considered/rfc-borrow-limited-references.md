- Feature Name: (fill me in with a unique ident, my_awesome_feature)
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- RFC Issue: (leave this empty)

Summary
=======

This proposal introduces a new notation, `limited`, allowing it to be
associated with a variable or as a parameter mode. The notation designates
values that should be moved rather than copied. It draws inspiration from the
C++ rvalue reference and incorporates some safety aspects from Rust's borrow
checker.

The proposal also delves into the natural extensions of limited types brought
forth by limited objects and parameters.

Motivation
==========

Ada excels in situations requiring high performance and a small footprint,
particularly in embedded environments. C++'s move semantics eliminate
unnecessary copies and reduce the reliance on pointers while maintaining
memory safety. Enhancing Ada's memory safety by flagging or prohibiting known
unsafe patterns will further solidify its standing.

Guide-level explanation
=======================

Limited References
------------------

We're introducing a new kind of object: the limited reference, introduced by the
reserved word `limited` in from of the type indication. A limited reference is a
reference that "owns" its value. Its value cannot be assigned.
When used between limited references, the assignment operation is modified
to a move operation. For example:

```Ada
   type A is access all Integer;

   Src : limited A := new Integer;
   Dst : limited A;
begin
   Dst := Src; -- performs a move
```

While move semantics have to be done through the 'Move attribute for regular
objects, they are implicit for limited references.

The language ensures that a limited reference always refers to the value its
owning. As a consequence, in the example above, the compiler won't issue
the warnings it would issue on a non-limited move operation.

Assignment to a limited reference can be done either through a literal (they
create a new value) or through another limtied reference.

Limited references can be variables, parameters or returned types:

```Ada
   V : limited Some_Type;

   procedure P (V : limited Some_Type);

   function F return limited Some_Type;
```

A value of a non-limited reference can be assigned to a limited reference by an
an explicit 'Clone call, which effectively is creating a new value. 'Move
and 'Borrow are also possible, although they may lead to unsafe situations
that the compiler will warn about (in particular in the context of a move,
as the source value may not be owned by the variable referencing it).

Here are a few examples of initialisation and assignments:

```Ada
   V1 : limited Integer := 1; -- OK
   V2 : limited Integer := 1 + 2; -- OK

   function F return Integer;
   V3 : limited Integer := F; -- OK

   V4 : Integer;
   V5 : limited Integer := V4; -- Compilation Error
   V6 : limited Integer := V4'Clone; -- OK

   V7 : constant Integer := 0;
   V8 : limited Integer := V7; -- Compilation Error

   procedure P (V : Integer) is
      L : limited Integer;
   begin
      L := V; -- Compilation Error
   end P;

   procedure F (V : limited Integer) return limited Integer
      L : limited Integer;
   begin
      L := V;

      return L; -- OK
   end P;

   V8 : limited Integer := 1;
   V9 : limited Integer := V8; --OK
```

Limited parameters can be either `in`, or `in out` For example:

```Ada

   procedure F (V : limited in out Integer) return limited Integer
      L : limited Integer;
   begin
      L := V;
      L := L + 1;

      return L; -- OK
   end P;

```

Overloading and limited References
----------------------------------

The `limited` mode can induce overloading on a subprogram parameter. When
this happens, the compiler invariably prioritizes the profile with the limited
parameter. For example:

```Ada
   V1 : Integer := 1;

   procedure P1 (V : Integer); -- (1)

   procedure P2 (V : limited Integer); -- (2)
   procedure P2 (V : Integer); -- (3)

   P1 (1);  -- calls (1)
   P1 (V1); -- calls (1)

   P2 (1);  -- Calls (2)
   P2 (V1); -- Calls (3)
```

This may lead to situations with multiple parameters where there are several
options. This would lead to a compiler errror.

```Ada
   V1 : Integer := 1;

   procedure P (V1 : limited Integer; V2 : Integer); -- (1)
   procedure P (V1 : Integer; V2 : limited Integer); -- (2)

   P (1, 1); -- Error, can't select between (1) and (2)
```

However, many cases can be resolved when always giving the priority to limited:

```Ada
   V1 : Integer := 1;

   procedure P (V1 : limited Integer; V2 : Integer); -- (1)
   procedure P (V1 : limited Integer; V2 : limited Integer); -- (2)

   P (1, 1); -- Calls (2)
```

This semantic allow to provide both operations in cases where an object can
still be used after a call or not. It is particularly useful in the context
of OOP (see the other part of the proposal refering to move constructors),
but could also used in different places. For example, we can provide two
version of a deallocation primitive, one that reset a value to null if the
object isn't limited and what that doesn't.

Limited Types
-------------

Limited types in Ada traditionally prevent objects from being copied or compared
for equality. A move operation doesn't imply a copy of the object. As a
consequence, move between limited references to limited types are allowed:

```Ada
   type Holder is limited ...

   V1 : limited Holder;
   V2 : limited Holder;
begin
   V2 := V1; -- OK
```

Components
----------

A component cannot be limited in isolation of its enclosing object. The following
is illegal:

```Ada
   type R is record
      F : limited Some_Type;
      --  compiler error
   end record;
```

When a reference is limited, all its components and sub-components are limited
as well:

```Ada
   type R is record
      F : Some_Type;
   end record;

   V : limited R;

   procedure P1 (V : Some_Type);
   procedure P2 (V : limited Some_Type);

begin

   P1 (V.F);
   --  Compilation error, cannot use a limited reference to value a non-limited parameter

   P2 (V.F);
   --  Ok, but V is partially moved, the compiler will consider that the entire
   --  object is.
```

Such call could be done by cloning ('Clone) or replacing in place ('Replace)
the values of the references, for example:

```Ada
   type R is record
      F : Some_Type;
   end record;

   V : limited R;

   procedure P1 (V : Some_Type);
   procedure P2 (V : limited Some_Type);

begin

   P1 (V.F'Replace (Some_Type'(others => <>)));
   --  OK

   P2 (V.F'Clone);
   --  OK
```

Note that memory safety on more complicated structure can be acheived through
limited access types, described in a separate part of the proposal.

Dereferencing Limited Accesses
------------------------------

The value pointed by a limited access is itself a limited reference. As
a consequence:

```Ada
   V : limited access Integer;

   procedure P1 (V : Integer);
   procedure P2 (V : limited Integer);

begin

   P1 (V.all);
   --  Compilation error, cannot use a limited reference to value a non-limited parameter

   P2 (V.F'Clone);
   --  Ok, but V and its targeted object are both moved
```

Limited Accesses and Deallocation
---------------------------------

When a limited access object goes out of scope and hasn't been moved, by
definition it can't be reached anymore. The compiler will therefore insert
a implicit deallocation upon reaching the end of the scope of the object or
another move instruction. For example:

```Ada
   V1 : limited access Integer := new Integer;
   V2 : limited access Integer := new Integer;

   procedure P (V : limited Integer);

begin
   P (V1);
   --  V1 is moved here;
end; -- V2 hasn't been moved, freed
```

Note that if the compiler cannot statically determine that an object hasn't been
moved in all paths, it's sufficient to determine that it may be moved, as its\
value will be null if indeed moved. For example:

```Ada
   V : limited access Integer := new Integer;

   procedure P (V : limited Integer);

begin
   if Some_Condition then
      P (V);
      --  V1 is moved here;
   end if;

   if Some_Other_Condition then
      -- V may have been moved here, deallocate if not null
      V := new Integer;
   end if;

end; -- V may have been moved here, deallocate if not null
```

Reference-level explanation
===========================

See above.

Rationale and alternatives
==========================

There are a few dimensioning decisions made in that proposal that
differ notably from Rust and C++:

- Limited References: The decision to introduce a new kind of reference
  (limited) similar to Rust's &mut and C++ &&. We're introducing a reserved word
  here, instead of a symbol Ada's syntax aims to be explicit and readable,
  even at the cost of being more verbose. This contrasts with other languages
  design decisions that favor conciseness. By having a limited keyword, we hope
  to provide Ada developers a clear and distinct signal of the difference
  between standard Ada references and the new references that introduce move semantics.

- Overloading with limited: The idea to use overloading with the limited
  parameter can be seen as introducing another layer of complexity.
  However, C++ has demonstrated its usefulness, in particular in the context
  of copy constructors and assignment overload.

Drawbacks
=========

See above.

Prior art
=========

This is influenced by C++ move and to a lesser extent Rust borrow checking
mechanism, see above for more details.

Unresolved questions
====================

There's a discernible overlap between this feature and a full-fledged
borrow-checker seen in languages like SPARK and Rust. The scope of this
overlap and its applicability within that context remain uncertain.

The interaction of this feature with tasking also presents an unresolved query.

Future possibilities
====================

While this proposal pinpoints new erroneous behaviors, they aren't gemuinly new
vulnerabilities. They stem from existing programming patterns developed through
less secure methods, such as pointer usage or ad-hoc move operations without
the safeguards provided by the limited notation. Currently, these behaviors
manifest as errors or warnings, depending on the certainty of occurrence.
Leveraging tools like SPARK to further elucidate or validate error
probabilities would be advantageous.


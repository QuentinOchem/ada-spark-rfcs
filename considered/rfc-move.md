- Feature Name: (fill me in with a unique ident, my_awesome_feature)
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- RFC Issue: (leave this empty)

Summary
=======

This proposal introduces a new notation, ``limited``, allowing it to be
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

We're introducing a new kind of object: the limited reference. It offers two
distinct ways of handling values:

- A value can be moved (instead of copied) from one reference to another. After
  the move, using the original reference before another assignment is erroneous,
  since its value may be invalid.
- A value can be borrowed by a parameter. It can be modified within the
  subprogram, but is expected to remain valid (i.e., not moved or if moved,
  assigned a new value) upon the subprogram's completion.

A limited reference is denoted by the mode ``limited``. Here are some examples:

```Ada
   V : limited Some_Type;

   type X is record
      F : limited Some_Type;
   end record;

   procedure P (V : limited Some_Type);
```

Assignments to limited references have constraints. They can only be assigned:

- Other limited references, in which case values are moved from one reference
  to the next as opposed to copied (more on the differences between move and
  copy later).
- A temporary value not referenced by any other object (some languages such as
  C++ call these rvalues).

```Ada
   V1 : limited Integer := 1; -- OK
   V2 : limited Integer := 1 + 2; -- OK

   function F return Integer;
   V3 : limited Integer := F; -- OK

   V4 : Integer;
   V5 : limited Integer := V4; -- Compilation Error

   V6 : constant Integer := 0;
   V7 : limited Integer := V6; -- Compilation Error

   procedure P (V : Integer) is
      L : limited Integer;
   begin
      L := V; -- Compilation Error
   end P2;
```

Limited references can't be copied implicitly. You can't assign them to
non-limited variables or fields. When passed by reference to parameters,
they can either be moved if the parameter is limited or temporarily borrowed
otherwise:

```Ada
   procedure P1 (V : Integer);
   procedure P2 (V : in out Integer);

   X : limited Integer;

begin

   P1 (X); -- Borrowed
   P2 (X); -- Borrowed
```

A limited parameter is a distinct mode, comparable to ``in``, ``in out``, or
``access``.  Such parameters only accept limited references.

```Ada
   procedure P (V : limited Integer);

   X1 : limited Integer;
   X2 : Integer;

begin

   P (X); -- OK
   P (X2); -- Compilation Error
```

Borrow Semantics
----------------

Borrow semantics apply when a limited type is passed to a non-limited
parameter. Here, the source object is temporarily moved to the target and then
move back.

```Ada
   V : limited T;

   procedure P1 (V in T);
   procedure P2 (V in out T);

begin

   P1 (V); -- OK
   P2 (V); -- OK
```

While some languages also facilitate borrow semantics during local variable
assignments, the current proposal doesn't explore this avenue.

Overloading and limited References
----------------------------------

The ``limited`` mode can induce overloading on a subprogram parameter. When
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

Limited Types
-------------

Limited types in Ada traditionally prevent objects from being copied or compared
for equality. This aligns with Ada's emphasis on safety, especially when c
ertain objects represent unique resources.

In line with evolving programming paradigms and efficient resource management,
we propose introducing move semantics for limited types, allowing the transfer
of ownership between variables.

For consistency, this proposal suggests allowing equality checks for limited
types. Instead of default mechanisms, developers can use abstract equality
operators or other tailored means.

While copying remains restricted for limited types, moving should be permitted.
This allows the transfer of an object's state without duplicating the resource.

Consider:

```Ada
   type Holder is limited ...

   V1 : Holder;
   V2 : limited Holder;
   V3 : limited Holder;
begin
   V2 := V3; -- legal, move semantics
   V2 := V1; -- Compilation error
```

Here, V2 inherits from V3 without duplication. But assigning from V1 to V2 is
illegal due to the copy restriction on limited types.

Limited Access References and Types
-----------------------------------

The idea of limited access types introduces further value to move semantics.
The primary distinction here is between a regular access type,
which allows both copying and moving, and a limited access type,
which allows only moving.

This first example demonstrates the usage of a limited reference for a
regular pointer object:

```Ada
   type Holder is  ...
   type A_Holder is access all Holder;

   V1 : limited A_Holder := new Holder;
   V2 : limited A_Holder;
begin
   V2 := V1; -- legal, move semantics. V1 is now null.
   -- V1 is null here, the compiler will detect or warn possible usage
```

The pointer type itself can be declared as limited, which not only prevents
copying but also enforces move semantics on all instances:

```Ada
   type Holder is  ...
   type A_Holder is limited access all Holder;

   V1 : A_Holder := new Holder;
   V2 : A_Holder;
begin
   V2 := V1; -- legal, move semantics.  V1 is now null.
   -- V1 is null here, the compiler will detect or warn possible usage
```

Access types can also target limited objects, thereby adding another layer of
restriction.

```Ada
   type Holder is  ...
   type A_Holder is access limited Holder;

   V1 : A_Holder := new Holder;
   V2 : limited Holder;
   V3 : Holder;
begin
   V2 := V1.all; -- legal, move semantics of the pointed object
   Free (V1);
   V1 := new Holder;
   V3 := V1.all; -- Compiler Error
```

Move Semantics with Object Orientation
--------------------------------------

Move semantics can significantly improve object-oriented design. When combined
with new constructors and assignment primitives from the OOP revamp proposal,
limited references can provide an efficient way to manage objects without
unnecessary copying

Copy and assignment primitives can now be provided with limited parameters.
Consider the following class:

```Ada
   type Holder is class record
      procedure Holder (Self : in out Holder; Source : T1);
      -- Regular by copy constructor

      procedure Holder (Self : in out Holder; Source : limited T1);
      -- Move constructor

      procedure ":=" (Self : in out T1; Source : T1);
      --  Regular assignment

      procedure ":=" (Self : in out T1; Source : limited T1);
      --  By copy assignment

      Size : Integer;
      Contents : Array_Access;
   end Holder;

   type Holder is class record

      ...

      procedure Holder (Self : in out Holder; Source : limited T1) is
      begin
         Self.Size := Source.Size;
         Self.Contents := Source.Content;

         Source.Size := 0;
         Source.Contents := null;
      end Holder;

      procedure ":=" (Self : in out T1; Source : limited T1);
      begin
         Free (Self.Contents);

         Self.Size := Source.Size;
         Self.Contents := Source.Content;

         Source.Size := 0;
         Source.Contents := null;
      end ":=";

  end Holder;
```

We can now write the following without copy:

```Ada

   function Create_Holder (Size : Integer := 0) is
   begin
      return Holder (Size);
   end Create_Holder;

   V : limited Holder;

begin

   V := Create_Holder (5); -- no need for any copy here

```

Summary of Copy, Move and Borrow Rules
--------------------------------------

The table provided offers an exhaustive list of potential interactions and
their results:

| A := B, A => B  | B : T    | B : limited T | B : constant T | (B : in T) | (B : in out T) | (B : limited T) |
| ----------------| -------- | ------------- | -------------- | ---------- | -------------- | --------------- |
| A : T           | copy     | error         | copy           | copy       | copy           | error           |
| A : limited T   | error    | move          | error          | error      | error          | move            |
| A : constant T  | copy     | error         | copy           | copy       | copy           | error           |
| (A : in T)      | copy/ref | borrow        | copy/ref       | copy/ref   | copy/ref       | borrow          |
| (A : in out T)  | copy/ref | borrow        | error          | copy/ref   | copy/ref       | borrow          |
| (A : limited T) | error    | move          | error          | error      | error          | move            |

'Move and 'Copy
------------------------

Two new attributes are introduced, offer explicit control over the operations,
making it easier for developers to understand and command the flow of data and
to override sitations that would otherwise lead to compilation error as
described above:

- `'Copy` can be applied to a limited reference or an object and triggers a
  copy operation. The result can be then moved to a limited reference or
  copied to an object.
- `'Move` can be applied to a limited reference or an object as long as they're
  variable (ie no constant, no in parameter) and triggers a move operation.
  The result can then ne moved to a limited reference or copied to an object.

For example:

```Ada
   function Create_Holder return Holder;

   V1 : Holder (Create_Holder); -- Move constructor
   V2 : Holder := Create_Holder; -- Move assignment
   V3 : limited Holder := Create_Holder;
   V4 : limited Holder;
begin
   V3 := V1'Move; -- OK - this is a move from V1 to V3
   V1 := V3'Copy; -- OK - this is a copy from V3 to V1
```

Erroneous Execution
-------------------

It is erroneous to use an object after it has been moved. The compiler is
supposed to generate an error where such object is definitely use, and a
warning where some path in the control flow may lead to such use. For example:

```Ada
   procedure P (V : limited Holder);
   V  : limited Holder;
   V2 : limited Holder;
   V3 : Holder;
   V4 : Holder;
   V5 : limited Holder;
begin
  P (V);
  V.Add_Value (10); -- compiler error, use after move

  V2 := V3'Move;
  V3.Add_Value (10); -- compiler error, use after move

  if Some_Condition then
     V5 := V4'Move;
  end if;

  V4.Add_Value (10);
  -- warning - may be use after move

  procedure P2 (V : in out Holder) is
  begin
     P (V'Move);
     -- warning - V may be use after move upon return
  end P2;
```

However, moving an object to a limited variable afterwards makes usage valid
again. For example:

```Ada
   V  : limited Holder := Create_Holder (1000);
   V2 : limited Holder;
begin
   V2 := V;
   -- Using V generates a compiler error here
   V := Create_Holder (500);
   -- Using V is ok now
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

- Borrow Semantics: While Rust incorporates borrowing semantics both at the
  variable and parameter levels, our proposal chooses to limit borrowing
  semantics primarily at the parameter level. This decision could be reverted
  in an extension of this proposal if deemed useful.

- Overloading with limited: The idea to use overloading with the limited
  parameter can be seen as introducing another layer of complexity.
  However, C++ has demonstrated its usefulness, in particular in the context
  of copy constructors and assignment overload.

- Limited Types and Limited Access References: The way Ada deals with
  limited types is inherently different from Rust. While Rust defines every
  type as potentially being moved, Ada's traditional types don't have this
  property. Introducing move semantics in a way that remains compatible with
  existing Ada codebases and idioms is crucial, and similiar to C++ choices.
  By allowing for the definition of limited access types and references, we
  provide a structured way for Ada developers to opt into move semantics without
  breaking or complicating their existing code.

- Use After Move: The Ada philosophy centers around safety. As such, ensuring
  that moved objects cannot be accessed is essential for preventing unforeseen
  bugs or undefined behavior. The way our proposal deals with "use after move"
  scenarios, by making them compilation errors or warnings, is in line with
  Ada's goal of catching potential problems at compile-time rather than runtime.
  SPARK could further extent this analysis and discharge unecessary warnings.

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

Additionally, adopting certain coding practices can bolster safety and
consistency:

- By default, pointer objects should be limited to mitigate unnecessary
  aliasing and enhance memory safety.
- Limited types should consistently be accessed via limited references.
- Some warnings, especially those related to "use after move" in parameters
  (due to their non-local impact), should escalate to errors.

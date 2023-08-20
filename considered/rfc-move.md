- Feature Name: (fill me in with a unique ident, my_awesome_feature)
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- RFC Issue: (leave this empty)

Summary
=======

This proposal introduces a new notation ``limited``, denoting a limited
reference to an object also known as ``rvalue`` in languages such as C++.
It can then be used to denote a reference to a so called ``rvalue``, and
implement move semantics similar to C++, as well as to some extent borrow
semantics coming from Rust.

This proposal also addresses natural extensions of limited types made possible
by limited objects and parameters.

Motivation
==========

Ada is a language that has high performances and small footprint requirements
for embedded situations. Move semantics as describe in C++ allows to avoid
unecessary copies and remove certain cases of use of pointers while keeping
memory safety.

On top of this, forbiding or marking erroneous known unsafe pattern will improve
the overall memory safety of standard Ada application.

Guide-level explanation
=======================

Limited References
------------------

We're introducing a new kind of object, a limited reference, associated with
two ways of manipulating values:

- A value can be moved from one reference to another one. In such case, it is
  erroneous to use the previous reference to this object prior to another
  assignment, as its value may be invalid.
- A value can be borrowed by a parameter. It can be modified by the body of
  the subprogram, but is expected to still be valid (ie not moved, or if moved,
  reassigned to a different value) when exiting the subprogram.

A limited reference is denoted by the introduction of the word ``limited`` in
front of the type, in a parameter, a variable or a field name. For example:

```Ada
   V : limited Some_Type;

   type X is record
      F : limited Some_Type;
   end record;

   procedure P (V : limited Some_Type);
```

Limited references can only be assigned from:

- Other limited references, in which case values are moved from one reference
  to the next as opposed to copied (more on the differences between move and
  copy later).
- A temporary value not referenced by any other object

```Ada
   V1 : limited Integer := 1; -- OK
   V2 : limited Integer := 1 + 2; -- OK

   function F return Integer;
   V3 : limited Integer := F; -- OK

   V4 : Integer;
   V5 : limited Integer := V4; -- Compilation Error

   V6 : constant Integer := 0;
   V7 : limited Integer := V6; -- OK

   procedure P (V : Integer) is
      L : limited Integer;
   begin
      L := V; -- error
   end P2;
```

Limited references cannot be copied implicitely. They can't be assigned to
non-limited variables or fields. They can be passed by reference to parameters
in ``in`` mode.

```Ada
   procedure P1 (V : Integer);
   procedure P2 (V : in out Integer);

   X : limited Integer;

begin

   P1 (X); -- OK
   P2 (X); -- compilation errror
```

Limited parameter references, like other, can be modes ``in``, ``in out``,
``access`` and ``aliased``. ``in`` limited parameters can accept either limited
and non-limted references. Other modes require a limited object, and apply
move semantics. For example:

```Ada
   procedure P1 (V : limited Integer);
   procedure P2 (V : limited in out Integer);

   X1 : limited Integer;
   X2 : Integer;

begin

   P1 (X); -- OK
   P1 (X2); -- OK

   P2 (X); -- OK
   P2 (X2); -- Compilation Error
```

Overloading and limited references
----------------------------------

The ``limited`` modifier on a subprogram parameter can lead to overloading. In
that case, the compiler will always select in priority the available profile
that has the limited parameter. For example

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

This may lead to situation with multiple parameters where there are several
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

Limited types in Ada are types that don't allow equality and copy. They have
obvious connection with limited variables and parameters.

In order to have a consistent model across the board for limited types and
objects, this proposal suggest to lift the restriction that limited types
can't be compared (which can be acheived by other means, such as using
an abstract equality operator for regular types. A specific mean for classes
can be considered outside of this proposal).

The overall rule is that while a limited type can't be copied, it can be moved.
As a consequence, the following are legal:

```Ada
   type Holder is limited ...

   V1 : Holder;
   V2 : limited Holder;
   V3 : limited Holder;
begin
   V2 := V3; -- legal, move semantics
   V2 := V1; -- Compilation error
```

Limited Access References and Types
-----------------------------------

Access types can now be declared limited. This means that their value can
only be moved, never copied. One of the consequence of moving a limited pointer
object is that its value is reset to null.

This first exemple shows usage of a limited reference on a regular pointer
object:

```Ada
   type Holder is  ...
   type A_Holder is access all Holder;

   V1 : limited A_Holder := new Holder;
   V2 : limited A_Holder;
begin
   V2 := V1; -- legal, move semantics. V1 is now null.
   V1.all.Some_Primitive; -- Constraint Error
```

The pointer type can itself be declared limited, which will have the added
constaint that it can't be copied.

```Ada
   type Holder is  ...
   type A_Holder is limited access all Holder;

   V1 : A_Holder := new Holder;
   V2 : A_Holder;
begin
   V2 := V1; -- legal, move semantics.  V1 is now null.
   V1.all.Some_Primitive; -- Constraint Error
```

Access types may themselves be marked as pointing to limtied references by
marking their target types ``limited``. In that case, they are also implied
``all`` - ie general access types. Move semantics rules apply to their
dereference view:

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

One of the most interesting use of limited references is the implemetnation
of move assignment and constructor in complex classes. This example assumes that
the OOP revamp RFC is implemented and rely on the new constructors and
assignment primitives.

Copy and assignment primitives can now be provided with limited parameters.
Consider the following class:

```Ada
   type Holder is class record
      procedure Holder (Self : in out Holder; Source : T1);
      -- Regular by copy constructor

      procedure Holder (Self : in out Holder; Source : limited in out T1);
      -- Move constructor

      procedure ":=" (Self : in out T1; Source : T1);
      --  Regular assignment

      procedure ":=" (Self : in out T1; Source : limited in out T1);
      --  By copy assignment

      Size : Integer;
      Contents : Array_Access;
   end Holder;

   type Holder is class record

      ...

      procedure Holder (Self : in out Holder; Source : limited in out T1) is
      begin
         Self.Size := Source.Size;
         Self.Contents := Source.Content;

         Source.Size := 0;
         Source.Contents := null;
      end Holder;

      procedure ":=" (Self : in out T1; Source : limited in out T1);
      begin
         Free (Self.Contents);

         Self.Size := Source.Size;
         Self.Contents := Source.Content;

         Source.Size := 0;
         Source.Contents := null;
      end ":=";

  end Holder;
```

Consider also that we have a function that returns a Holder:

```Ada
function Create_Holder return Holder is
   Local_Object : Holder;
begin
   return Holder;
end Create_Holder;
```

Note that returned value of a function is always a ``rvalue``.
Move semantics will be used both in the case of limited and non limited
variables and parameters whenever the object that would be otherwise copied
is an ``rvalue``. For example:

```Ada
   V1 : Holder (Create_Holder); -- Move constructor
   V2 : Holder := Create_Holder; -- Move assignment
   V3 : limited Holder := Create_Holder;
   V4 : limited Holder;
begin
   V4 := V3; -- MOVE V3 into V4

   V1 := V4; -- COPY V4 into V1
```

'Move, 'Borrow and 'Copy
------------------------

Three new attributes are introduced:
- `'Move`, to convert a value that can't move moved, typically a `lvalue`, into
  a movable object.
- `'Copy`, to force a copy on a limited reference
- `'Borrow`, to allow borrowing of a limited reference from a parameter that
  shouldn't normally allow it.

```Ada
   V1 : Holder (Create_Holder); -- Move constructor
   V2 : Holder := Create_Holder; -- Move assignment
   V3 : limited Holder := Create_Holder;
   V4 : limited Holder;
begin
   V3 := V1'Move; -- MOVE V1 into V3
   V1 := V3'Copy; -- COPY V3 into V1
```

Note that Move requires a variable view of the object (a variable or an in out
parameter for example) also known a ``lvalue`` while copy can work with any
kind of view, in particular ``rvalue`` and ``constant``.

Parameter passing allow for a additional kind of conversion, a borrow
conversion that can be applied on an ``limited`` reference:

```Ada
   procedure P (V : in out Holder);
   V : limited Holder := Create_Holder (1000);

   procedure P (V2 : in Holder);
begin
   P (V); -- illegal
   P (V'Borrow); -- legal

   P2 (V) -- legal
```

When applying ``'Borrow`` to a limited reference, the compiler is allowed to
pass that object as a parameter. Note however that moving a variable parameter
is always unsafe, so the following will raise a warning:

procedure P (V : in out Holder) is
   L : aliased Holder;
begin
   L := V'Move;
   -- warning, possible use after move
end;

More on this in the next section.

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

Summary of assignments copy and move rules
------------------------------------------

| A := B, A => B     | T        | limited T | constant T | (T)      | (in out T) | (limited in T) | (limited in out T) |
| T                  | copy     | error     | copy       | copy     | copy       | error          | error              |
| limited T          | error    | move      | error      | error    | error      | error          | move               |
| constant T         | copy     | error     | copy       | copy     | copy       | error          | error              |
| (T)                | copy/ref | ref       | copy       | copy     | copy/ref   | error          | error              |
| (in out T)         | copy/ref | error     | error      | copy/ref | copy/ref   | error          | error              |
| (limited in T)     | error    | borrow    | error      | error    | error      | borrow         | borrow             |
| (limited in out T) | error    | move      | error      | error    | error      | error          | move               |

Reference-level explanation
===========================

See above.

Rationale and alternatives
==========================

There are a few dimentioning decisions made in that proposal that differ notably
from Rust:

- The use of move / borrow semantics is a choice made by the developer. It's
  an additional constrained imposed on object references. Specific coding
  standard may impose further restrictions, however it's not the default mode.
  This is a trade-off different from Rust. There's no strong argument here in
  whether one is better than the other, however given Ada legacy, it feels
  more natural to keep semantic of current programs unchanged.
- Use of borrow and move operations is not always implicit. In particular,
  when crossing the boundaries of regular and limited references, the developer
  often has to precise his itent, and mark the right end part of the operation
  with ``'Move``, ``'Borrow`` or ``'Copy``. While these could be inferred,
  it's impose to the developer as a way to explicit his intent in cases that
  are at the boundary of the model and which misunderstanding could lead
  to unanticipated behavior.

Drawbacks
=========

See above.

Prior art
=========

This is heavily influenced by C++ move semantics and to a lesser extent Rust
move mechanism.

Unresolved questions
====================

There is a potential overal between that feature and a full fetched
borrow-checker as provided by languages such as SPARK and Rust. The extent of
that overlap and the extent to which this feature could be used in that context
is unclear.

Another unresolved question is the potential interaction with tasking.

Future possibilities
====================

This proposal identifies new erroneous behaviors. Note that these are not new
per se, they are inherent to the underlying programming patterns that can
be developped with less safe methods, for example through the use of pointers
or through the use of ad-hoc move operations without the tracking offer by
the limtied notation. At present, these behaviors are identified through either
errors when they are certain, or warnings when they can occur only in certain
cases. Having a tools such as SPARK further clearing or confirming the
likelyhood of an error would be usefult.

Further coding style could also improve safety and consistency. For example:
- pointers objets should be forced to be limited by default, as to avoid
  unecessary aliasing and increase memory safety.
- limited types should always be referenced to by limited references.
- certain kind of warning on potential use after move should be turned
  to errrors, in particular when applied on parameter as their effect is non
  local.

- Feature Name: (fill me in with a unique ident, my_awesome_feature)
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- RFC Issue: (leave this empty)

Summary
=======

This proposal introduces a new notation ``limited``, denoting a reference to an
object also known as ``rvalue`` in languages such as C++. It can then be used
to denote a reference to a so called rvalue, and implement move semantics
similar to C++.

This proposal also addresses natural extensions of limited types made possible
by limited objects and parameters.

Motivation
==========

Ada is a language that has high performances and small footprint requirements
for embedded situations. Move semantics as describe in C++ allows to avoid
unecessary copies and remove certain cases of use of pointers while keeping
memory safety.

Guide-level explanation
=======================

limited variables and parameters
--------------------------------

We first need to define a ``rvalue``. A ``rvalue`` is a value that can only
be used in the right-end side of an assignment. It can either be a temporary, a
literal, or the return value of a function.

A variable or a parameter marked ``limited`` can only receive a ``rvalue``. For
example:

```Ada
   V1 : limited Integer := 1; -- OK
   V2 : limited Integer := 1 + 2; -- OK

   function F return Integer;
   V3 : limited Integer := F; -- OK

   V4 : Integer;
   V5 : limited Integer := V4; -- NOK

   procedure P (V : limited Integer);

   P (1); -- OK
   P (F); -- OK
   P (V1); -- NOK
   P (V4); -- NOK
```

Subprograms can be overloaded to with ``limited``. To resolve the overloading,
if the value provided is a ``rvalue`` and there is a subprogram that
accepts such value, it should always be prefered to one that doesn't.


```Ada
   V1 : Integer := 1;
   function F return Integer;

   procedure P (V : Integer); -- (1)
   procedure P (V : limited Integer); -- (2)

   P (1); -- Calls (2)
   P (F); -- Calls (2)
   P (V1); -- Calls (1)
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

Limited variables and parameters can only take rvalues - while "regular" ones
can take both, as it's the case today. For example:

```Ada
   V1 : Integer := 1;
   V2 : limited Integer := V1; -- Error, V1 is a lvalue
```

Move semantics
--------------

limited parameters allow to implement so-called move semantics. For that, we
assume that the OOP revamp RFC is implemented and rely on the new constructors
and assignment primitives.

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

'Move and 'Copy
---------------

An lvalue can be converted to a rvalue through the use of the 'Move attribute
or to an rvalue through the use of the 'Copy attribute. For example:

```Ada
   V1 : Holder (Create_Holder); -- Move constructor
   V2 : Holder := Create_Holder; -- Move assignment
   V3 : limited Holder := Create_Holder;
   V4 : limited Holder;
begin
   V3 := V1'Move; -- MOVE V1 into V3
   V1 := V3'Copy; -- COPY V3 into V1
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

Relationships with limited types
--------------------------------

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
   type Holder is limited class ...

   V1 : Holder;
   V2 : limited Holder;
   V3 : limited Holder;
begin
   V2 := V1'Move; -- legal, move semantics
   V3 := V2; -- legal, move semantics
   V1 := V2'Copy; -- compilation error, can't copy a limited type
```

Relationships with access types
-------------------------------

Access types can now be declared limited. This means that their value can
only be moved, never copied. E.g.:

```Ada
   type Holder is class ...
   type A_Holder is limited access all Holder;

   V1 : A_Holder := new Holder;
   V2 : A_Holder;
begin
   V2 := V1; -- legal, move semantics
   V1.all.Some_Primitive; -- illegal, value pointed by V1 has been moved
```

Unpon move of an access value, the previous pointer is reset to null. As a
consequence, re-using it will lead to constraint error until it's assigned:

```Ada
   type Holder is class ...
   type A_Holder is limited access all Holder;

   V1 : A_Holder := new Holder;
   V2 : A_Holder;
begin
   if Some_Condition then
      V2 := V1; -- legal, move semantics
   end if;

   V1.all.Some_Primitive; -- warning, may raise constraint error
```

Access types may themselves only point to ``rvalues`` by marking their target
types ``limited``. In that case, they are also implied ``all`` - ie general
access types. Move semantics rules apply to their dereference view

```Ada
   type Holder is class ...
   type A_Holder is access limited Holder;

   V1 : A_Holder := new Holder;
   V2 : limited Holder;
   V3 : Holder;
begin
   V2 := V1.all; -- legal, move semantics of the pointed object
   Free (V1);
   V1 := new Holder;
   V3 := V1.all'Copy; -- legal, copy semantics
   V3 := v1.all; -- illegal
```

Reference-level explanation
===========================

See above

Rationale and alternatives
==========================

TBD

Drawbacks
=========

TBD

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

Future possibilities
====================

TBD

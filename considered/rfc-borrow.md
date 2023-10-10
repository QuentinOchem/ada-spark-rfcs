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

This proposal studies the possibilty to develop a memory-safe and thread-safe
model for Ada, inspired by C++ move semantics and Rust borrow checker. In
particular we introduce two fundamental concepts:

- A new kind of variable, called a limited reference, which is a references to
  an object or a value that can't be referenced (or "owned" / "aliased")
  referenced by others. This is similar to a C++ rvalue reference or a rust
  variable.
  See [Limited References](https://github.com/QuentinOchem/ada-spark-rfcs/blob/move_semantics/considered/rfc-borrow-limited-references.md)
- An extension to limited types, which redefines assignment and parameter
  passing as being moved or borrow operations. This allows to implement in
  particular so called smart-pointers.
  See [Limited Types](https://github.com/QuentinOchem/ada-spark-rfcs/blob/move_semantics/considered/rfc-borrow-limited_types.md)

Object Orientation is making a specific usage of move semantics, inspired by C++.
See [Move Constructor](https://github.com/QuentinOchem/ada-spark-rfcs/blob/move_semantics/considered/rfc-borrow-move_constructor.md)

On top of these two fundamental concepts, we propose to extend their semantic
to provide thread safety capabilities..
See [Thread Safety](https://github.com/QuentinOchem/ada-spark-rfcs/blob/move_semantics/considered/rfc-borrow-thread_safety.md)

Finally, in order to offer sensible default behavior and handle legacy code, we
introduce profiles.
See [Language profiles](https://github.com/QuentinOchem/ada-spark-rfcs/blob/move_semantics/considered/rfc-borrow-profile.md)

The rest of this RFC define the fundamental concepts that the above is based on.

Move
----

A move operation is a new Ada operation that allows to move a value from an
reference to another one. It is introduced by the 'Move attribute. For example:

```Ada
   type A is access all Integer;

   Srce : A := new Integer;
   Dst : A;
begin
   Dst := Src'Move;
```

Two requirements need to be met for this operation to
be safe and correct:
- the source reference moved must own its value
- the source reference cannot be referenced after the move operation

In general, there is no provision in Ada to verify that either of these
requirements are met. As a consequence, the compiler will warn of potential
problems:

```Ada
   type A is access all Integer;

   Src : A := new Integer;
   Dst : A;
begin
   Dst := Src'Move;
   --  warning, Src may not own its value
   --  warning, Src may be referenced after its ownership is moved
```

We can define special cases where the compiler is responsible for statically
determinig that there is no problem just by looking at the control flow.
However, these would only cover a small range of simple cases. The mechanism
used to get guarantees is the one provided by limited references, described
in a sub-part of this proposal.

Using 'Move has specific effects on access types and class records:
- for access types, the value of the source reference is reset to null.
- for class records, the move constructor is called instead of the copy
  constructor, which may in effect reset some of the fields of the source
  reference.

Borrow
------

A borrow operation is a temporary move operation, introduced by the 'Borrow
attribute. Borrowing is always scoped, it starts on the initial introduction of
the 'Borrow call up until the end of the scope of the reference receiving the borrow
(which could be a variable, a parameter or a temporary value). Is is equivalent
to doing a 'Move in one direction, then on the other at the end of said scope.
For example, in the context of a local constant:

```Ada
   type Handle is new Integer;

   V1 : Handle := 0;
begin
   declare
      V2 : Handle := V1'Borrow;
   begin
      null;
      --  At this point, V2 has the value of V1
      --  At this point, accessing V1 is erroneous.
   end;
   --  At this point, V1 can be accessed again, V2 is out of scope.
```

Or in the context of parameter passing:

```Ada
   V : Handle;

   procedure P (Param : Handle);

begin

   P (V'Borrow); -- V is borrowed during the call
   --  V is accessible again here
```

Note that this syntax works for dynamic objects too - at compilation time, a
straighforward implementation is to generate a temporary for the borrowed
pointer in order to be able to restore it, so that:

```Ada
declare
   V2 : constant Handle := X.Y.Z'Borrow;
begin
   X := new Some_Structure;
end; -- X.Y.Z is released here
```

is equivalent to:

```Ada
declare
   V2 : Handle;
begin
   -- tmp = X.Y.Z'Access;
   V2 := X.Y.Z'Borrow;
   -- X.Y.Z = null
   X := new Some_Structure;
   -- tmp.all = V2
end; -- X.Y.Z is released here
```

Borrowed value needs to be locally scoped. The following is illegal:

```Ada
   G : Handle;

   procedure P (V : Handle) is
   begin
      G := V'Borrow; - compilation error, borrowing to a value outside of scope
   end P;
```
Contrary to a move operation, a borrow operation doesn't impose constraints
on the source reference, as the value is given back at the end of the operation.

Clone and Alias
---------------

Two new operations are provided:

- 'Alias, which is in principle providing a so-called "shallow" copy
- 'Clone, which is in principle providing a so-called "deep" copy

The meaning of shallow and deep copies / 'Alias and 'Clone depends on the limited
types. For example, the semantic is the same for scalar types, but differ
significantely for tagged types. See specific sections for more details.

These operations are available for all Ada types with the exception of limited
types.

A limited type that allow 'Alias is introduced by the Alias (On|Off) aspect.
A type that allows copies is introduced by the Clone aspect. The private
view of a type may restrict the types of operations allowed. For example:

```Ada
package P is

   type T1 is limited private;
   type T2 is limited private with Alias => On;
   type T3 is limited private with Alias => On;

private

   type T1 is limited null record with Alias => On, Clone => On; -- OK
   type T2 is limited null record with Alias => On;  -- OK
   type T3 is limited null record; -- compilation error, Alias is Off by default

end P;
```

These can then be used as follows:

```Ada
package body P is

   procedure Proc (V : T1) is
      L : T1 := V'Clone;
   begin
      null;
   end Proc;

end P;
```

It is the responsibility of the type implemented to decide is an Alias is ok or
not. For example, an alias to a linked list would not be ok unless specific
reference counting mechanism is provided.

Aliasing and Clone cannot be enabled at the object level.

Replace
-------

'Replace is an operation that moves the value of an object, replacing the
value of the previous object in the same operation. For example:

```Ada
   type A is access all Integer;

   Src : A := new Integer;
   Dst : A;
begin
   Dst := Src'Replace (Src, null);
   --  warning, Src may not own its value
   --  Src is null here
```

It is legal to access the value of the previous object after the replace
operation (while doing so after a move is erroneous).

Swap
----

Move semantics are also provided with a 'Swap attribute, which allows to
exchange two values using move semantics, e.g.:

```Ada
   A, B : Handle;
begin
   A'Swap (B);
```

It's important to provide this primitive, as move semantics do not allow to
temporarily topy a value as it's usually done in swap implementation.

Reference-level explanation
===========================

See above.

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

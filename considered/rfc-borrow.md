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
model for Ada, inspired by C++ move semantics and Rust borrow checker. The
following aspects are discussed:

- The capacity to identify an object as containing a unique value. This is
  similar to what's called a rvalue reference in C++. See
  `Limited objects <https://github.com/QuentinOchem/ada-spark-rfcs/blob/move_semantics/considered/rfc-borrow-limited-objects.md>`_
- The capacity to control the number of aliases and allowed read / write
  operations for objects pointed by dynamic memory. This is similar to Rust
  borrow semantics. See
  `Safe access <https://github.com/QuentinOchem/ada-spark-rfcs/blob/move_semantics/considered/rfc-borrow-limited_access.md>`_
- Additional thread safety capabilities.
See `Limited <https://github.com/QuentinOchem/ada-spark-rfcs/blob/move_semantics/considered/rfc-borrow-thread_safety.md>`_
- A way to control backward incompatible semantics on the above and ensure that
  the default mode is the safe one.
  See `Language profiles <https://github.com/QuentinOchem/ada-spark-rfcs/blob/move_semantics/considered/rfc-borrow-profile.md>`_

The rest of this RFC define the fundamental concepts that the above is based on.

Move
----

A move operation is a new Ada operation that allows to move a value from an
reference to another one. Once a value has been moved, it cannot be refered to
from the previous reference, which needs to be assigned to a new value before
being used. Depending on the situation, this can be checked at run-time, by the
compiler or by static analysis and formal analysis tools.

This concept of move can be used in various ways by specific proposals. However,
it can be used right away in Ada through the use of the 'Move attribute. When
applied to a pointer, the semantic of a move operation is:
- to assign the value from the source pointer to the target pointer
- to reset the source pointer to null.

For example:

```Ada
   type A is access all Integer;

   V1 : A := new Integer;
   V2 : A;
begin
   V2 := V1'Move;
   --  At this point, V2 points to the object previously pointer by V1
   --  At this point, V1 is null. Accessing V1 is erroneous.
```

Move can be used for other types. For example:

```Ada
   type Handle is new Integer;

   V1 : Handle := 0;
   V2 : Handle;
begin
   V2 := V1'Move;
   --  At this point, V2 has the value of V1
   --  At this point, accessing V1 is erroneous.
```

Note that move operation introduce erroneous execution which may or may not
be automatically detectable. The compiler must:
- issue a compiler error where incorrect usage can be statically detected
- issue a compiler warning where incorrect usage is possible

For example:

```Ada
   type Handle is new Integer;

   V1 : Handle := 0;
   V2 : Handle;
begin
   V2 := V1'Move;

   Put_Line (V1'Img); -- compiler error, used previously moved value
```

```Ada
   procedure (X : in out Handle) is
   begin
      V2 : Handle;
   begin
      V2 := X'Move;
      --  compiler warning, X may be used after moved
   end;
```

Borrow
------

A borrow operation is a temporary move operation, introduced by the 'Borrow
attribute. Borrowing is always scoped, it starts on the initial introduction of
the 'Borrow up until the end of the scope of the reference receiving the borrow
(which could be a variable, a parameter or a temporary value). Is is equivalent
to doing a 'Move in one direction, then on the other at the end of said scope.
An additional constraint is that a borrow operation is always constant. For
example, in the context of a local constant:

```Ada
   type Handle is new Integer;

   V1 : Handle := 0;
begin
   declare
      V2 : constant Handle := V1'Borrow;
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
end; -- still release the value of the previous pointer
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
end; -- still release the value of the previous pointer stored in tmp
```

Note that the borrowed value needs to be locally scoped. The following is
illegal:

```Ada
   G : Handle;

   procedure P (V : Handle) is
   begin
      G := V'Borrow; - compilation error, borrowing to a value outside of scope
   end P;
```

Copy and Alias
--------------

Two new operations are provided:

- 'Alias, which is in principle providing a so-called "shallow" copy
- 'Copy, which is in principle providing a so-called "deep" copy

The meaning of shallow and deep copies / 'Alias and 'Copy depends on the limited
types. For example, the semantic is the same for scalar types, but differ
significantely for tagged types. See specific sections for more details.

By default, these operations are not available for types.

When possible, a type that allow 'Alias is introduced by the Alias (On|Off) aspect.
A type that allows copies is introduced by the Copiable aspect. The private
view of a type may restrict the types of operations allowed. For examples:

```Ada
package P is

   type T1 is limited private;
   type T2 is limited private with Alias => On;
   type T3 is limited private with Alias => On;

private

   type T1 is limited null record with Alias => On, Copy => On; -- OK
   type T2 is limited null record with Alias => On;  -- OK
   type T3 is limited null record; -- compilation error, missing Aliasable aspect

end P;
```

These can then be used as follows:

```Ada
package body P is

   procedure Proc (V : T1) is
      L : T1 := V'Copy;
   begin
      null;
   end Proc;

end P;
```

It is the responsibility of the type implemented to decide is an Alias is ok or
not. For example, an alias to a linked list would not be ok unless specific
reference counting mechanism is provided.

Aliasing and Copy cannot be enabled on a specific object.

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

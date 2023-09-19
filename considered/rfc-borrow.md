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

Safe Accesses
--------------

This RFC introduces the concept of safe access (somehwat similar to smart
pointer) which is an access type with additional capabilities such as automatic
memory reclaimation or reference counting. A smart access is declared on a
pointer type with the `Safe_Access` aspect applied on it:

```Ada
   type Cell is record
      X : Integer;
   end record;

   type A is access all Cell with Safe_Access;
```

By default, the value pointed by a safe access is not modifiable.
The following is illegal:

```Ada
   V : A := new Cell (X => 42);

   V.X := 0; --  run-time error
```

One direct consequence is that (by default) you can't pass a safe access
directly to a subprogram:

```Ada
   V : A := new Cell (X => 42);

   procedure P (Param : A);

   P (V); -- compilation error
```

By default, aliasing safe accesses is illegal. You can't do the following:

```Ada
   V1 : A := new Cell (X => 42);
   V2 : A := V1; --  compilation error
```

By default, smart access can't point to the stack. The following is illegal:

```Ada
   L : aliased Cell;
   V : A := L'Access; --  compilation error
```

Aliasing values, modification of the pointed value as well as accessing the
stack can be done through additional capabilities described in this proposal.

Memory pointed by smart accesses can be manually reclaimed:

```Ada
   declare
      V : A := new Cell (X => 42);

      procedure Free is new Unchecked_Deallocation (Cell, A);
   begin
      Free (V);
   end;
```

It will be automatically reclaimed upon finalization of the pointer as well, or
if assigned a different value:

```Ada
   declare
      V : A := new Cell'(X => 42);

      procedure Free is new Unchecked_Deallocation (Cell, A);
   begin
      V := new Cell'(X => 43); -- Free first allocated value
   end; -- Free the second allocated value
```

Move
----

A new attribute, 'Move, is provided to smart accesses. Once moved, the
value of the access is not reachable by the previous variable - to ensure
this behavior at run-time, it is reset to null. For example:

You can however move the value from one pointer through the next with the 'Move
attribute:

```Ada
   V1 : A := new Cell (X => 42);
   V2 : A := V1'Move; --  V1 is null after that statement

   procedure P (Param : A);

   P (V'Move);
```

An additional attribute is provided to swap to values (moving one value to the
next respectively):

```Ada
   V1 : A := new Cell (X => 42);
   V2 : A := new Cell (X => 43);
begin
   V1'Swap (V2);
```

Borrowing
---------

Borrowing is a way to create a temporary move of a pointer to a specific
object. Borrowing is always scoped, it starts on the initial introduction of
the 'Borrow up until the end of the scope. Is is equivalent to doing a 'Move
in one direction, then on the other at the end of said scope. For example:


```Ada
   type A is access all Cell with Safe_Pointer;

   V1 : A := new Cell;
begin

   declare
      V2 : A;
   begin
      V2 := V1'Borrow; -- at this stage, V1 is null, borrowed by V2.
   end; -- at this stage, V2 get back the borrowed value V1
```

Note that this synax works for dynamic objects too - at compilation time, a
straighforward implementation is to generate a temporary for the borrowed
pointer in order to be able to restore it, so that:

```Ada
declare
   V2 : A;
begin
   V2 := X.Y.Z'Borrow;
   X := new Some_Structure;
end; -- still release the value of the previous pointer
```

is equivalent to:

```Ada
declare
   V2 : A;
begin
   -- tmp = X.Y.Z'Access;
   V2 := X.Y.Z'Borrow;
   -- X.Y.Z = null
   X := new Some_Structure;
   -- tmp.all = V2
end; -- still release the value of the previous pointer stored in tmp
```

Borrowing is the canonical way to pass pointers to subprogram where smart
pointers do not allow aliases:

```Ada
   V : A;

   procedure P (Param : A);

begin

   P (V'Borrow);
   --  V is accessible again here
```

One of the consequences of the above is that it's not possible to escape a
borrowed value. Consider the following:

```Ada
   G : A;

   procedure P (V : A) is
   begin
      G := V'Borrow;
   end P;
```

In the above example, borrowed value is returned at the end of the procedure
and G will be reset to null.

The above should be source of analysis for potential mistake by the compiler,
static analysis or formal proof tools.

Aliased Safe Access
-------------------

A safe access can be configured to allow  aliases. Allowing
aliases on safe pointers enables refconting. This can be specified by adding
a parameter to the Safe_Access aspect:

```Ada
   type A is access all Cell with Safe_Access => (Alias);
```

Adding this capabilty to a pointer allows for a new kind of assignment, 'Alias,
which aliases the pointer.

```Ada
      V1 : A := new Cell; -- the target object is created, refcount 1
      V2 : A;
   begin
      V2 := V1'Alias;
      --  V1 and V2 point to the same object
      --  The target object is now refcount 2

      Free (V1); -- refcount is now 1, no free of the data
      Free (V2); -- refcount is now 0, data is freed
```

An alias can be used in a subprogram call:

```Ada
   V : A := new Cell (X => 42);

   procedure P (Param : A);

   P (V'Alias); -- compilation error
```

Unchecked Aliases
-----------------

In some situations, it's necessary to create a "weak" reference, that is a
reference to an object that is not ref-counting. This is fundamentally a safety
risk, as there's no way to ensure that the memory will not be accessed after
all the other object are dereferenced. It's however possible, as long as the
pointer type allows such operation:

```Ada
   type A is access all Cell with Safe_Access => (Unchecked_Alias);
```

Note that Unchecked_Alias implies Alias.

Adding this capabilty to a pointer allows for a new kind of assignment,
'Unchecked_Alias:

```Ada
      V1 : A := new Cell; -- the target object is created, refcount 1
      V2 : A;
   begin
      V2 := V1'Unchecked_Alias;
      --  V1 and V2 point to the same object
      --  The target object is still refcount 1

      Free (V1); -- data is freed
      -- There's no protection against V2 usage at this stage
```

Other restrictions for safe accesses are still enabled on V2 - in particular
V2 is readonly in this example.

Write-Enabled Borrowing
-----------------------

By default, safe pointers can only be read. It is however possible to
temporarily borrow a mutable view of the accessed object. Note that as long as
a mutable view is borrowed, no other object can acquire another mutable view
nor read the object.

Borrowing a mutable view is done through the 'Write_Borrow aspect:

```Ada
   type A is access all Cell with Safe_Pointer;

   V1 : A := new Cell;
begin

   declare
      V2 : A;
   begin
      V2 := V1'Write_Borrow;
      V2.X := 5; -- legal
   end; -- borrow to V1 is released
```

Note that it's prefectly reasonable to have a shorcut version of the above if
only one statement is needed:

```Ada
   type A is access all Cell with Safe_Pointer;

   V1 : A := new Cell;
begin
   V1'Write_Borrow.X := 5
```

In that case, the borrow is released right after the assignment.

```Ada
   type A is access all Cell with Safe_Pointer (Alias);

   V1 : A := new Cell;
   V2 : A := V1'Alias;
begin

   declare
      V3 : A;
   begin
      V3 := V1'Write_Borrow;
      V3.X := 5; -- legal
      Put_Line (V2.all'Img); -- runtime error
   end;
```

Writable Views
--------------

A smart access can be refered to through its "writable" view, through the
'Writable aspect associated with the type, e.g.:

```Ada
   type A is access all Cell with Safe_Pointer (Alias);

   V1 : A'Writable := new Cell;
begin

   V1.X := 0; -- legal, V1 is writable
```

Objects of this kind need to be assigned a writable view, either a borrowed
or an aliased one. Moving a writable value is also permitted. E.g.:

```Ada
   V1 : A := new Cell;
   V2 : A'Writable := V1'Write_Borrow; -- legal

   V3 : A := new Cell;
   V4 : A'Writable := V3'Alias; -- compilation error

   V7 : A := new Cell;
   V8 : A'Writable := V7'Move; -- run-time error

   V9 : A := new Cell;
   V10 : A := V9'Write_Alias;
   V11 : A'Writable := V10'Move; -- OK
```

'Writable can also be used in subprogram parameters to express the fact that
the target object needs to always be writable:

```Ada
   V1 : A := new Cell;

   procedure P (V : A'Writable);
begin
   P (V1'Borrow); -- run-time error
   P (V1'Writable_Borrow) -- legal
```

An object can also be assigned to a writable view. It can be further borrowed,
but can never be aliased (as aliasing would create a readable view which can
never be read):

```Ada
   V1 : A'Writable := new Cell;

   procedure P (V : A);

   P (V1'Borrow); -- legal
   P (V1'Write_Borrow); -- also legal BUT WHY ARE WE ALLOWING THIS HERE??? WRITABILITY IS NOT ASKED
   P (V1'Alias); -- compilation error
```

Accessing the Stack with Safe Pointers
--------------------------------------

By construction, the memory in stack or global data is managed automatically
by the application. As a consequence, it is not possible to have a regular
alias to it. The only available aliasing method is weak alias.

Setting the value of an object through the 'Access or 'Unchecked_Access
attributes is equivalent to creating a weak alias to it:

```Ada
   type A is access all Cell with Safe_Pointer (Unchecked_Alias);

   C : aliased Cell;
   V : A := C'Access; -- legal, V is a weak alias to the C value.
```

More on non-writable accessed data
----------------------------------

Note that a safe pointer is only controlling the fact that it pointed object
is writable or not. The pointed object may have additional fields that are
themselves pointers and handled separately. As a consequence, the following
is legal:

```Ada
   type C1 is record
      X : Integer;
   end record;

   type A1 is access all C1;

   type C2 is record
      C : C1;
   end record;

   type A2 is access all C2 with Safe_Access;

   V : A2 := new C2'(C => new C1);
begin
   V.C.X := 0;
```

Safe accesses and returned values
---------------------------------

Returning safe access values follow similar rules as for limited types. In
particular, it is not possible to return directly a safe access as it may
generate a copy:

```Ada
   G : A := new Cell;

   function Get_G return A is
   begin
      return G; -- compilation error
   end Get_G;
```

It is possible however to create an aliased view, use the move attribute or
create an in-place initialization:

```Ada
   G : A := new Cell;

   function Get_G return A is
   begin
      return G'Alias; -- OK
   end Get_G;

   function Move_G return A is
   begin
      return G'Move; -- OK
   end Move_G;

   function New_G return A is
   begin
      return new Cell; -- OK
   end New_G;
```

Boolean attributes
------------------

The following boolean attributes are available for safe pointer in order to
allow quering their state:

- V'Is_Unchecked_Alias - return true if V is an unchecked reference
- V'Is_Writable - return true if V can be written
- V'Is_Readable - return true if the value can be read (ie there's no write reference elsewhere)
- V'Aliases - return the current value of the reference counter

This allows to write contracts and checks on the status of pointers that go
beyong what could be done with the structural 'Writable attribute:

```Ada
   procedure Proc (P1, P2 : A)
   with Pre => P1'Is_Writable or P2'Is_Writable

   procedure Proc (P1, P2 : A) is
   begin
      if P1'Is_Writable then
         P1.X := 0;
      else
         P2.X := 0;
      end if;
   end Proc;
```

Thread-Safe pointers
--------------------

By default, safe pointers are thread-safe. This means that they can be accessed
from multiple thread, and that reference counting as well as flag setting
operations are atomic. An optimized implementation would implement these
operations through so called lock-free instructions.

An optional configuration can be offered to optimize safe accesses in cases
where they're either single threaded, or where the developper wants to take
the responsbilty of ensuring absence of race conditions:

```
   type A is access all Cell with Safe_Access (Thread_Safe => False);
```

Conversion from unsafe pointers
-------------------------------

Converting a regular access type to a safe access is a fundamental unsafe
operation. It is forbidden:

```
   type A1 is access all Cell;
   type A2 is access all Cell with Safe_Access;

   V1 : A1 := new Cell;
   V2 : A2 := A2 (V1); -- illegal
```

A workaround to the above is to consider the accessed data to be a weak access
and use 'Access as if it were data on the stack:

```
   type A1 is access all Cell;
   type A2 is access all Cell with Safe_Access (Unchecked_Alias);

   V1 : A1 := new Cell;
   V2 : A2 := V1.all'Access;
```

The guarantees provided there are then the same ones as for weak accesses.

Generalization to non-access types
----------------------------------

The concept of safe accesses can be extended to any types. Indeed, it is
sometimes convenient to create one "virtual" access, for example through
an integer handle:

```
   type Handle is Integer with Safe_Access;
```

In these cases, the compiler will add necessary data to implement semantics.
For example:

```
   type Handle is Integer with Safe_Access (Alias);

   V1 : Handle := Get_Handle;

   V2 := V1'Alias; -- refcount
```

If the type has a destructor associated with it (assuming OOP RFC) or is a
controlled type (assuming controlled types are not obsolete), then finalization
will be called when refcount reaches 0.


Data representation and perfomance considerations
-------------------------------------------------

A natural implementation of safe pointers is to add additional fields to the
pointer types:

- The accessed data needs to be possibly associated with a reference count
- The accessed data needs to be possibly associated with a flag following if it
  has an active write reference.
- The access needs to have a flag to know if it's a weak access

An optimised implementation would only create the above data when necessary
(that is, for access types allowing the above operations).

In addition to the above, it's important to note additional checks needed
upon access to the data when write references are allowed. Each time an access
is performed, a boolean check need to be performed to know if the operation is
allowed.

Reference-level explanation
===========================

See above.

Rationale and alternatives
==========================

This proposal uses an aspect-based syntax. The same semantic could be kept with
a trait-based syntax, should traits be included in a further version of Ada.

Drawbacks
=========

See above.

Prior art
=========


Unresolved questions
====================


Future possibilities
====================

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

Dereferencing the value of a smart pointer is done through an extension of the
borrow semantics, which prevents in particular aliased borrowing. This is
described in more details in a later section in this RFC.

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

Move and Borrow
---------------

Assignment or parameter passing is not legal by default with safe accesses.
They need to be done through explicit 'Move and 'Borrow attributes. This is
allowing to clearly differenciate these operations for aliasing (see below).

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

Note that the reference counter is automatically decreased upong finalization
of the safe access - which can lead to a deallocation of the object.

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

Dereferencing and Borrowing
---------------------------

Borrowing a safe pointer is subject the the same semantics as borrowing any
other object.

Dereferencing always implies a borrow on the target object operation. For
example:

```Ada
   procedure P (X : Cell);
   V : A;
begin
   P (V.all);
   V.X := 0;
```

is equivalent to:

```Ada
   procedure P (X : Cell);
   V : A;
begin
   P (V'Borrow.all);
   V'Borrow.X := 0;
```

Depending on the context, this borrow operation may create either a readonly
or a read-write borrow operation. For example:

```Ada
   type A is access all Cell with Safe_Pointer;

   V1 : A := new Cell;

   procedure P1 (V : Cell);
   procedure P2 (V : in out Cell);
begin
   P1 (V1.all); --  Create a readonly borrowing
   P2 (V1.all); -- Creates a read-write borrowing for the duration of the call

   V1.X := 0; -- Create a read-write borrowing for the duration of the assignment
```

Borrowing a read-write view of an object has consequences on the operations that
can then be performed by the pointer. For the duration of the borrowing, no
other code can dereference that pointer, even just for reading. This is checked
dynamically by some state stored by the pointer. As a consequence, the following
will yield an exception:

```Ada
   type A is access all Cell with Safe_Pointer;

   V1 : A := new Cell;

   procedure P1 (Arg1 : Cell; Arg2 : A) is
   begin
      Arg2.X := 0; -- Exception raised, the Arg2 actual is protected against access
   end Pl;

begin
   P1 (V1.all, V1);
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

   function New_G return A is
      L : A := new Cell;
   begin
      return L'Move; -- OK
   end New_G;
```

Boolean attributes
------------------

The following boolean attributes are available for safe pointer in order to
allow quering their state:

- V'Is_Unchecked_Alias - return true if V is an unchecked reference
- V'Is_Borrowable - return true if V can be borrowed
- V'Aliases - return the current value of the reference counter

This can help checking wether a particular operation is possible on a smart
access. The example in a previous section could be safely rewritten:

```Ada
   type A is access all Cell with Safe_Pointer;

   V1 : A := new Cell;

   procedure P1 (Arg1 : Cell; Arg2 : A) is
   begin
      if Arg2'Is_Borrowable then
         Arg2.X := 0; -- Exception raised, the Arg2 actual is protected against access
      end if;
   end Pl;

begin
   P1 (V1.all, V1);
```

This could also be used as a precondition:

```Ada
   procedure P1 (Arg1 : Cell; Arg2 : A)
   with Pre => Arg2'Is_Borrowable;
```

Note that in presences of aliases, the borrowable property of an argument can
change within the procedure. A more restrictive version of the above ensuring
that there are no other aliases when calling that subprogam:

```Ada
   procedure P1 (Arg1 : Cell; Arg2 : A)
   with Pre => Arg2'Is_Borrowable and then Arg2'Aliases = 0;
```

This is still insufficient in the context of multi-thread applications, which
are discussed in a separate RFC.

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


Data representation and perfomance considerations
-------------------------------------------------

A natural implementation of safe pointers is to add additional fields to the
pointer types:

- The accessed data needs to be possibly associated with a reference count
- The accessed data needs to be possibly associated with a flag following if it
  has an active write borrow.
- The access needs to have a flag to know if it's a weak access

An optimised implementation would only create the above data when necessary
(that is, for access types allowing the above operations). In particular,
smart pointers that allow neither aliasing nor weark references do not need
any additional data.

In addition to the above, it's important to note additional checks needed
upon access to the data when write references are allowed. Each time an access
is performed, a boolean check need to be performed to know if the operation is
allowed.

Generalization to non-access types
----------------------------------

TODO: isn't it where we hit what we already describe in the limited types?
Could we just have either one or the other concept?

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

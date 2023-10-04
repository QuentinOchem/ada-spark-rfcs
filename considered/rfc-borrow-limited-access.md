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

Limited Accesses
----------------

This RFC defines rules related to limited access (somehwat similar to smart
pointer) which is an access type with additional capabilities such as automatic
memory reclaimation or reference counting. A limited access is introduced by
the usage of the word `limited` in its declaration:

```Ada
   type Cell is record
      X : Integer;
   end record;

   type A is limited access all Cell;
```

Note that it is also possible to declare a type as being explicitely
non-limited:

```Ada
   type A is not limited access all Cell;
```

This can be useful as there are ways (described outside this RFC) to decide
on which kind of access should be the default one.

By default, aliasing limited accesses is illegal. You can't do the following:

```Ada
   V1 : A := new Cell (X => 42);
   V2 : A := V1; --  compilation error
```

By default, limited access can't point to the stack. The following is illegal:

```Ada
   L : aliased Cell;
   V : A := L'Access; --  compilation error
```

Dereferencing the value of a limited access is done through an extension of the
borrow semantics, which prevents in particular aliased borrowing. This is
described in more details in a later section in this RFC.

Aliasing values, modification of the pointed value as well as accessing the
stack can be done through additional capabilities described in this proposal.

Memory pointed by limited accesses can be manually reclaimed:

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

Aliased Limited Accesses
------------------------

Limited access do not provide a 'Copy primitive, copies cannot be enabled.
By default, they do not allow aliasing, but can be configured to allow that

```Ada
   type A is limited access all Cell with Alias => On;
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

   P (V'Alias);
```

Note that the reference counter is automatically decreased upong destruction
of the limited access - which can lead to a deallocation of the object.

Unchecked Aliases
-----------------

In some situations, it's necessary to create a "weak" reference, that is a
reference to an object that is not ref-counting. This is fundamentally a safety
risk, as there's no way to ensure that the memory will not be accessed after
all the other object are dereferenced. It's however possible, as long as the
pointer type allows such operation:

```Ada
   type A is limited access all Cell with Unchecked_Alias => On;
```

Note that Unchecked_Alias => On implies Alias => On.

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

Other restrictions for limited accesses are still enabled on V2 - in particular
V2 is readonly in this example.

Dereferencing and Borrowing
---------------------------

Dereferencing a limited access always implies a borrow on the target object
operation. For example:

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
   type A is limited access all Cell;

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
   type A is limited access all Cell;

   V1 : A := new Cell;

   procedure P1 (Arg1 : Cell; Arg2 : A) is
   begin
      Arg2.X := 0; -- Exception raised, the Arg2 actual is protected against access
   end Pl;

begin
   P1 (V1.all, V1);
```

Accessing the Stack with Limited Pointers
-----------------------------------------

By construction, the memory in stack or global data is managed automatically
by the application. As a consequence, it is not possible to have a regular
alias to it. The only available aliasing method is weak alias.

Setting the value of an object through the 'Access or 'Unchecked_Access
attributes is equivalent to creating a weak alias to it:

```Ada
   type A is access all Cell with Unchecked_Alias => On;

   C : aliased Cell;
   V : A := C'Access; -- legal, V is a weak alias to the C value.
```

Limited accesses and returned values
------------------------------------

Returning limited access values follow similar rules as for other limited
objects. In particular, it is not possible to return directly a limited access
as it may generate a copy:

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
   type A is limited access Cell;

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

The following boolean attributes are available for limited accesses in order to
allow quering their state:

- V'Is_Unchecked_Alias - return true if V is an unchecked reference
- V'Is_Borrowable - return true if V can be borrowed
- V'Aliases - return the current value of the reference counter

This can help checking wether a particular operation is possible on a smart
access. The example in a previous section could be safely rewritten:

```Ada
   type A is limited access all Cell;

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
   type A1 is not limited access all Cell;
   type A2 is limited access all Cell;

   V1 : A1 := new Cell;
   V2 : A2 := A2 (V1); -- illegal
```

A workaround to the above is to consider the accessed data to be a weak access
and use 'Access as if it were data on the stack:

```
   type A1 is not limited access all Cell;
   type A2 is limited access all Cell with Unchecked_Alias => On;

   V1 : A1 := new Cell;
   V2 : A2 := V1.all'Access;
```

The guarantees provided there are then the same ones as for weak accesses.

Data representation and perfomance considerations
-------------------------------------------------

A natural implementation of limited accesses is to add additional fields to the
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

On of the topic that is not addressed in these proposals is the concept of
lifetimes. Ada has a somewhat similar topic, accessibilty checks, which could
be extended. It's not clear yet how useful it would be to go beyond what's
already available, but can be considered as an extension.

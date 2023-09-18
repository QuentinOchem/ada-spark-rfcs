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

Limited pointers
----------------

On the previous RFC, we introduced the concept of limited pointers. One of
the specificities of these pointers is to implement move semantics upon
assignment:

```Ada
      type A is limited access Cell;

      V1 : A := new Cell;
      V2 : A;
   begin
      V2 := V1;
      --  equivalent to V2 := V1'Move;
      --  V1 has been moved to V2
      --  V1 is not accessible anymore and reset to null
```

This proposal introduces three new operations:
- The ability to enable re-counted pointers, and create read-only memory aliases
- The ability to create a temporary write reference to an object
- The ability to create a non ref-counted "weak" reference to an object

Refcounted pointers
-------------------

Move semantics are always the defaul mode for an assignment. However, a limited
access type can be extended with the capability of doing refcounting:

```
   type A is limited access Cell
      with Smart_Access => (Alias);
```

Adding this capabilty to a pointer allows for a new kind of assignment, 'Alias,
which aliases the pointer. Note that aliases must be readonly. In order to call
'Alias, the object needs to be marked as readonly:

```Ada
      V1 : A := new Cell; -- the target object is created, refcount 1
      V2 : A;
   begin
      V1'Readonly; - bah, need something else......
      V2 := V1'Alias;
      --  V1 and V2 point to the same object
      --  The target object is now refcount 2

      Free (V1); -- refcount is now 1, no free of the data
      Free (V2); -- refcount is now 0, data is freed
```

Weak references
---------------

In some situations, it's necessary to create a "weak" reference, that is a
reference to an object that is not ref-counting. This is fundamentally a safety
risk, as there's no way to ensure that the memory will not be accessed after
all the other object are dereferenced. It's however possible, as long as the
pointer type is declared with the right attributes, possibly together with
refcounting capabilities:

```
   type A is limited access Cell
      with Smart_Access => (Weak_Alias);
```

Adding this capabilty to a pointer allows for a new kind of assignment,
'Weak_Alias:

```Ada
      V1 : A := new Cell; -- the target object is created, refcount 1
      V2 : A;
   begin
      V1'Readonly;
      V2 := V1'Weak_Alias;
      --  V1 and V2 point to the same object
      --  The target object is still refcount 1

      Free (V1); -- data is freed
      -- There's no protection against V2 usage at this stage
```

Borrowed write access
---------------------

By default, aliased pointers can only be read. It is however possible to
temporarily borrow a writable view of an object pointed by a pointer.
When borrowed, the pointed object can only be accessed from that view. It's
possible to create other references but none can be accessed before releasing
the borrowed view.

Ability to borrow an object must be specified at the pointer type declaration:

```
   type A is limited access Cell
      with Smart_Access => (Write_Alias);
```

The cleanest way to access to a borrowed read-write reference is to create a
pointer specifically for this objective:


```Ada
      V1 : A := new Cell; -- the target object is created, refcount 1
      V2 : A;
   begin
      V1'Readonly;
      V2 := V1'Write_Alias;

      -- At this point V2 can be written
      V2.X := 0;

      Free (V2); -- Write alias is reclaimed here, data can be red by others
```

Another pattern may be to only turn Write_Alias directly on the statement:

```Ada
      V1 : A := new Cell; -- the target object is created, refcount 1
      V2 : A;
   begin
      V1'Readonly;
      V2 := V1'Alias;

      V2'Write_Alias.X := 0;
```

When there is an active write alias, other references can't be access for read
or write. For example, this will raise an exception:

```Ada
      V1 : A := new Cell; -- the target object is created, refcount 1
      V2 : A;

      procedure Proc (P1, P2 : A) is
      begin
         P1.X := 0; -- This will raise an exception from the call below, the pointed object is considered constant
         P2.X := 0;
      end Proc;

   begin
      Proc (V1, V2'Write_Alias);
      -- There's a dynamic check on P1.X
```

Write alias is released when freeing the pointer. That can happen when manually
invoking `Free`, or upon automatic memory reclaim, as shown later.

Note that it's possible to specify and test that pointers can be writable:

```Ada

   procedure Proc (P1, P2 : A)
   with Pre => P1'Write_Alias or P2'Write_Alias

   procedure Proc (P1, P2 : A) is
   begin
      if P1'Write_Alias then
         P1.X := 0;
      else
         P2.X := 0;
      end if;
   end Proc;

```


Automatic reclaim of memory
---------------------------

Smart accesses are automatically reclaiming memory. Note that to enable this
capability, you only need to mark a limited pointer as being smart:

```
   type A is limited access Cell with Smart_Access;
```

Pointers will automatically reclaim the memory with their associated data when
they are themselve finalized. For example:

```Ada
   declare
      V1 : A := new Cell;
   begin
      V1.X := 0;
   end; -- V1 is freed at this stage
```

It is legal to explitely free a pointer, in which case the reclaimation is
manually performed.

Refcounted pointers are dereferenced automatically upon finalization, the data
they point to will be freed once their refcount equals 0:

```Ada
   declare
      V1 : A := new Cell;
   begin
      V1.X := 0;
      declare
         V2 : A := V1'Alias;
      begin
         null;
      end; -- V2 is only refcounting -1, the object still exist
   end; -- V1 is freed at this stage
```

Weak reference do not lead to either decrement of refcounting nor deallocation.

Accessing the stack with limited accesses
-----------------------------------------

Data representation considerations
----------------------------------

Conversion from unsafe pointers
-------------------------------

Generalization to non-access types
----------------------------------

Reference-level explanation
===========================

See above.

Rationale and alternatives
==========================

Drawbacks
=========

See above.

Prior art
=========


Unresolved questions
====================


Future possibilities
====================

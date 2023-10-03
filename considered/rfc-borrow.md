- Feature Name: (fill me in with a unique ident, my_awesome_feature)
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- RFC Issue: (leave this empty)

Summary
=======


Motivation
==========

TODO: option types may allow for control over thread safety without the need of
a pointer! Maybe this is linked to that.

TODO: synced objects would require mutexes to be created, doesn't look proper
to let arbitrary accesses be synced. Maybe instead extend / simplify the
concept of protected?

X : protected Integer; -- X'Move X'Borrow and X'Swap are now synced

TODO: Smart_Access should be the default access under a default profile?

Pragma Ada_Profile (Memory_Safe, Thread_Safe)


Guide-level explanation
=======================

Two needs have emerged somewhat simulaneously to improve the language:

- The capacity to identify an object as containing a unique value. This is
  similar to what's called a rvalue reference in C++. See
  `Limited objects <https://github.com/QuentinOchem/ada-spark-rfcs/blob/move_semantics/considered/rfc-borrow-limited-objects.md>`_
- The capacity to control the number of aliases and allowed read / write
  operations for objects pointed by dynamic memory. This is similar to Rust
  borrow semantics. See
  `Safe access <https://github.com/QuentinOchem/ada-spark-rfcs/blob/move_semantics/considered/rfc-borrow-safe-access.md>`_

These two concept introduce commonly new notions to the Ada programming
language, notably the concept of moving and borrowing an object.

This part of the RFC is about defining these two notions, used in following
RFCs:

- limited references, used to implement uniqueness of values
- safe access, used to control usage of dynamic memory.

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

Thread-Safety
-------------

By default, Move and Borrow operations are not thread-safe. However, it is
possible to introduce this thread safety with 'Sync_Move, 'Sync_Borrow and
'Sync_Swap respectively. In these cases, the move operations (or back & forth
move operations) are done atomically.

White thread-safe, Sync_Move does not guarantee that asynchronous operations are
free of erroneous behaviors. Notably:

```Ada
   X : Handle;

   task T1 is
      Y : Handle;
   begin
      Y := X'Sync_Move;
   end T1;

   task T2 is
      Y : Handle;
   begin
      Y := X'Sync_Move;
   end T2;
```

In the above code, one of the task does the proper move operation. For the
other, accessing X is erroneous. Depending on the run-time behavior, this
can be however bounded. For example, if X is a pointer, its value is null and
dynamic checking on the value of Y can be made (ie - did I manage to move the
value indeed).

'Sync_Swap is an atomic operation. It is thread safe.

'Sync_Borrow creates a lock on the object itself so that no other thread
can either move, swap or borrow the value synchronously. The above can be
rewritten:

```Ada
   X : Handle;

   task T1 is
      Y : Handle;
   begin
      Y := X'Sync_Borrow;
      -- Y is valid until the end of the scope, then returned
   end T1;

   task T2 is
      Y : Handle;
   begin
      Y := X'Sync_Borrow;
      -- Y is valid until the end of the scope, then returned
   end T2;
```

Reference-level explanation
===========================

See above.

Rationale and alternatives
==========================

Drawbacks
=========

It's unclear that 'Sync_X operations are reasonably optimize here as they
relate to a value which is not necessary boxed. So for example, there needs
to be some kind of association between the address and the mutex, is that
something that OS APIs typically offer? Otherwise these should always be pointed
by safe pointers.

Prior art
=========


Unresolved questions
====================


Future possibilities
====================


Thread-Safety Additional Constraints
------------------------------------

To fully acheive memory safety in the context of threads, we need to go beyond
what this proposal is making. We need to be able to mark clearly operation as
being thread safe. For example, such subprogram could be marked "Thread_Safe":

```Ada
   procedure P with Thread_Safe;
```

To be thread-safe, a procedure must follow the following restrictions:

- It can only call other thread-safe operations
- It can only refer to copies or thread-safe copies / move / borrows of objects
  that are declared outside of its scope or that can be escaped from its scope
  (notably all pointers need to be dereferenced in a thread_safe way).

For example the following is illegal:

```Ada
   procedure P (V : access Integer) with Thread_Safe is
   begin
      V.all := 10; -- Error, dereference is not thread safe
   end P;
```

However this will work:

```Ada
   procedure P (V : access Integer) with Thread_Safe is
   begin
      V'Sync_Borrow.all := 10;
   end P;
```

Note that a thread safe subprogram may be called in a thread unsafe context.
Take for example:

```Ada
   procedure P (V : in out Integer) with Thread_Safe is
   begin
      V.all := 10;
   end P;

   A : access Integer;

   P (A.all);             -- Called from a thread unsafe context
   P (A'Sync_Borrow.all); -- Called from a thread safe context
```

Interrestingly, protected objects and tasks are not thread safe as they're
currently able to operate on arbitrary pointers. Under this proposal, we
would mark tasks and protected objects procedures Thread_Safe by default. This
is backwards-incompatible but is probably important enough to allow it and
force users to explicitely allow non-thread safety, e.g. in:


```Ada
   A : access Integer;

   task T with Thread_Safe => False;

   task T is
   begin
      A.all := 0; -- OK because T is not thread safe
   end T;
```

We should also allow for Thread_Safe to be default true for a given package
or a given partition.

Other proposals (safe pointers notably) further refines what can be done with
thread safety

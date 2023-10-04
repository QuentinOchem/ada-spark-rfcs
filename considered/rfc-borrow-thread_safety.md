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

Thread-Safe Objects
-------------------

By default, operations on types, in particular Move and Borrow and other
operations are not thread-safe. Thread safety can be introduced by making these
types `with Thread_Safe`:

```Ada
   type T is new Integer with Thread_Safe;
```

Conceptually, a synchronized type is a type that is protected against mutual
accesses for all operations. Such protection can be implemented through
mutexes, lock free instructions or even atomicity.

Borrow and Move operation notably become thread-safe for such objects:

```Ada
   X : Handle;

   task T1 is
      Y : Handle;
   begin
      Y := X'Move;
   end T1;

   task T2 is
      Y : Handle;
   begin
      Y := X'Move;
   end T2;
```

In the above code, one of the task does the proper move operation. For the
other, accessing X is erroneous. Depending on the run-time behavior, this
can be however bounded. For example, if X is a pointer, its value is null and
dynamic checking on the value of Y can be made (ie - did I manage to move the
value indeed).

'Swap is an atomic operation. It is thread safe.

'Borrow creates a lock on the object itself so that no other thread
can either move, swap or borrow the value synchronously. The above can be
rewritten:

```Ada
   X : Handle;

   task T1 is
      Y : Handle;
   begin
      Y := X'Borrow;
      -- Y is valid until the end of the scope, then returned
   end T1;

   task T2 is
      Y : Handle;
   begin
      Y := X'Borrow;
      -- Y is valid until the end of the scope, then returned
   end T2;
```

Thread-Safe operations
----------------------

To fully acheive memory safety in the context of threads, we need to restrict the
operations that are allowed for a given subprogram, to only refer to thread safe
operations.

A subprogram can be marked Thread_Safe:

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
   type Int_Access is limited access all Integer;

   procedure P (V : Int_Access) with Thread_Safe is
   begin
      V.all := 10; -- Error, dereference is not thread safe
   end P;
```

However this will work:

```Ada
   type Int_Access is limited access all Integer with Thread_Safe;

   procedure P (V : Int_Access) with Thread_Safe is
   begin
      V.all := 10; -- OK, borrowing is thread safe here
   end P;
```

Note that a thread safe subprogram may be called in a thread unsafe context.
Take for example:

```Ada
   procedure P (V : in out Integer) with Thread_Safe is
   begin
      V.all := 10;
   end P;

   type Safe_Access is limited access all Integer with Thread_Safe;
   type Unsafe_Access is limited access all Integer;

   Safe_A : Safe_Access;
   Unsafe_A : Unsafe_Access;

   P (Safe_A.all);   -- Called from a thread unsafe context
   P (Unsafe_A.all); -- Called from a thread safe context
```

Protected objects and tasks are not thread safe as they're
currently able to operate on arbitrary pointers and global objects, and they
can call thread-unsafe operations. Under this proposal, we
would mark tasks and protected objects procedures Thread_Safe by default. This
is backwards-incompatible but is probably important enough to allow it and
force users to explicitely allow non-thread safety, e.g. in:


```Ada
   A : access Integer;

   task T with Thread_Safe => False;

   task T is
   begin
      A.all := 0; -- Compilation Error
   end T;
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

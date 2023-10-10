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
- It can only have parameter that are passed by copy, move, or typed as thread
  safe.

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

Similarily:

```Ada
   type Int_Array is array (Integer range <>) of Integer;

   procedure P (Int_Array : Int_Access) with Thread_Safe;
   --  compilation error, non thread-safe parameter Int_Array may be passed
   --  as reference
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
      A.all := 0; -- Ok since we marked T as non-thread safe
   end T;
```

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

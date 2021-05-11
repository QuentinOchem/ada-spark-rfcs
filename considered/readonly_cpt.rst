- Feature Name: Read-Only Components
- Start Date: 
- RFC PR: (leave this empty)
- RFC Issue: (leave this empty)

Summary
=======

The Ada programming language provides a great way to map a specific record type
to device driver memory, thanks to representation clauses. However, some of the
constraints associated with record types make it difficult to model situations 
where certain memory addresses needs to be read-only, write-only, as well as
situation where the same memory address has a different semantic when being read
or being written. While this can be addressed through e.g. unchecked unions, 
it would be more faithful to the underlying representation to be able to specify
that a component fall into one of these specific categories. This proposal is 
aimed at providing these low level capabilites.

Motivation
==========

See above.

Guide-level explanation
=======================

Readonly and writeonly
----------------------

Components can be marked "read-only" and "write-only" within a record:

.. code-block:: ada

   type R is record
      A : Integer with Readonly;
      B : Integer with Writeonly;
      C : Integer;
   end record;

When doing so, operations that can be done where referencing a field are 
constrained to what the field allows. For example:

.. code-block:: ada

      X : R;
      V : Integer;
   begin
      X.A := 5; -- compilation error, A is readonly
      V := X.B; -- compilation error, B is readonly

When assigning values to an aggregate, readonly values are ommited.

.. code-block:: ada

   X1 : R := (0, 0); -- correct, intializing B and Character
   X2 : R := (0, 0, 0); -- incorrect, there only two writable values on R
   X3 : R := (A => 0, others => 1); -- incorrect, A is readonly
   
When assigning objects as a whole, readonly fields are considered not read,
writeonly fields are not written. E.g.:

.. code-block:: ada

   X1 : R := (0, 0); -- correct, intializing B and Character
   X2 : R := X1; -- X2.B value is undefined, X1.A is not copied

Aliased values
--------------

In addition to the above, it is possible to alias fields to the same address.
This alias can only be done on a volatile record type as we anticipate that
values can change independently of the normal data flow. In addition, fields 
must be explicitely marked as aliased in the representation in order to 
avoid aliasing fields by mistake

.. code-block:: ada

   type R is record
      A : Integer with Readonly;
      B : Integer with Writeonly;
      C : Integer;
   end record with Volatile;

   for R use
      A at aliased 0 range 0 .. 32;
      B at aliased 0 range 0 .. 32; -- OK, record is volatile, A and B are aliased
      C at 4 range 0 .. 32;
  end record;


Drawbacks
=========

TBD

Prior art
=========

TBD

Unresolved questions
====================

TBD

Future possibilities
====================

TBD

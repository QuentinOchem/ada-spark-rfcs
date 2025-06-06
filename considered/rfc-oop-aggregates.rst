- Feature Name:
- Start Date:
- RFC PR:
- RFC Issue:

Summary
=======

Motivation
==========

This RFC describes aggregates semantics in the context of new the new OOP
models. One of the guiding principles is that, unlike the current situation,
using an aggregate does not forgo the call of constructors.

The following Ada language constructions need to be supported:

.. code-block:: ada

      type Root is tagged record
         A, B : Integer;
      end record;

      type Child is new Root with record
         C, D : Integer;
      end record;

      V1 : Root := (1, 2);
      V2 : Child := (1, 2, 3, 4);
      V3 : Child := (Root with 3, 4);
      V4 : Child := (V1 with 3, 4); -- Call A (V1, Child) which by default then calls A (V1)
      V5 : Root := (V1 with delta A => 1); -- Copy constructor on V1, then modify the value (... we don't have a special constructor here - should we have a delta constructor???)
      --  TODO: Look at grandchild cases
   begin
      Root (V2) := (1, 2); -- Handle like a delta
      Root (V2) := V1; -- Same as constructor by extension - 'Aggregate will be the old values of V2 and can be modified. Eventually, we can have assignment constructor(?)
      Root (V2) := Root (V3); --  ????? hum.... should we call a copy constructor on V3 ?????
      --  TODO: Look at grandchild cases

      In assigments today:
         - you first finalize the destination object
         - then you construct

      So in:
         Root (V2) := (1, 2);

         - Capture the V2 values in a 'aggregate
         - Finalize V2
         - Perform a by extension aggregate on the entire value set (1, 2) for root and whatever previous values are

Aggregate Constructors
----------------------

Two new constructors are introduced:
- Aggregate Constructors
- Aggregate Constructor by extensions

Together with a new notation <type_name>'Aggregate which denotes a record with
the components of a specific to record, including its parents values. It can
be used in 2 ways:
- Directly in an Initializes aspect of a constructor of any of its type and / others
  subtypes, in which case it will initialize the components of that type (but
  not the parent).
- To refer to individual values
User cannot use this type anywhere else - and in particular not variable of such
type can be created.
In addition, any child aggregate type can be converted to its parent through
the usual view conversion.
For the avoidance of doubt, no primitive nor constructor can be called on these
types.

Note that aggregate do not call default constructors. As such (much like
constructors by copy) they are bypassing default construction and may need to be
either provided or removed to maintain full object consistency.

The default implementation of these constructors look as follows:

.. code-block:: ada

   procedure Root'Constructor (Self : in out Root; A : Root'Aggregate)
      with Initializes => (Root'Aggregate)
   is
   begin
      null;
   end Root'Constructor;

   procedure Child'Constructor (Self : in out Child; A : Child'Aggregate)
      with Super => (Self, Root'Aggregate (Child'Aggregate)),
           Initializes => (Child'Aggregate)
   is
   begin
      null;
   end Child'Constructor;

   procedure Root'Constructor (Self : in out Root; Base : Root; A : Root'Aggregate)
      with Initializes => (Child'Aggregate)
   is
   begin
      null;
   end Root'Construct;

   procedure Child'Constructor (Self : in out Child; Base : Root; A : Child'Aggregate)
      with Super => (Self, Base, Root'Aggregate (A)),
           Initializes => (Child'Aggregate)
   is
   begin
      null;
   end Child'Constructor;

Note that the constructor by extension must accept any type of the parent
derivation tree. So it is always has the very top of the derivation
tree type as second parameter. I.e. If we had a grandchild type it would look
like:

.. code-block:: ada

   procedure Grandchild'Constructor (Self : in out Child; Base : Root; A : Grandchild'Aggregate)
   with Super => (Self, Base, Child'Aggregate (A)),
         Initializes => (Grandchild'Aggregate)
   is
   begin
      null;
   end Child'Constructor;

These constructors are provided by default with the above semantic. A user can
also remove a constructor, for example if there's no way to create consistent
objects out of values, in the same way as other default constructors, e.g.:

.. code-block:: ada

   procedure Root'Constructor (Self : in out Root; A : Root'Aggregate) is abstract;

In that case, all constructions relying on this constructor are illegal, including
default generation of children constructors that would rely on it.

Simple Aggregate Constructor Scenario
-------------------------------------

Any situation where all individual components of a record type are provided
directly calls the simple aggregate constructor. Notably:

.. code-block:: ada

      V1 : Root := (1, 2);
      -- Calls Root'Constructor (V1, (1, 2))

      V2 : Child := (1, 2, 3, 4);
      -- Calls Child'Constructor (V2, (1, 2, 3, 4))

      V3 : Child := (Root with 3, 4);
      -- Calls Child'Constructor (V3, (Default values Root (prior to constructor) with 3, 4))

Partial assignment requires additional steps, as the values of the root object
need to be preserved, but the child ones are modified:

.. code-block:: ada

      Root (V2) := (1, 2);
      --  Create an aggregate in 2 steps (we can declare a 'Aggregate conceptually in generated code)
      --  First capture the Child values
      --  Agg : Child'Aggregate;
      --  Agg.C := V2.C;
      --  Agg.D := V2.D;
      --  Calls Root'Destructor (V2)
      --  Agg.A := 1;
      --  Agg.B := 2;
      --  Calls Child'Constructor (V2, Agg)

Note that an aggregate constructor should not presume of the consistency of the
values provided to it. In particular, in `Root (V2) := (1, 2);`, the partial
finalization of Root may introduce inconsistencies in the values in Child that
a constructor is responsible for detecting. If that's not possible, then the
implementer of the class has the possibilty to removing the aggregate
constructor.

Aggregate by Extension Constructor Scenario
-------------------------------------------

Situations that require taking into account a parent type will lead to the call
of a by extension constructor aggregate, which will then be responsible of
doing the necessary copies. Notably:

.. code-block:: ada

      V1 : Root := (1, 2);
      V4 : Child := (V1 with 3, 4);
      -- Calls Child'Constructor (V4, V1, (Values of V1 with 3, 4))

Note that there is a duplication here. You will have the original object
reference (V1) together with the whole aggregate. The user will have the
responsibility to implement the by-copy semantics of the base value together
with assigning the child values.

Constructor by extension also play a role in partial assignment. The following
is similar to the partial assignment with an aggregate, but will use the by
extension constructor in order to do the necessary copy of the original V1 value,
only the last step changes:

.. code-block:: ada

   begin
      Root (V2) := V1;
      --  Create an aggregate in 2 steps (we can declare a 'Aggregate conceptually in generated code)
      --  First capture the Child values
      --  Agg : Child'Aggregate;
      --  Agg.C := V2.C;
      --  Agg.D := V2.D;
      --  Calls Root'Destructor (V2)
      --  Agg.A := V1.A;
      --  Agg.B := V1.B;
      --  Calls Child'Constructor (V2, V1, Agg)

Indirect Aggregate by Extension Constructor Scenario
----------------------------------------------------

Aggregate by extension may refer to a type that is higher up in the derivation
chain, The generate code in this case is the exact same:

.. code-block:: ada

      type Root is tagged record
         A, B : Integer;
      end record;

      type Child is new Root with record
         C, D : Integer;
      end record;

      type Grandchild is new Root with record
         E, F : Integer;
      end record;

      V_Root : Root;
      V_Child : Child;
      V_Grandchild : Grandchild := (V_Root with 3, 4, 5, 6);
      -- Calls Child'Constructor (V_Grandchild, V_Root, (Values of V_Root with 3, 4, 5, 6))

The aggregate by extension has the topmost type has second parameter. Partial
assignment will need to create aggregates in two steps, first with the unchanged
values (the children) then with the changed values (the parents), eg.g
Similarily in:

.. code-block:: ada

   begin
      Root (V_Grandchild) := V_Root;
      --  Agg : Grandchild'Aggregate;
      --  Agg.C := V_Grandchild.C;
      --  Agg.D := V_Grandchild.D;
      --  Agg.E := V_Grandchild.E;
      --  Agg.F := V_Grandchild.F;
      --  Calls Root'Destructor (V_Grandchild)
      --  Agg.A := V_Root.A;
      --  Agg.B := V_Root.B;
      --  Calls Grandchild'Constructor (V_Grandchild, V_Root, Agg)

      Child (V_Grandchild) := V_Child;
      --  Agg : Grandchild'Aggregate;
      --  Agg.E := V_Grandchild.E;
      --  Agg.F := V_Grandchild.F;
      --  Calls Root'Destructor (V_Grandchild)
      --  Agg.A := V_Child.A;
      --  Agg.B := V_Child.B;
      --  Agg.C := V_Child.C;
      --  Agg.D := V_Child.D;
      --  Calls Grandchild'Constructor (V_Grandchild, V_Child, Agg)

Delta Aggregates
----------------

Delta aggregate use the regular constructor by copy, followed by assignment to
the individual components. For example:

.. code-block:: ada

   V1 : V_Root;
   V2 : V_Root := (V1 with delta A => 0);

   -- Calls Root'Constructor (V1, V2);

TODO: This doesn't work with assignment, as the destination object would be
freed. This is actually an endemic problem to be fixed as V1 := V1 wouldn't work
either. Probably has an impact on partial assignment too.

Impact on Generics
------------------

Like other default constructors, aggregates constructors are passed by default
to generic formals. They are all required for tagged types formals unless
explicitely marked abstract.

For the aggregate by extension constructor, the type of the second parameter
(the extended object) need to be the topmost visible type. E.g.:

.. code-block:: ada

   generic
      type T1 is tagged private;

      procedure T1'Constructor (Self : in out T1; Base : T1; Agg : T1'Aggregate) is abstract;

      type T2 is new Some_Type with private;

      procedure T1'Constructor (Self : in out T2; Base : Some_Type; Agg : T1'Aggregate) is abstract;
  begin

Re-introducing aggregate constructors
-------------------------------------

Aggregates constructors can be re-introduced in the derivation chain. In that
case, since it's an explicit constructor, the developer will chose which
parent constructor to call. This will have impact on legal operations. E.g.

.. code-block:: ada

      type Root is tagged record
         A, B : Integer;
      end record;

      type Child is new Root with record
         C, D : Integer;
      end record;

      procedure Child'Constructor (Self : in out Child; Base : Root; Agg : Child'Aggregate) is abstract;

      type Grandchild is new Root with record
         C, D : Integer;
      end record;

      procedure Grandchild'Constructor (Self : in out Grandchild; Base : Root; Agg : Grandchild'Aggregate);

      V_Root : Root;

      V_Child_1 : Child := (Root with 1, 2);
      -- legal, this relies on the aggregate constructor

      V_Child_2 : Child := (V_Root with 1, 2);
      -- error, this relies on the by extension constructor

      V_Grandchild : Grandchild := (V_Child_1 with 3, 4);
      -- legal, by extension constructor is provided

Optimization Considerations
---------------------------

While this is not required by the language, the compiler should optimize
aggregate where no explicit constructors are provided.

Reference-level explanation
===========================


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

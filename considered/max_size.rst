 - Feature Name: Max_Size
 - Start Date: 2020.03.01
 - RFC PR: 
 - RFC Issue: 

Summary
=======

This proposal introduces a new aspect, “Max_Size” that can be applied to various entities (types, objects) and denotes the maximum 
size (in bits) that these objects can take. This allows to have indefinite objects being stored without the use of pointers, as the 
compiler will reserve the maximum amount of memory to cover all possible cases. For example, while:

.. code-block:: ada

   V : R’Class;

Is not allowed, something like:

.. code-block:: ada

   V : R’Class with Max_Size => 8 * 16;

Would be.

Motivation
==========

No matter how much capabilities are developed to ensure their safety, pointers introduce problems. They are less efficient than direct 
memory access, require to cater for null values, invalid references and so. To that end, anything that can prevent the use of pointer 
is an improvement in terms of safety and efficiency. It also makes some code possible to write in the first place, as pointers are 
forbidden in many safety-related applications.

Ada 2012 AI provides Bounded_Indefinite_Holders which solves partially the issue. However, this RFC goes beyond in its attempt at g
eneralize the notion of maximum size.

Guide-level explanation
=======================

Max_Size can be applied to various entities in Ada:

 - Local and global variables
 - Components of records and arrays
 - Public views of private types
 - Tagged types
 - Parameters and return types
 - Generic formal parameters

In each situation, they define the maximum size allowed for a given type / object and enable the usage of these types and objects in 
places where only definite types would be allowed otherwise. The usage of such notation also removes a number of discriminants / 
bounds / tag checks.

Max_Size on Array Types
-----------------------

When used on types, Max_Size denotes the maximum size that any value of such type can have. For example:

.. code-block:: ada
   type T is array (Integer range <>) of Boolean with Max_Size => 8 * 64;
  
Instances of the above types can be used as definite types. For example, one can write:

.. code-block:: ada

      X : T;
   begin   
      X := (True, True);
      X := (True, True, False);
     
Notice that not only X can be declared without bounds, those bounds can change in the course of its lifetime without a check failure.
The above requires a default value to be provided for this type (alongside of all types that can have a Max_Size aspect). In the case
of arrays, it would be an empty array, so that:

.. code-block:: ada

      X : T;
   begin
      Pragma Assert (X’Length == 0);
      
Is true. Dynamic checks are inserted when needed to verify that the size of the value fits within type constraints.

Max_Size on Tagged Types
------------------------

When applied on a tagged type, Max_Size denotes the maximum size of this type as well as all of its descendants. The max size can be 
further reduced by child, but never extended. This can also be set on an interface. In this case, when implementing multiple
interfaces, the smallest Max_Size is taken. So that:

.. code-block:: ada

   type I1 is interface with Max_Size => 8 * 64;
   type I2 is interface with Max_Size => 8 * 16;

   type R is new I1 and I2 with record
      V : Integer;
   end record  -- max is implicitly 6 * 16 here ;

   type C1 is new R with record
      W : Integer;
   end record with Max_Size => 8 * 16;
   
   type C2 is new R with record
      X, Y, Z, A, B, C, D, E : Integer;
   end record; -- not ok, size too big
   
As soon as a type is either derived from a type that has a Max_Size aspect, or has itself a Max_Size aspect, it becomes possible to
write:

.. code-block:: ada

      X : I1’Class;
   begin
      X := R’(V => 10);
      X := C’(V => 10, W => 20);
     
As before, we need a special default initialization here. There’s not really such a thing for class-wide access type, we could create
a special null tag value to identify those.

Max_Size on Private Types
-------------------------

Max_Size can be defined on a private type, e.g.:

.. code-block:: ada

   type T1 (<>) is private with Max_Size => 8 * 64;
   type T2 (D : Integer) is private with Max_Size => 8 * 64;

As before, the above allows to write things such as:

.. code-block:: ada

  V : T1;
  W : T2;

Note that having a max size automatically makes record with discriminants “mutable” - ie the value of the discriminant can change if
they are assigned to a value.

Max_Size on Definite Types
--------------------------

For the sake of completeness, Max_Size can also be provided on a definite type, for example:


.. code-block:: ada

   type T is new Integer with Max_Size => 8 * 4;

In that case, the only thing that the compiler will do is to check that indeed the type fits in its maximum size constraint, but the
usage of the type will be the same as otherwise.

Max_Size on Variables
---------------------

Even if the type is indefinite, it’s possible to specify a Max_Size attribute directly on the object itself. For example:


.. code-block:: ada

      V : String with Max_Size => 8 * 10;
   begin
      V := “abc”;
      V := “defg”;

Is perfectly appropriate. A dynamic check verifies that each assignment is done with a value of proper size. This would work with
indefinite and class wide types as well, for example:


.. code-block:: ada

      type I is interface;
      type R is new I record
         V : Integer;
      end record;

      V : I’Class with Max_Size => 8 * 16;
   begin
      V := R’(V => 10);
      
Note that in order for the above case to be statically checkable, the compiler need to have some way to dynamically know the size of a
type, which may lead to additional code generation (e.g. size depending on the length of an array, tag of an object, discriminant of a 
record, etc.).

Also note that Max_Size on object can further reduce the size of a type which already have a Max_Size clause associated with it:


.. code-block:: ada

   type T is array (Integer range <>) of Boolean with Max_Size => 8 * 64;
   X : T with Max_Size => 8 * 16;
   Y : T with Max_Size => 8 * 32;
  
If the value is larger, it will just be ignored (e.g. the smallest of the two is used).

Max_Size on Fields
------------------

Similarly to variables, it’s possible to use Max_Size clause on fields, so that the following is legal:

.. code-block:: ada

   type T_I (<>) is private;
   type R is record
       V : String with Max_Size => 8 * 10;
       V2 : I’Class with Max_Size => 8 * 16;
       V3 : T_I with Max_Size => 8 * 16;
   end record;
   
The clear benefit is that it’s now possible to create data structures with indefinite fields avoiding usage of dynamic memory. This is 
the only way V2 or V3 could be created. Arguably, declaration of V this way is simpler than the notation with the discriminants.

This is not creating a variant record. The compiler is responsible for creating a layout that allocate enough memory for each field,
so that the user can write something like:


.. code-block:: ada

     L : R;
  begin
     L.V := “abc”;
     L.V := “abcd”;
    
Max_Size on Array Components
----------------------------

There’s a subtlety on array types, as when you write:

.. code-block:: ada
  
  type T is array (Integer range <>) of Boolean with Max_Size => 8 * 64;

The Max_Size applies to the size of the array as a whole, not its individual components. You may however want to constraint the size of
the components themselves. This can be done through the Max_Component_Size attribute, e.g.:

.. code-block:: ada

   type T is array (Integer range <>) of String with Max_Component_Size => 8 * 10;

In which case the compiler will reserve the right amount of memory for each component. The two clauses can be used together, for example:

.. code-block:: ada

  type T is array (Integer range <>) of String with 
     Max_Component_Size => 8 * 10, 
     Max_Size => 8 * 80;
    
Max_Size on Parameters and Returned Value
-----------------------------------------

Parameters and return values can already be indefinite. However, putting a Max_Size constraints on them will generate a check at call
time, and a guarantee within the procedure of a given size:


.. code-block:: ada

  function F (V : String with Max_Size => 8 * 10) return String with Max_Size => 8 * 10;

Max_Size on Generic Parameters
------------------------------

Similar to functions, it’s possible to specify maximum size in generic parameters:

.. code-block:: ada

  generic
     type T (<>) is private with Max_Size => 8 * 16;
     type R is tagged private with Max_Size => 8 * 16;
  package P is
     type R is record  
        V1 : T;
	V2 : R;
     end record;
  end P;
  
The constraint is that the formal parameter must guarantee the max_size aspect, either because they have a max_size clause equal or
lower themselves, or because they’re definite.

Reference-level explanation
===========================

Rationale and alternatives
==========================

A more ambitious proposal could have been to allow a record to be of a given Max_Size, spreading the size amongst components, for 
example:

.. code-block:: ada

   type R is record
   	V1, V2, V3 : String;
   end record with Max_Size =>100;

However, this means that V2 and V3 positions are dependent on V1, V3 dependent on V2. Access require either a table of offset to be
available in R (memory footprint) or to compute dynamically the value for accesses (performance footprint). On top of that, it’s not
clear what changing the boundaries of one field does to the others, if allowed that would require copies of fields below, if forbidden 
that means that boundaries can only be changed when modifying the record as a whole.

From a syntactical point of view, we could envision expanding on the <> notation, for example allowing a number in the box for the max
size. You could have e.g.:

.. code-block:: ada
   
   type I <8 * 16> is interface;

To denote an interface of 16 bytes at most. However, this create arguably less readable syntax - the aspect is much more readable.

Drawbacks
==========

Prior art
=========

An Ada 2012 AI introduced Bounded_Indefinite_Holder to solve a similar problem. It has several limitations over the Max_Size model.
The main drawback is that it can’t be used to describe constraints at type level. The usage of a tagged type around the holder imposes 
a footprint in terms of data that may be too much of a penalty for the memory constraints of some of the applications that would 
benefit the most of such feature. As a side effect, this feature usage is also much more compact for the cases that were already 
supported by Bounded_Indefinite_Holder. As a simple comparison, a program with two strings an assignment, for example:

.. code-block:: ada

   with Ada.Containers.Bounded_Indefinite_Holder;
   procedure Main is
      package String_Holder is new Ada.Containers.Bounded_Indefinite_Holder (String, 8 * 16);
      V1 V2 : String_Holder.Holder;
   begin
      V1 := To_Holder (“abc”);
      V2 := To_Holder (“abcd”);
      Replace_Element (V1, Element (V2));
   end;

becomes:

.. code-block:: ada

   procedure Main is
      V1 V2 : String with Max_Size => 8 * 16;
   begin
      V1 := “abc”;
      V2 := “abcd”;
      V1 := V2;
   end;
   
Unresolved questions
====================

Future possibilities
====================

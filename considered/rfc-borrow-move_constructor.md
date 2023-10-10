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

Move semantics can significantly improve object-oriented design. When combined
with new constructors and assignment primitives from the OOP revamp proposal,
limited references can provide an efficient way to manage objects without
unnecessary copying

Copy and assignment primitives can now be provided with limited parameters.
Consider the following class:

```Ada
   type Holder is class record
      procedure Holder (Self : in out Holder; Source : T1);
      -- Regular by copy constructor

      procedure Holder (Self : in out Holder; Source : limited T1);
      -- Move constructor

      procedure ":=" (Self : in out T1; Source : T1);
      --  Regular assignment

      procedure ":=" (Self : in out T1; Source : limited T1);
      --  By copy assignment

      Size : Integer;
      Contents : Array_Access;
   end Holder;

   type Holder is class record

      ...

      procedure Holder (Self : in out Holder; Source : limited T1) is
      begin
         Self.Size := Source.Size;
         Self.Contents := Source.Content;

         Source.Size := 0;
         Source.Contents := null;
      end Holder;

      procedure ":=" (Self : in out T1; Source : limited T1);
      begin
         Free (Self.Contents);

         Self.Size := Source.Size;
         Self.Contents := Source.Content;

         Source.Size := 0;
         Source.Contents := null;
      end ":=";

  end Holder;
```

We can now write the following without copy:

```Ada

   function Create_Holder (Size : Integer := 0) is
   begin
      return Holder (Size);
   end Create_Holder;

   V : limited Holder;

begin

   V := Create_Holder (5); -- no need for any copy here

```

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

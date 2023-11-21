- Feature Name: (fill me in with a unique ident, my_awesome_feature)
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- RFC Issue: (leave this empty)

Summary
=======

This proposal introduces two concepts:

- A new pragma Ghost_Scope which allows to name a specific ghost code
  classification.
- An extension for the syntax of all ghost code (preconditions,
  postconditions, assertions...) that allows to designate ghost code expression
  as being related to a specific classification.

Tools such as compilers, provers or static analysers will be able to be
configured to enable or not certain ghost code classes.

Motivation
==========

When using programming by contract extensively, it is often necessary to
classify ghost code for different purposes, and have this classification
impacting run-time code generation. A very common case is to differenciate
ghost code that can be activated at run-time and generate tests from ghost
code that cannot be possibly compiled (because it's too slow, needs too much
memory or references entities that have no body) but is nonetheless useful
for the purpose of proof.

Guide-level explanation
=======================

Multiple Ghost scopes
---------------------

A new pragma is introduced, ``Ghost_Scope``, which takes an identifier as
parameter:

```Ada
package Some_Package is
   pragma Ghost_Scope (Silver);
   pragma Ghost_Scope (Gold);
   pragma Ghost_Scope (Platinium);
end Some_Package;
```

A new package is introduced to the Ada standard, Ada.Ghost, defined as follows:

```Ada
package Ada.Ghost is
   pragma Ghost_Scope (Default);
   pragma Ghost_Scope (Always_Runtime);
   pragma Ghost_Scope (Never_Runtime);
   pragma Ghost_Scope (Performance);
   pragma Ghost_Scope (Safety);
end Ada.Ghost;
```

The Default ghost scope is the one used in the absence of specific
parametrization of assertions. Code marked "Always_Runtime" should always be
executed under all compiler settings. Code marked "Never_Runtime" should never
be executed under any compiler setting. Other scopes are provided as a way to
standardize the most common execution modes.

Ghost identifiers are scoped like regular variables. They can be prefixed by
their containing package. They need to be declared at the library level.

Ghost on Assertions
-------------------

Assertions can be associated with specific scopes using the Ada arrow
assocations. This can be used in pragma Assert, Assume, Loop_Invariant, e.g.:

```Ada
pragma Assert (Gold => X > 5);
```

or in aspects Pre, Post, Predicate, Invariant, e.g.:

```Ada
procedure Sort (A : in out Some_Array)
   with Post =>
      (Gold      => (if A'Length > 0 then A (A'First) <= A (A'Last)),
       Platinium => (for all I in A'First .. A'Last -1 =>
                     A (I) <= A (I-1)));
```

```Ada
procedure Sort (A : in out Some_Array)
   with Contract_Case =>
      (Gold      =>
          (Something      => (if A'Length > 0 then A (A'First) <= A (A'Last))
           Something_Else => Bla),
       Platinium => (for all I in A'First .. A'Last -1 =>
                     A (I) <= A (I-1)));
```

A given assertion can be provided for multiple ghost scopes, for example:

```Ada
pragma Assert ([Gold, Platinium] => X > 5);
```

Ghost on Entities
-----------------

Entities can be associated with specific scopes of ghost code. When that's the
case, the scope is given as a parameter of the Ghost argument. A given entity
can be associated with more than one ghost scope. For example:

```Ada
V : Integer with Ghost (Platinium);

procedure Lemma with Ghost ([Never_Runtime, Platinium]);
```

Dependencies between Ghost code
-------------------------------

When declaring ghost scopes, you can describe dependencies, in other word, what
data can flow from one kind of ghost to the other. This dependency is
unidirectional. In the current SPARK behavior, the default ghost code is
already expressed as being able to depend on runtime code:

```Ada
pragma Ghost_Scope (Default, Depends => [Always_Runtime]);
```

This effectively prevents run-time code to depend on ghost code (indeed, ghost
code may not be compiled).

Ghost code that can never be executed at run-time may also depend and runtime
code. In addition, it may also depend on Default ghost code. The reverse is
not true as Default ghost may be executed but not Never runtime:

```Ada
pragma Ghost_Scope (Never_Runtime, Depends => [Default, Always_Runtime]);
```

Note that dependencies are transitive, so the above is actually written:

```Ada
pragma Ghost_Scope (Never_Runtime, Depends => [Default]);
```

Always_Runtime ghost code can't depend on any other kind of ghost code, which
can be expressed by an empty value:

```Ada
pragma Ghost_Scope (Always_Runtime, Depends => []);
```

Users would also be able to decribe their own dependencies. A typical use
case for someone that goes through the Silver / Gold / Platinium nomenclatura,
and wishes to differenciates code that can be activated at runtime from
code that can't would be to do as follows:

```Ada
pragma Ghost_Scope (Silver,   Depends => [Default]);
pragma Ghost_Scope (Gold,     Depends => [Silver]);
pragma Ghost_Scope (Platinum, Depends => [Gold]);

pragma Ghost_Scope (Silver_No_Runtime,   Depends => [Silver, Never_Runtime]);
pragma Ghost_Scope (Gold_No_Runtime,     Depends => [Silver_No_Runtime, Gold]);
pragma Ghost_Scope (Platinum_No_Runtime, Depends => [Gold_No_Runtime, Platinum]);
```

Activating / Deactivating Ghost code
------------------------------------

Specific ghost code can be activated / deactivated through the Assertion_Policy
pragma:

```Ada
pragma Assertion_Policy (Gold => Check, Platinium => Disable);
```

Compiler and prover options may also have an impact on the default policies.

Note that activating or deactivating ghost scopes have an impact on dependent
ghost scopes as follows:
- Deactivating one ghost scope will deactivate all ghost scopes that are
  allowed to depend on it.
- Activating one ghost scope will activate all ghost scopes that it depends on.

Activation / deactivation can be done either from the perspective of run-time
checks or proofs.

Reference-level explanation
===========================

See above

Rationale and alternatives
==========================

TBD

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

We could envision to activate / deactivate ghost code across an entire call
graph, introducing a new pragma ``Deep_Assertion_Policy``. For example:

```Ada
   if Urgency = Low then
      pragma Deep_Assertion_Policy (Safety => Enabled, Performance => Disabled);
      Call_Something; -- all Safety is disabled here
   else
      pragma Deep_Assertion_Policy (Safety => Disabled, Performance => Enabled);
      Call_Something; -- all Safety is disabled here
   end if;

   -- Go to whatever state we had before the if.
```

Or through a different example:

```Ada
   begin
      pragma Deep_Assertion_Policy (Safety => Enabled);
      Call_Something; -- all Safety is disabled here at that level and below
   exception when others =>
      pragma Deep_Assertion_Policy (Safety => Disabled);
      Log_Safey_Error;
      Call_Something; -- try again without safety, but then treat the result
      --  with caution
   end;

   -- Go to whatever state we had before the if.
```

This could help model various execution modes in a high integrity application,
allowing to implement degraded safety mode when needed. Knowning wether this
kind of advanced capabilty is valuable would require some industrial feedback.

Another idea that came up is to define relationships between scopes. This could
be used when using them in ghost code, e.g.:

```Ada
   procedure Sort (A : in out Some_Array)
      with Post =>
         (Gold               => (if A'Length > 0 then A (A'First) <= A (A'Last)),
          Gold or Platinium => (for all I in A'First .. A'Last -1 =>
                                 A (I) <= A (I-1)>));
```

Or even when defining these scopes:

```Ada
   pragma Ghost_Scope (Platinium, Implies => Gold and Silver);
   --  Platinium always activates Gold and Silver
```

Again, it's not clear yet if the extra complexity bring sufficient value.

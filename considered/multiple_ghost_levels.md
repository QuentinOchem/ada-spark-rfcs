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
                        A (I) <= A (I-1)>));
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

   procedure Lemma with Ghost (Never_Runtime);
```

Activation and Consistency of Ghost code
----------------------------------------

Ghost scopes can be controlled at declaration time through the Runtime_Check
and Proof_Check values:

- Runtime_Check => Always, the check is always enabled at run-time. This can't
  be overriden by other pragmas or compiler configurations
- Runtime_Check => Never, the check is never enabled at run-time. This can't
  be overriden by other pragmas or compiler configurations
- Runtime_Check => On, the check is enabled at runtime by default, can be
  changed.
- Runtime_Check => On, the check is not enabled at runtime by default, can be
  changed. This is the default mode for all ghost code.

- Proof_Check => Always, the check is always checked by the prover. This can't
  be overriden by other pragmas or compiler configurations
- Proof_Check => Never, the check is never checked by the prover. This can't
  be overriden by other pragmas or compiler configurations
- Proof_Check => On, the check is checked by the prover by default, can be
  changed. This is the default mode for all ghost code.
- Proof_Check => On, the check is not checked by the prover by default, can be
  changed. This is the default mode for all ghost code.

- Proof_Assume => Always, the check is always assumed true by the prover.
  This can't be overriden by other pragmas or compiler configurations
- Proof_Assume => Never, the check is never assumed true by the prover.
  This can't be overriden by other pragmas or compiler configurations
- Proof_Assume => On, the check is assumed by the prover by default, can be
  changed. This is the default mode for all ghost code.
- Proof_Assume => On, the check is not assumed by the prover by default, can be
  changed. This is the default mode for all ghost code.

For example:

```Ada
package Ada.Ghost is
   pragma Ghost_Scope (Default, Runtime_Check => Off);
   pragma Ghost_Scope (Always_Runtime, Runtime_Check => Always);
   pragma Ghost_Scope (Never_Runtime, Runtime_Check => Never);
   pragma Ghost_Scope (Performance, Runtime_Check => Off);
   pragma Ghost_Scope (Safety, Runtime_Check => On);
end Ada.Ghost;
```

Policy can also be changed for attributes that are not marked Always or Never,
through the use of the pragma Ghost_Policy. A policy is active until the end of
the scope or until another Ghost_Policy pragma is found. For example:

```Ada

   pragma Ghost_Policy (Default, Runtime_Check => On);

   -- Activates run-time checks

   pragma Ghost_Policy (Default, Runtime_Check => Off);

   -- Deactives run-time checks
```

Turning On and Off Proof_Check may allow to run proof in different context,
for example, local users may be required to only activate certain proof while
a CI/CD integration system may need to activate more.

Ghost checks (that is, expressions within an assertion) must be consistent with
regards to their Runtime_Check and Proof_Check properties. The compiler is responsible for identifying when an assertion that needs to be proven and executed has
elements in its expressions that are marked as not checked for proof and run-time
check respectively. For example:

```Ada
   V : Integer with Ghost (Runtime_Check => Never);

   pragma Assert (V = 0);
   --  Compiler error, the assertion is marked default and can be activated for
   --  run-time checks while V cannot.
```

Within a ghost entity however - e.g. a lemma - inconsistenc

```Ada
   procedure L1 with Ghost (Never_Runtime);

   procedure L2 with Ghost (Always_Runtime);
   -- L2 will always be called at runtime

   procedure L2 with Ghost (Always_Runtime) is
   begin
      L1;
      -- OK - even if L2 is always called at run-time, L1 will not.
   end;
```

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

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

A new pragma is introduced, ``Ghost_Scope``, which takes an identifier as
parameter:

```Ada
package Some_Package is
   pragma Ghost_Scope (Silver);
   pragma Ghost_Scope (Gold);
   pragma Ghost_Scope (Platinium);
end Some_Package;
```

Assertions can be associated with specific scopes using the Ada arrow
assocations:

```Ada
   procedure Sort (A : in out Some_Array)
      with Post =>
         (Gold      => (if A'Length > 0 then A (A'First) <= A (A'Last)),
          Platinium => (for all I in A'First .. A'Last -1 =>
                        A (I) <= A (I-1)>));
```

Ghost identifiers are scoped like regular variables. They can be prefixed by
their containing package. They need to be declared at the library level.

The pragma ``Assertion_Policy`` can be parametrized with ghost scopes:

```Ada
   pragma Assertion_Policy (Gold => Check, Platinium => Disable);
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

# Error messages

Florian: we added more explanations to error messages for functors
applications, module signatures, and records. But now the error
message get very verbose, in particular you can get a quadratic blowup
on module signatures.


# Extending levels to HO polymorphism

Thomas: I'm proposing to extend the handling of levels in the type
system to also handle binders at arbitrary positions in types.

Didier: I think this is going in the right direction. But then we
should think very carefully about how we count things and what we need
from our levels. I would have liked to see a very detailed description
of how they work.

Now in OCaml we have System F and equi-recursive types. It is known to
be difficult to check, and I'm not sure we do it properly. We are
going in the right direction but it may be too early, as we haven't
formalized what the new levels would do.

What we do with levels is like De Bruijn levels. I would like to see
this correspondence.

Leo: the algorithms that you need for the typechecker are in this
document. You are asking for a more formal presentation. It would be
great to have that, but it should not lock work, we already don't have
that for the rest of the system.

Thomas: difference between implicit and explicit. How to formalize that?

Gabriel: high-level questions
- is Jacques convinced?
- does Thomas have an implementation?

Thomas: not finished yet; my second attempt is going to be much smoother.

Jacques: I was never opposed to the proposal, I had some
commnts/details but the ideas sound truly great. The problems we have
with the current representation, there is a comment in ctype saying
"this is way too inefficient", maybe we should get rid of it.

Florian: I think it would be quite nice to reduce the number of
mechanisms to type-check in the type-checker.

Leo: it also simplifies the modular-{implicit/explicit} implementation.

Didier: I would like to know what's the algorithm that is used now to
check the equality of recursive types.

TODO: a "how OCaml checks equality of types" submeeting.


# Making stuff more abstract

Jacques: row_field can be extended, but we don't want to give the user
the direct access to the reference, because they are not supposed to
modify it directly.

The result is an API ....

Gabriel: question, why do you have both the _view as a nice sum type,
and the match_row_field encoding of pattern-matching?

Jacques: match_row_field is only used in printtyp, it exposes the
low-level information directly.

Jacques: The other PR is doing the same for objects: field_kinds etc.

Gabriel: high-level question, how are things going on the general project with Takafumi?

Jacques: I think it's going well. We have other plan for abstraction
boundaries, in typecore.ml and also below ctype.ml.

Florian: I'm volunteering to review the variant and objects PRs.



# Rewriting Ctype.subst

There is mutual recursion between expansion and unification. Everytime
you expand, you perform unification on its parmaeters. It's morally
ok, but given that unification is a complex beast in OCaml, mixing the
two is really complicated. I think we can relatively easily modify
Ctype.subst, the function doing the substitution step of expansion, so
that it uses matching rather than unification. (When matching, you
recurse on the pattern and try to extract information from the
subject.) When doing expansion you already know that the subject
matches the constraint...

Leo: not always! Unification can fail inside expansion in some places.

Jacques: once you are defining a type, you can fail. But once you have
defined the type, failure is not possible.

Leo: how do you deal with the case where you are still defining the types?

I need to experiment more with the recursive-type case.

Leo: we recently made the rules more restrictive about recursion
inside constraints. It's possible that it's simpler now.

Stephen: I think this change could also speed things up quite
a lot. I've been profiling the work quite a lot, a lot of the work is
expansion, and a lot of the expansion work is unification.


# Performance of the type-checker?

Gabriel: Florian built a prototype to measure the build time of
several packages, and in particular the type-checking time.

Florian: people at ocamllabs have been thinking about doing continuous
testing of the compiler performance.

Stephen: my module-system-speedup work was tested against the Jane
Street codebase, but the wins were very obvious.

Stephen's module-system performance PR:
  https://github.com/ocaml/ocaml/pull/10599

Stephen: it's not obvious what costs in the type-checker. Expanding
('a, 'b) t to ('a, 'b) t' can take a lot of time, because module
strenghtening will do this a lot. Module inclusion operations can be
quite expensive, even when they spend most of their time checking
a module signature against itself for slightly different paths.

Leo: ocamlc at Jane Street spends the vast majority of its time
checking a large signature against itself.

Stephen:
  struct module M = Map.Make(X) end
  : sig moduel M : Map.S with type key = X.t end



# Gabriel on constructor unboxing

Gabriel: nicer than fuel conceptually, but also more complex.

Gabriel: two places where 

Consensus: it's okay to go with this slightly more complex approach, if the performance is good.


## jsoo support

Leo: maybe we could unbox nothing in bytecode.

Gabriel: FFI issues, then bytecode and native have different interfaces.

Stephen: is it only about float, or other types more people care about?

Xavier:
- we want bytecode and native to use the same representation
- we can probably reason about just ocamlc or js-of-ocaml.

Leo: I would assume the problem is "float vs. immediate value".

Xavier: we could have a flag that says "I will not compile to javascript".

Consensus: just aim for the conservative version for now, we can add
a flag/setting later.



# Transparent ascriptions

## Aliasing functor applications

Leo: the problem is the "present/absent" bit on aliases: functor
applications have to be present.

Consensus: have explicit syntax to request an absent alias, but we
also get to forbid them from outside toplevel definitions
(inside modules/functors).

## Transparent ascriptions

module M = (N :> S)

Drup: by not having transparent ascriptions, we:
- use "module type of" too much
- lose equalities between modules,
  which is problematic for modular-{ex,im}plicits

Drup: should we merge Matthew's PR?

Leo: I haven't looked at the code yet, I'm focusing on getting Matthew
to implement the right specification first.

The syntax needs to be (M <: S), not (M :> S).

Consensus:
1. Drup clarifies his specification for transparent ascription,
   gets a caml-devel review
2. Leo and Drup review Matthew's implementation, once the semantics has converged
3. if (1) and (2) pass, the PR is merged.


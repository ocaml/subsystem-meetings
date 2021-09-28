Extending levels to HO polymorphism
-----------------------------------

A group of people (Didier, Gabriel, Jacques, Leo, Matthew and Thomas) recently
started looking into extending OCaml's level mechanism to handle binders in
arbitrary positions inside type expressions.
This would of course be helpful for polymorphic methods (and record fields), but
also later (but hopefully quite soon) for modular explicits.

Note: this document, while trying to give some context for people who didn't
participate in earlier discussions, discuss rather low-level / implementation
details and assumes that the reader is somewhat familier with the typecheckers
implementation.

## Levels: a refresher

*Disclaimer:* a proper introduction to levels is available at
<http://okmij.org/ftp/ML/generalization.html>.
What follows is a short description of my (Thomas) intuitions regarding levels.

For me the key observation is that the typing environment is ordered (by time of
insertion): types can only refer to types that were in the environment before
them. If we annotate types with their position in the environment (and call that
their level), we can then use these annotations to (relatively) efficiently
implement operations such as generalisation and instantiation, or checks to
prevent scope escapes.

Going further, we can annotate all type nodes, not only those of variables (and
type constructors).
You can view the level of a structure node `N` as the position at which a
declaration `α = N[...]` is added to the environment (for some free α). Since
types can only refer to types that appear before them, we have the following
nice (and simple) invariant:

> levels in a type are decreasing (as we progress down the structure).

Variables bound in a type are not in the environment, so they don't have a level
in the "naïve" sense I just gave (i.e. they don't have a position in the
environment). However we can (and do) mark them with a distinguished level
(`generic_level` in the implementation, but following Oleg's suggestion, let's
call it ω). This can be understood as the encoding/embedding of a sum type
(where ω simply marks things that are not in the environment).

All of this taken together allows us to implement the operations mentioned above
more efficiently.

### HO polymorphism

However, what I just described does not help with higher order polymorphism. For
instance, in `val foo: α * < x: β. α -> β -> α >` everything (variables as well
as the structure) will be at level ω.
Which is problematic: `β` is bound deeper than `α`, so we'd expect it to stay
"generic" longer.

The current implementation in OCaml doesn't rely on levels to manipulate types
with explicit binders. It maintains the decreasing-level invariant on types, but
there's no implied meaning for the bound variables.

## Extending levels for handling deep binders

Informally, I like to think about bound variables as variables that will be
added in the environment at a later point. Hence their level being greater than
the current level.
Since binders can be arbitrarily nested, not all bound variables are going to be
added to the environment at the same time (at the same level), so a natural
extension of the system would be to have several "generic levels": instead of
stamp that only indicates that the variable will enter the environment later on,
we can specify how much later it's going to be added. That is: how many other
bindings we need to "open" before this variable can be added to the environment.

So on the example above, we'd assign the level ω to `α` and `ω+1` to `β`.  And
since we're still enforcing  the invariant that types can only refer to things
that appear before them in the environment, it follows that the structural nodes
of the type will need to be added to the environment at the same time (or later)
than the bound variables. So we can deduce their level (annotated in-between
brackets):

        val foo: α[ω] *[ω] (∀ β. α[ω] ->[ω+1] β[ω+1] ->[ω+1] α[ω])[ω]

Let's call _spine_ all the nodes between a binder (a `Tpoly` node in the
implementation) and the variables it binds. On the example above, the spine
consists of those nodes that received the level `ω+1`.

You should consider the spine as an atomic/unbreakable construction: it doesn't
make sense to share part of a spine with the outer environment, one needs to go
through the binder to access it in a meaningful way.

N.B. level ω is reserved for nodes which are implicitely quantified, nodes under
an explicit binder (i.e. spine nodes) get levels ω+k (for some k > 0).

### Pseudo-code version of basic operations on types

#### Generalization

Generalization and "abstraction" (i.e. placing some nodes under a binder, making
them unusable until the binder is "opened") are two distinct operations.

Generalization is mostly unchanged, except that it now traverses some nodes
(spine nodes) without touching them, so an explicit marking (and unmarking) is
required:

```
gen_mark(n, ty):
  - ty.level <= n || ty.level = ω: stop
  - ω < ty.level:
      mark; // necessary in case of loops under explicit binders
      recurse
  - n < ty.level < ω:
      ty.level <- ω;
      mark;
      recurse

generalize(n, ty):
  gen_mark(n, ty);
  unmark(ty)
```

Once a type has been generalized, you can abstract the generic variables:

```
abstr_mark(ty):
  - ty.level < ω: stop
  - ty.level >= ω:
      ty.level++
      mark
      recurse

abstract(ty):
  abstr_mark(ty);
  unmark(ty)
  vars = gather Tvars at ω+1; // optionaly turn them into Tunivars
  return Tpoly (ty, vars)
```

Note that in the implementation we could also provide a
`generalize_and_abstract` function which would do both operations in one
traversal.

#### Instance

Similarly, we need an operation to take an instance of a generic type, and one
to open a binder.

`instance` behaves as it always did: keep the sharing on types which are already
in the context, and getting fresh copies of types which aren't yet.

```
instance(ty, n):
  - ty.level = ω:
      recurse on the children
      rebuild the node at level n using instantiated children
      return it
  - ty.level < ω
      return ty
  - ty.level > ω:
      recurse on the children
      rebuild the node at level ty.level using instantiated children
      return it
```

As for the opening operation, notice that it doesn't take an explicit list of
variables anymore, and is much simpler than the current implementation in the
compiler (`instance_poly`):

```
instance_poly(ty):
  - ty.level <= ω:
      return ty
  - ty.level > ω:
      recurse on the children
      rebuild the node at (ty.level - 1) using instantiated children
      return it
```

Here too, we could make a function which opens and instantiate (which is what
`instance_poly` currently does).

#### Update level

This one is used to move a type higher in the environment (which happens for
instance during unification).
However, there's some subtlety: it can be used to move types from a level
strictly higher than ω to some level below ω.

This can happen because our `generalize_and_abstract` function is a bit too
blunt, it will move more things than strictly necessary to ω+1. For instance in
`∀α. α * foo`, `foo` is not on the path to `α`, so not really on the spine,
which means it doesn't need to be at the level ω+1. But with our implementation,
it probably will be.
Yet, we want to be able to unify it with some other type already present in the
environment, so we allow `update_level` to bring it in scope.

While changing `foo`'s level is allowed, it would be incorrect to allow it on
spine nodes. This restriction is enforced by the recursive nature of the
operation: nodes which are on a spine have a bound variable as descendant, and
bound variables are rigid so (though the details are omitted in the pseudo-code)
we don't allow changing their level.

There is one final thing to note here: if the node which is being moved is a
Tpoly, we will need to shift the level of its spine. This is what makes the
operation more complicated than it previously was.

```
update_level_mark(ty, n, shift):
  - ty.level <= n: stop
  - shift = Some k && ty.level > ω+k:
      // N.B. ty.scope also gets shifted at this point!
      ty.level -= k
      mark
      recurse
  - otherwise:
      shift :=
        if ty = ∀:
          ty_binding_depth = max(0, ty.level - ω)
          n_binding_depth  = max(0, n - ω)
          Some (ty_binding_depth - n_binding_depth)
        else
          None
      ty.level <- n
      mark
      recurse with new shift

update_level(ty, n):
  update_level_mark(ty, n, None)
  unmark(ty)
```

### Invariants on types and spine

What follows is an extract (from an email discussion) which lists the invariants
of the current system:

> Currently, there are very few [invariants used in the representation of types],
> at least if we ignore temporary states.
> Namely, in live nodes
>
> - level should go down in children
> - scope should go up in children
> - level must not be lower than scope
> - for polymorphic variants, types sharing the same row variable must be
>   equivalent: i.e., they must develop as the same tree, and share the same
>   variables and Reither extensions
> - all Tunivars bound by a Tpoly must be distinct
> - free Tunivars are restricted to those in the matching stack, and they cannot
>   appear in types unified to possibly external unification variables
>
> In particular, there are currently no restrictions on sharing, what matters is
> essentially the semantics of types as regular trees.
>
> These invariants may be broken temporarily
>
> - during generalization
> - during copying
> - in operations that use negative levels as marks
>
> Types of already typed subexpressions are not seen as live after
> generalization in their parents; one has to use the function `correct_levels`
> to create a copy that respects these invariants.

With the proposed extension, these invariants are kept. Remember: we see a
spine as an atomic construction (its level is that of its binder)!  
To which we add the following invariants:

- the level of nodes in a spine should be greater than ω
- the level of _nested spines_ should be strictly increasing
- variables bound by a binder must be on that binder spine <!-- though that's
  true by definition -->

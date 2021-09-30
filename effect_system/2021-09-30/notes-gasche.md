# Current state


KC: currently we support untyped effect handlers. There is no typing,
this is as bad as exceptions. This is useful for encoding all of our
simple examples, but if you look at the literature, when people
introduce effect handlers, they introduced it with an effect
system. That was always the plan, but we were focusing on having the
runtime work first, before moving into the effect system. The effect
system is also a moving target, Leo has been working on that for
years. We decided last time to support just Fibers for 5.0, all
through Obj functions, with no new syntax at all.

If you look at the current PR that was merged a few weeks ago:
  https://github.com/ocaml-multicore/ocaml-multicore/pull/651
we expose both deep and shallow handlers. (Deep handlers are folds,
shallow handlers are case matches.) The implementation is a little bit
optimized towards deep handlers. The usual takeaway is that it's
easier to encode deep from shallow than the other way around. We
support both features in 5.0; we could optimize shallow handlers
better. The majority of the implementation stays the same.

The idea is that we shouldn't ever have both untyped effects with
syntax, and the typed effects. So we only expose Obj-level function
without syntax.

Leo: they could go in their own submodule rather than Obj.


Implementation:
  https://github.com/ocaml-multicore/ocaml-multicore/pull/651/files#diff-3ab5041c2947a7b5862990c522329c8f1be5ded4501cded7d7d12ce67eb29b53R200
 
Interface:
  https://github.com/ocaml-multicore/ocaml-multicore/pull/651/files#diff-dad5a1dd42d5707fd9801bd1657263f15021d1f9ce9f2e2bd9c67a1b5fe08725R198


Gabriel: the interface looks type-safe and memory-safe, is that correct?
KC: yes, it is.


Florian: we could an "experimental" alert to tag these features as
experimental, a mirror of the deprecation alert.

François: the mlis are not so documented, is there a plan to flesh it out?
KC: right now to follow along you should look at examples of encodings. We could add that to the documentation.



# Multishot

KC: currently you have to explicitly discontinue continuations, our
stacks are malloced and if you don't discontinue them you are leaking
memory.

KC: there may be legacy code that does resource cleanup in a correct
way, but dropping continuations would leak.

Explicitly discontinuing a continuation will "throw an effect" into
the continuation, and in particular run resource-cleanup code from
Fun.protect etc.

Xavier: trying to track dead continuations would be some amount of
work. We don't do that for threads.

Damien: it's worse than that, if you let the GC detect unused
continuations it will have to call the cleanup code.

Stephen: there was an original version of Multicore that would use
finalizers. It more or less worked, but the semantics are not what you
want. When someone writes Fun.protect, they expect deterministic
cleanup; they don't want to wait for the GC.


Didier: some people may want to experiment with multishot
continuations. Could they do it now, or could they do it on top of
existing continuations? Could the runtime provide low-level support,
where people are on their own?

Leo: if you add a duplication function for continuations, it's
impossible to use correctly and safely. (Optimizing local "let" into
stack variables is broken.) I think this is much better done the Oleg
way, by writing low-level C code.

Stephen: the semantics issue is, there is no way in general how much
you want to copy.

Xavier: let's keep that for future experimentations, but kind of out
the original design.


# Type system

Leo: we have operational semantics. The interesting part is the
handling of names.

François: Multicore OCaml has dynamic name generation, right?
(Local declaration of operations.)
Do we agree that we have the right semantics in the implementation at the moment?

<complex discussion without much context>

Leo starts reading the slide.

Stephen on name representation: you have a pair of a name as just
a string, and then a De Bruijn index to disambiguate between different
binding sites of the *same* name.

Gabriel: here it looks like it's the same syntactic construct that
handles operations and reinstates continuations.

So 
  (match e with n#op, k -> ...)
would be encoded as
  (continue (continuation (fun () -> e)) () with n#op, k -> ...)
?

Leo: so here our operations are of the form n#op, where n is a name.

Gabriel: to encode exceptions here, would we have a single name?

Leo: [..] (unclear)


Didier: names pre-exist, if you reuse a name, you push other similar
names one level. You don't have a feature to create a fresh name.


Didier: so any time you call find, each call on `p` has an extra level
of try..with, this is expensive.

Leo: this will only cost if it was needed.

Gabriel: so the renaming here lift/shifts effect: if it's not there,
a Not_found raised from `p x` will be handler by the caller of
`find`. If it is here, it will traverse the handler of `find` and be
raised to the user.


Leo: existing proposals are playing with operations that are very
clearly structural rules.

Leo: if you don't separate the names from the effects, you lose the
ability to abstract over effects. You need to know concretely what
they are so that you can even type them. But hte effects you want to
be abstract.


François: so a renaming is like a handler that catches and
reraises. Why make it a primitive construct?

Leo: this works without needing to know what the operations on the
effects are. Even if the effect is abstract, you can rename it but you
can't handle it.


Gabriel: what you are proposing, could it be built easily on top of
the current runtime support? What about the name stuff for example?

Leo: names are locations on the heap right now.

Stephen: currently handlers are implemented as pattern-match. If the
pattern does not match, they delegate to a further handler. With this
proposal you are matching on three things, and sometimes you need to
substract one from an index.


Stephen: I think you basically need something along these lines. I do
quite like having fresh names sometimes. I'm not totally sure if it's
the exact feature you need in the end.

Leo: fresh name abstraction is very good, and fresh name generation is
also useful, I think we could add it on top. The system I presented is
the minimum you can get away with, I think.

Drup: how much do you expect average users to have to use these
renamings explicitly?

Leo: the kind of renaming where you rename 'counter' into 'dogs',
I wouldn't think it is very common. (Gabriel: of course everyone would
go for 'cats'.) In terms of having to do the renaming, two situations:

- clash you need to avoid, two libraries use the same name and you
  need to move one away

- higher-order functions like 'find'

Gabriel: if users write annotations on higher-order function
arguments, we can infer the shifts.

François: but then you lose the type-erasure property.

François: I find the current proposal simpler.

Stephen: I think you need to separate the semantics from the
type. Currently you introduce simultaneously a new type and a new
collection of constructor names.

Stephen: the renamings used in the shifted-names system differ from
nominal swaps. If you rename a to b, a.0 becomes b.0, [...] the
important point is that renamings compose in ways that swaps
don't. (rename a b); (rename b c) is equivalent to (rename a c), which
is not the case for swaps.

Didier: the proposed operational semantics is order-of-magnitude more
complex than Core ML or even Core Multicore ML.

Leo: denotational semantics look simpler!
(Operations are algebraic datatypes applied to copmutations, and handlers are substitutions.)


François: Daniel, KC, Anil, how do you feel about this?

Daniel: I think something like this is necessary; accidental capture
is both a feature and a problem. Sometimes it gives you exactly what
you want, sometimes it leads to problems.


François: I like the Effekt stuff better, it's simpler
  https://effekt-lang.org/

Stephen: the slides were showing the complex features to demonstrate
them. But if you write the simplest possible version, you see that
not_found leaks into the higher-order argument, and you wonder whether
this is right or not. and you can refine the program.


Gabriel: are we happy with the statu quo, no type-system and no syntax
for effect handlers?

Xavier: that's good to me. I don't find untyped effects evil, I think
they are also workable. Maybe having syntax for them would not be evil
either, but maybe we can wait.


Drup: we also need a good approach to effect polymorphism, I know you
have a proposal inspired by Frank.

Leo: yes, let's take 20mn to discuss it next time.



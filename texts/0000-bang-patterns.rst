- Feature Name: Bang Patterns
- Start Date: 2016-07-25
- RFC PR:
- Haskell Report Issue:

Much of this was copied from the Haskell Prime Trac page for ``BangPatterns``.
https://prime.haskell.org/wiki/BangPatterns

#######
Summary
#######


Our goal is to make it easier to write strict programs in Haskell. Programs that
use ``seq`` extensively are possible but clumsy, and it's often quite awkward or
inconvenient to increase the strictness of a Haskell program. This proposal
changes nothing fundamental; but it makes strictness more convenient.

The ``BangPatterns`` GHC extension introduces a new production to the syntax of
patterns,

.. code-block::

    pat ::= !pat

Matching an expression e against a pattern !p is done by first evaluating e (to
WHNF) and then matching the result against p, making strictness annotations much
less obtrusive and easier to use.




##########
Motivation
##########

.. code-block ::

    f1 x = x `seq` True

is strict in ``x``. Unfortunately, this scales very poorly once guards or
mulitple patterns are introduced,

.. code-block:: haskell

    -- Duplicate the seq on y
    f2 x y | g x       = y `seq` rhs1
           | otherwise = y `seq` rhs2

    -- Have a weird guard
    f2 x y | y `seq` False = undefined
           | g x           = rhs1
           | otherwise     = rhs2

    -- Use bang patterns
    f2 !x !y | g x       = rhs1
             | otherwise = rhs2

Bang patterns can be nested,

.. code-block:: haskell

    f3 (Just x,y) = x `seq` [x,y]

    -- using Bang Patterns
    f3 (Just !x, y) = [x,y]


Bang patterns and let bindings
------------------------------

In Haskell, ``let`` and ``where`` bindings can bind patterns. We propose to
modify this by allowing an optional bang at the top level of the pattern. Thus
for example:

.. code-block:: haskell

    let ![x,y] = e in b

The `!` should not be regarded as part of the pattern; after all, in a function
argument `![x,y]` means the same as `[x,y]`. Rather, the `!` is part of the
syntax of let bindings.

The semantics is simple; the above program is equivalent to:

.. code-block:: haskell

    let p@[x,y] = e in p `seq` b

That is, for each bang pattern, invent a variable p, bind it to the banged
pattern (removing the bang) with an as-pattern, and seq on it in the body of the
`let`. (Thanks to Ben Rudiak-Gould for suggesting this idea.)

A useful special case is when the pattern is a variable:

.. code-block:: haskell

    let !y = f x in b

means

.. code-block:: haskell

    let y = f x in y `seq` b

which evaluates the `f x`, thereby giving a strict `let`.




###############
Detailed design
###############

Grammar
-------

In section 3.17, add `pat ::= !pat` to the syntax of patterns.

Informal Semantics of Pattern Matching
--------------------------------------

In section 3.17.2, add new bullet 10, saying "Matching the pattern ``!pat``
against a value ``v`` behaves as follows: if ``v`` is bottom, the match diverges.
otherwise, ``pat`` is matched against ``v``.


Formal Semantics of Pattern Matching
------------------------------------

Fig 3.1, 3.2, add a new case (t):

.. code-block:: haskell

    case v of { !pat -> e; _ -> e' }
        ==> v `seq` case v of { pat -> e; _ -> e' }


Let expressions
---------------


In section 3.12, in the translation box, first apply the following
transformation: for each pattern ``π`` that is of form ``!qi = ei``, transform
it to ``xi@qi = ei``, and and replace ``e0`` by ``xi `seq` e0``. Then, when none
of the left-hand-side patterns have a bang at the top, apply the rules in the
existing box.


#########
Drawbacks
#########

Tricky syntax
-------------

What does this mean?

.. code-block:: haskell

    f ! x = True

Is this a definition of ``(!)`` or a banged argument? (Assuming that space is
insignificant.)

Proposal: resolve this ambiguity in favour of the bang pattern. If you want to
define ``(!)``, use the prefix form

.. code-block:: haskell

    (!) f x = True

Another point that came up in implementation is this. In GHC, at least, patterns
are initially parsed as expressions, because Haskell's syntax doesn't let you
tell the two apart until quite late in the day. In expressions, the form ``! x``
is a right section, and parses fine. But the form ``(!x, !y)`` is simply
illegal. Solution: in the syntax of expressions, allow sections without the
wrapping parens in explicit lists and tuples. Actually this would make sense
generally: what could ``(+ 3, + 4)`` mean apart from a tuple of two sections?


Pattern-matching semantics
--------------------------

A bang is part of a pattern; matching a bang forces evaluation. So the exact
placement of bangs in equations matters. For example, there is a difference
between these two functions:

.. code-block:: haskell

    f1  x True  = True
    f1 !x False = False

    f2 !x True  = True
    f2  x False = False

Since pattern matching goes top-to-bottom, left-to-right, ``f1 bottom True`` is
``True``, whereas ``f2 bottom True`` is ``bottom``.




In Haskell 98, these two bindings are equivalent:

.. code-block:: haskell

    { p1=e1; p2=e2 }
    -- and
    { (~p1,~p2) = (e1,e2) }

But with bang patterns this transformation only holds if ``p1``, ``p2`` are not
bang patterns. Remember, the bang is part of the binding, not the pattern.



Nested bangs (part 1)
---------------------

Consider this:

.. code-block:: haskell

    let (x, Just !y) = <rhs> in <body>

Is y evaluted before ``<body>`` is begun? No, it isn't. That would be quite
wrong. Pattern matching in a ``let`` is lazy; if any of the variables bound by
the pattern is evaluated, then the whole pattern is matched. In this example, if
``x`` or ``y`` is evaluated, the whole pattern is matched, which in turn forces
evaluation of ``y``. The binding is equivalent to

.. code-block:: haskell

    let t = <rhs>
        x = case t of { (x, Just !y) -> x }
        y = case t of { (x, Just !y) -> y }
    in <body>


Nested bangs (part 2)
---------------------

Consider this:

.. code-block:: haskell

    let !(x, Just !y) = <rhs> in <body>

This should be equivalent to

.. code-block:: haskell

    case <rhs> of { (x, Just !y) -> <body> }

Notice that this meant that the entire pattern is matched (as always with
Haskell). The Just may fail; x is not evaluated; but y is evaluated.

This means that you can’t give the obvious alternative translation that uses
just let-bindings and ``seq``. For example, we could attempt to translate the
example to:

.. code-block:: haskell

    let t = <rhs>
        x = case t of (x, Just !y) -> x
        y = case t of (x, Just !y) -> y
    in t `seq` <body>

This won't work, because using ``seq`` on ``t`` won't force ``y``. However, the
semantics says that the original code is equivalent to

.. code-block:: haskell

    let p@(x, Just !y) = <rhs> in p `seq` <body>

and we can desugar that in obvious way to

.. code-block:: haskell

    let t = <rhs>
        p = case t of p@(x, Just !y) -> p
        x = case t of p@(x, Just !y) -> x
        y = case t of p@(x, Just !y) -> y
    in p `seq` <body>

which is fine.

You could also build an intermediate tuple, thus:

.. code-block:: haskell

    let t = case <rhs> of p@(x, Just !y) -> (p,x,y)
        p = sel13 t
        x = sel23 t
        y = sel33 t
    in t `seq` <body>

Indeed GHC does just this for complicated pattern bindings.

Polymorphism
------------

Haskell allows this:

.. code-block:: haskell

    let f :: forall a. Num a => a->a
        Just f = <rhs>
    in (f (1::Int), f (2::Integer))

But if we were to allow a bang pattern, ``!Just f = <rhs>``, with the
translation to a case expression given earlier, we would end up with

.. code-block:: haskell

    case <rhs> of { Just f -> (f (1::Int), f (2::Integer) }

But if this is Haskell source, then ``f`` won’t be polymorphic.

One could say that the translation isn't required to preserve the static
semantics, but GHC, at least, translates into System F, and being able to do so
is a good sanity check. If we were to do that, then we would need

.. code-block:: haskell

    <rhs> :: Maybe (forall a. Num a => a -> a)

so that the case expression works out in System F:

.. code-block:: haskell

    case <rhs> of
        Just (f :: forall a. Num a -> a -> a)
            -> (f Int dNumInt (1::Int), f Integer dNumInteger (2::Integer)

The trouble is that ``<rhs>`` probably has a type more like

.. code-block:: haskell

    <rhs> :: forall a. Num a => Maybe (a -> a)

…and now the dictionary lambda may get in the way of forcing the pattern.

This is a swamp. **Conservative conclusion**: no generalisation (at all) for
bang pattern bindings.


#######################
Alternatives/extensions
#######################

Let bindings are irrefutable by default
---------------------------------------

Currently, bindings defined in ``let`` are irrefutable;

.. code-block:: haskell

    let x = bottom
    in expr

I (quchen) think this is out of scope of this proposal, since it changes the
semantics of a lot of existing bindings.



####################
Unresolved questions
####################

(none)

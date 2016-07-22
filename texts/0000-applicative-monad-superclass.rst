- Feature Name: Applicative Monad Proposal
- Start Date: 2016-07-22
- RFC PR:
- Haskell Report Issue:

.. highlight:: haskell

#######
Summary
#######

Make ``Applicative`` a superclass of ``Monad``, and ``Alternative`` of
``MonadPlus``, as already implemented in GHC as of 7.10 (released in early
2015).



##########
Motivation
##########


It's the right thing to do™
~~~~~~~~~~~~~~~~~~~~~~~~~~~

Math. You've all heard this one, it's good and compelling so I don't need to
spell it out.



Redundant functions
~~~~~~~~~~~~~~~~~~~

- ``pure`` and ``return`` do the same thing.
- ``>>`` and ``*>`` are identical.
- ``liftM`` and ``liftA`` are ``fmap``. The ``liftM*`` are ``liftA*``, ``<*>``
  is ``ap``.
- Prelude's ``sequence`` requres ``Monad`` right now, while ``Applicative`` is
  sufficient to implement it. The more general version of this issue is
  captured by ``Data.Traversable``, whose main typeclass implements the *same*
  functionality twice, namely ``traverse`` and ``mapM``, and ``sequenceA`` and
  ``sequence``.

That very much violates the “don't repeat yourself” principle, and even more so
it *forces* the programmer to repetition in order to achieve maximal generality.




Using Functor/Applicative functions in monadic code
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Whenever there's Monad code, you can use ``Functor``/``Applicative`` functions,
without introducing an additional constraint. Keep in mind that
“``Functor``/``Applicative`` functions” does not only include what their
typeclasses define but many more, for example ``void``, ``(<$>)``, ``(<**>)``.

Even if you think you have monadic code, strictly using the least restrictive
functions may result in something that requires only ``Applicative``. This is
similar to writing a function that needs ``Int``, but it turns out any
``Integral`` will do – more polymorphism for free.



Beginner friendliness
~~~~~~~~~~~~~~~~~~~~~

How often did you say …

- “A Monad is always an Applicative but due to historical reasons it’s not but
  you can easily verify it by setting ``pure = return`` and ``(<*>) = ap``”
- “``liftM`` is ``fmap`` but not really.” – “So when should I use ``fmap`` and
  when ``liftM``?” - *sigh*

With the new hierarchy, the answer would *always* be “use the least restrictive
one”.




###############
Detailed design
###############

Add the following definitions to the Prelude, and demand their existence in the
Report:

.. code-block:: haskell

   -- (Unchanged)
   class  Functor f  where

       -- Minimal complete definition: fmap

       fmap :: (a -> b) -> f a -> f b

       (<$) :: a -> f b -> f a
       (<$) =  fmap . const



   -- (Unchanged)
   class Functor f => Applicative f where

       -- Minimal complete definition: pure and (<*>)

       pure :: a -> f a

       (<*>) :: f (a -> b) -> f a -> f b

       (*>) :: f a -> f b -> f b
       (*>) x y = const id <$> x <*> y

       (<*) :: f a -> f b -> f a
       (<*) x y = const <$> x <*> y



   class Applicative m => Monad m where

       -- Minimal complete definition: (>>=) or join

       (>>=) :: m a -> (a -> m b) -> m b
       m >>= f = join (fmap f m)

       (>>) :: m a -> m b -> m b
       (>>) = (*>)

       return :: a -> m a
       return = pure

       fail :: String -> m a
       fail s = error s



   -- (Unchanged)
   class Applicative f => Alternative f where

       -- Minimal complete definition: empty and (<|>)

       empty :: f a

       (<|>) :: f a -> f a -> f a

       some :: f a -> f [a]
       some v = some_v
         where
           many_v = some_v <|> pure []
           some_v = (:) <$> v <*> many_v

       many :: f a -> f [a]
       many v = many_v
         where
           many_v = some_v <|> pure []
           some_v = (:) <$> v <*> many_v



   class (Alternative m, Monad m) => MonadPlus m where

       -- Minimal complete definition: nothing :-)

       mzero :: m a
       mzero = empty

       mplus :: m a -> m a -> m a
       mplus = (<|>)


These should also come with the usual/current laws, as can be seen in the
current ``base`` library.



#########
Drawbacks
#########


These are the kinds of issues to be expected:

1. Monads lacking Functor or Applicative instances. This is easily fixable by
   either setting ``fmap = liftM``, ``pure = return`` and ``(<*>) = ap``,
   although more efficient implementations may exist, or by moving an already
   existing definition from ``Control.Applicative`` to the appropriate module.

2. This one is specific to building GHC: importing ``Control.Monad/Applicative``
   introduces a circular module dependency. In this case, one can rely on
   handwritten implementations of the desired function, e.g.
   ``ap f x = f >>= ...``.

3. Libraries using their own ``(<*>)``. This one is much tougher, as renaming
   the operator may require a lot of effort. For building GHC though, this only
   concerns Hoopl, and a handful of renames.

All of these are found during compile time. Fixing is busy work, but neither
hard nor dangerous. To put this into perspective, fixing GHC took roughly 150
new Applicative/Functor superclass definitions, which took an afternoon to get
right.



#################################
Interactions with other proposals
#################################

This proposal lists a ``fail`` function, which is removed with the MonadFail
proposal (MFP). This proposal does not rely on ``fail``’s existence; the
definition is only included in order to decouple the two.



####################
Unresolved questions
####################

Since this has already been implemented in GHC and is enabled by default,
breakage as a result of being in the Report is minimal, if at all.
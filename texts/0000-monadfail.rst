- Feature Name: MonadFail Proposal (MFP)
- Start Date: 2016-07-22
- RFC PR:
- Haskell Report Issue:



#######
Summary
#######

Remove the ``fail`` function from the Monad type class, and move it to its own
``MonadFail`` class.

See also the Haskell Prime wiki: https://prime.haskell.org/wiki/Libraries/Proposals/MonadFail


#################
Motivation
#################

Currently, the ``<-`` symbol is unconditionally desugared as follows:

.. code-block:: haskell

    do pat <- computation    >>>    let f pat = more
       more                  >>>        f _ = fail "..."
                             >>>    in  computation >>= f

The problem with this is that fail cannot (!) be sensibly implemented for many
monads, for example ``Either``, ``State``, ``IO``, and ``Reader``. In those
cases it defaults to error As a consequence, in current Haskell, you can not use
``Monad`` polymorphic code safely, because although it claims to work for all
``Monad``, it might just crash on you. This kind of implicit non-totality baked
into the class is terrible.

The goal of this proposal is adding the ``fail`` only when necessary and
reflecting that in the type signature of the do block, so that it can be used
safely, and more importantly, is guaranteed not to be used if the type signature
does not say so.




###############
Detailed design
###############

Introduce a new typeclass:

.. code-block:: haskell

    class Monad m => MonadFail m where
        fail :: String -> m a

Desugaring can now be changed to produce this constraint when necessary. For
this, we have to decide when a pattern match can not fail; if this is the case,
we can omit inserting the ``fail`` call.

The most trivial examples of unfailable patterns are of course those that match
anywhere unconditionally,

.. code-block:: haskell

    do x <- action     >>>     let f x = more
       more            >>>     in  action >>= f

In particular, the programmer can assert any pattern be unfailable by making it
irrefutable using a prefix tilde:

.. code-block:: haskell

    do ~pat <- action     >>>     let f ~pat = more
       more               >>>     in  action >>= f

A class of patterns that are conditionally failable are newtype , and single
constructor data types, which are unfailable by themselves, but may fail if
matching on their fields is done with failable patterns.

.. code-block:: haskell

    data Newtype a = Newtype a

    -- "x" cannot fail
    do Newtype x <- action            >>>     let f (Newtype x) = more
       more                           >>>     in  action >>= f

    -- "Just x" can fail
    do Newtype (Just x) <- action     >>>     let f (Newtype (Just x)) = more
       more                           >>>         f _ = fail "..."
                                      >>>     in  action >>= f



############################
Discussion of design choices
############################

- What laws should fail follow?

  - Left zero: ``∀ s f. fail s >>= f ≡ fail s``
  - Right zero would rule out an ``IO`` instance

- What is the relationship to MonadPlus?

  - As the laws above indicate, fail is a close relative of mzero. We could
    suggest a default definition of ``fail _ = mzero``, which shows the
    intended usage and effect of the ``MonadFail`` class.
  - However, we should not remove ``fail`` and use only ``mzero`` instead. Not
    all types with ``Monad`` instances have ``MonadPlus`` instances. Some types
    do use the ``String`` argument to fail. For example, a parser might fail
    with a message involving positional information. Binary uses fail as their
    only interface to fail a decoding step. Some types have different
    definitions for mzero and fail. Although ``STM`` is ``MonadPlus`` it uses
    the default ``fail = error``. It should therefore not get a ``MonadFail``
    instance.

- Rename fail?

  - **No.** Old code might use fail explicitly and we should avoid breaking it.
    The Report talks about fail and we have a solid migration strategy that does
    not require a renaming.

- Remove the ``String`` argument?

  - **No.** The ``String`` might help error reporting and debugging. ``String``
    may be ugly, but it's the de facto standard for simple text in GHC. No high
    performance string operations are to be expected with fail so this breaking
    change would in no way be justified. Also note that explicit fail calls
    would break if we removed the argument.

- How sensitive would existing code be to subtle changes in the strictness
  behaviour of do notation pattern matching?

  - **It doesn't.** The implementation does not affect strictness at all, only
    the desugaring step. Care must be taken when fixing warnings by making
    patterns irrefutable using ``~`` as that does affect strictness. (Cf.
    difference between lazy/strict ``State``)

- Do we need a class constraint (e.g. ``Monad``) on ``MonadFail``?

  - **Yes.** The intended use of fail is for desugaring do-notation, not
    generally for any ``String -> m a`` function. Given that goal, we would
    rather keep the constraints simple as ``MonadFail m =>`` rather than the
    somewhat redundant ``(Monad m, MonadFail m) =>``.

- Can we relax the class constraint from Monad to ``Applicative``?

  - We don't necessarily have to choose now. Since ``Applicative`` is a
    superclass of ``Monad``, it is possible to change the superclass for
    ``MonadFail`` to ``Applicative`` later. This will naturally require a
    migration period, and the name will, of course, become misleading.

  For the sake of discussion, let's use the following definition:

  .. code-block:: haskell

      class Applicative f => ApplicativeFail f
        where
          fail :: String -> f a

  - Pros

    - ``ApplicativeDo`` is coming, and fail may be useful to combine pattern
      matching and ``Applicative`` code.

    - If the ``Monad`` constraint is kept, that would force ``Applicative`` code
      with pattern matching to be ``Monad`` code.

  - Cons

    - The constraints for Monad code using ``fail`` become ``(Monad m,
      ApplicativeFail m) =>`` instead of the simpler ``MonadFail m =>``. If we
      expect the common use of ``fail`` to be in ``Monad`` – not ``Applicative`` –
      do-notation, this leaves us with more verbose constraints.

  - Here are alternative definitions (with names open to debate) that would
    allow us to keep the constraints simple:

    - ``class Applicative f => ApplicativeFail f where failA :: String -> f a``
    - ``class ApplicativeFail m => MonadFail m where fail :: String -> m a; fail = failA``
    - Since we do not have much experience using ``ApplicativeDo``, it is not
      yet clear that this large of a change is useful.

- Which types with ``Monad`` instances will **not** have ``MonadFail``
  instances?

  - ``base``: ``Either``
  - ``transformers``: ?
  - ``stm``: ``STM``

- What ``MonadFail`` instances will be created?

  - ``base``: ``IO``
  - ``transformers``: Proposal for an ``Either`` instance using ``Monad`` instance in
    ``Control.Monad.Trans.Error``:
    ``instance MonadFail (Either String) where fail = Left``



#########
Drawbacks
#########

To estimate the breakage of this proposal, I compiled stackage-nightly
[somewhere in Fall 2015], and grepped the logs for the warnings. Assuming my
implementation is correct, the number of “missing ``MonadFail``” warnings
generated is 487. Note that I filtered out ``[]``, ``Maybe`` and ``ReadPrec``
since those can be given a ``MonadFail`` instance from within GHC, and no
breakage is expected from them.



####################
Unresolved questions
####################

See `Discussion of design choices`_

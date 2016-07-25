- Feature Name: Empty data types
- Start Date: 2016-07-25
- RFC PR:
- Haskell Report Issue:



#######
Summary
#######

Some data types do not have any constructors, such as

.. code-block:: haskell

    data Void

or the (in)famous

.. code-block:: haskell

    data State# s

used in the GHC implementation of ``IO``.

This proposal is about adding data types like these in a consistent, systemactic
way.



##########
Motivation
##########

Currently, there is no way to state that a data type has no constructors at all.
hile this might sound strange in isolation, consider a sum type like

.. code-block:: haskell

    data Either a b = Left a | Right b

This might arise in a scenarion in which it is known that the parameter ``a`` is
not inhabited, hence the ``Left`` case is impossible (up to ⊥). In GHC, we can
currently (and umambiguously) define a type for this,

.. code-block:: haskell

    data Void

without constructors (and indeed without even an ``=`` sign). Now,

.. code-block:: haskell

    Either Void a ~ a
    Maybe Void    ~ ()
    [Void]        ~ ()

In short, this allows us to define the unique data type that has no non-⊥ values
(up to isomoprhism). As such, this serves a smililar purpose as ``()``, the
unique data type that has exactly a single non-⊥ inhabitant (again, up to
isomorphism).

A particularly useful case of having *different* uninhabited types is as use as
phantom parameters such as

.. code-block:: haskell

    data Tagged tag a = Tagged a
    data Euro
    data USDollar

The phantom parameter ``tag`` is never used on the value level, yet the values
``Tagged 12 :: Tagged Euro Int`` and ``Tagged 12 :: Tagged Dollar Int`` can be
distinguished by the typechecker.





###############
Detailed design
###############

GHC currently ships with two extensions, which are proposed to be standardized
here: empty data declarations (``-XEmptyDataDecls``), and the related empty case
(``-XEmptyCase``).

Empty data declarations
-----------------------

An empty data declaration is a type declaration of the form

.. code-block:: haskell

    data Con      -- :: *
    data Con2 a b -- :: * -> * -> *

that has ``n`` type parameters, but lacks the ``=`` and any data constructors.
Such data types are of course only inhabited by ⊥.


Empty case expressions
----------------------

In order to match on empty data types, enable the possibility to write empty
case expressions,

.. code-block:: haskell

    case x of {}
    \case {} -- if lambda case is allowed

to explicitly match on no constructors at all. This is useful if the scrutinee
has no non-⊥ values, such as in

.. code-block:: haskell

    data Void
    absurd :: Void -> a
    absurd x = case x of {}


#########
Drawbacks
#########

The value of having empty case alternatives is debatable in the absence of other
type-level features; empty data declarations make sense even in the case of
current Haskell as phantom parameters. I (quchen) think that empty case should
be included mostly for consistency.



############
Alternatives
############

(none)


####################
Unresolved questions
####################

(none)

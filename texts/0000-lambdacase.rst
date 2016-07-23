- Feature Name: LambdaCase
- Start Date: 2016-07-23
- RFC PR:
- Haskell Report Issue:



#######
Summary
#######

For convenience, add the shorthand

.. code-block:: haskell

    \case            >>>   \x -> case x of
        Foo a -> …   >>>       Foo a -> …
        …            >>>       …

where ``x`` is a fresh variable.

This is currently implemented via the ``LambdaCase`` GHC extension.



##########
Motivation
##########


``LambdaCase`` is a syntactic convenience feature whose sole purpose is making
code a bit cleaner.


Example 1: Multiple pattern matches
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The following example repeats the function name multiple times:

.. code-block:: haskell

    myFunction (Foo (Just a)) = …
    myFunction (Bar b)        = …
    myFunction (Qux (a,b))    = …

Using lambda case, this could be refactored to

.. code-block:: haskell

    myFunction = \case
        Foo (Just a) -> …
        Bar b        -> …
        Qux (a,b)    -> …

This lacks one level of parentheses, indents the patterns to make the top-level
definition stand out more, and avoids repeating the function name multiple
times.


Example 2: Monadic binding
~~~~~~~~~~~~~~~~~~~~~~~~~~

To pass a value through a monadic chain of binds, pattern matching on an
intermediate result requires creating a new variable whose only purpose is to be
scrutinized by a ``case`` again in the next step:

.. code-block:: haskell

    main = foo >>= \x -> case x of Just bar -> …
                                   Nothing  -> …

Lambda case would clean this up by getting rid of the intermediate variable,

.. code-block:: haskell

    main = foo >>= \case Just bar -> …
                         Nothing  -> …



###############
Detailed design
###############

The Summary_ already describes this to sufficient detail.



#########
Drawbacks
#########

There are no technical drawbacks. Since ``case`` is a reserved keyword in
Haskell, it cannot be used as a variable name. Therefore, in standard Haskell,
``\case`` is invalid. Thus, this proposal gives meaning to a previously
non-existing construct, resulting in no breakage.



############
Alternatives
############

No alternatives tackling the same issue are known (to me).



####################
Unresolved questions
####################

(none)
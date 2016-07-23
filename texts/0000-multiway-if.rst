- Feature Name: Multiway If
- Start Date: 2016-07-23
- RFC PR:
- Haskell Report Issue:



#######
Summary
#######

Add multiway if expressions to the language,

.. code-block:: haskell

    if | condition1 -> …
       | condition2 -> …
       …

This is currently implemented as the ``MultiWayIf`` extension in GHC.



##########
Motivation
##########

Although we can have long chains of guards that fall through to each other on
the top level, there is no such capability for use inside expressions. Multiway
If remedies this by introducing the new expression

.. code-block:: haskell

    if | condition1 -> …
       | condition2 -> …
       …

which is equivalent to the standard Haskell expression

.. code-block:: haskell

    case () of
        _ | condition1 -> …
          | condition2 -> …
          …

This syntax takes syntactic noise away if there is no pattern matching,
``case``’s primary domain, is involved. It also agrees nicely with the standard
guard syntax that we already have right now.



###############
Detailed design
###############

The GHC user’s guide provides a fairly good explanation of the design; the
manual contents are as follows:


With ``-XMultiWayIf`` flag GHC accepts conditional expressions with
multiple branches:

.. code-block:: haskell

      if | guard1 -> expr1
         | ...
         | guardN -> exprN

which is roughly equivalent to

.. code-block:: haskell

      case () of
        _ | guard1 -> expr1
        ...
        _ | guardN -> exprN

Multi-way if expressions introduce a new layout context. So the example
above is equivalent to:

.. code-block:: haskell

      if { | guard1 -> expr1
         ; | ...
         ; | guardN -> exprN
         }

The following behaves as expected:

.. code-block:: haskell

      if | guard1 -> if | guard2 -> expr2
                        | guard3 -> expr3
         | guard4 -> expr4

because layout translates it as

.. code-block:: haskell

      if { | guard1 -> if { | guard2 -> expr2
                          ; | guard3 -> expr3
                          }
         ; | guard4 -> expr4
         }

Layout with multi-way if works in the same way as other layout contexts,
except that the semi-colons between guards in a multi-way if are
optional. So it is not necessary to line up all the guards at the same
column; this is consistent with the way guards work in function
definitions and case expressions.


#########
Drawbacks
#########

No new ambiguities are introduced, so no breakage is to expect.



############
Alternatives
############

(none?)



####################
Unresolved questions
####################

(none)
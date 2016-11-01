- Feature Name: Tuple sections
- Start Date: 2016-07-25
- RFC PR:
- Haskell Report Issue:



#######
Summary
#######

Standardize GHC’s TupleSections extension, introducing the sugar

.. code-block:: haskell

    (a,)
        ==> \x -> (a,x)
    (,b)
        ==> \x -> (x,b)
    (a,,c,)
        ==> \x y -> (a, x, c, y)

This makes tuple construction significantly easier, and reduces code noise. It
is also somewhat in line with the already existing infix operator section
syntax.



##########
Motivation
##########

GHC’s ``TupleSections`` extensions enables partially applied tuple constructors
like

.. code-block:: haskell

    (, True)

as a shorthand for the more unwieldy expression

.. code-block::

    \x -> (x, True)


###############
Detailed design
###############

Continuing the train of thought started in the Motivation_ section, the
extension goes even further and enables an omission of an arbitrary number of
omissions in n-tuples,

.. code-block::

    (, "'tis", , , , Just 'a', "scratch")

to mean

.. code-block::

    \x y z w -> (x, "'tis", y, z, w, Just 'a', "scratch")


The grammar for tuple syntax simply gets optional fields; it currently reads

.. code-block::

    aexp    → (exp1 , … , expk)  (k ≥ 2)

which would then become

.. code-block::

    aexp    → ([exp1] , … , [expk])  (k ≥ 2)



#########
Drawbacks
#########

Readability is a matter of taste; I would not consider this a drawback, but
others might.



############
Alternatives
############

(none)


####################
Unresolved questions
####################

(none, this has been in GHC for a long time)

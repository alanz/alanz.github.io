---
layout: post
title: "Scrap Your Zippers and GHC"
description: ""
category: "Haskell Refactorer"
tags: SYB
---
{% include JB/setup %}

Having used `Data.Tree.Zipper` in TokenUtils.hs for the Haskell
Refactorer, I have come to appreciate the power of zippers in terms of
having the whole structure available for modification as a context to
the specific location your code is working on.

I am currently working on a refactoring to lift a declaration one
level up in the code. For example, lifting the `pow = 2` declaration
in the code below

{% highlight haskell %}
sumSquares x y = sq x + sq y
    where
        sq::Int->Int
        sq 0 = 0
        sq z = z^pow
            where pow=2
{% endhighlight %}


should result in

{% highlight haskell %}
sumSquares x y = sq x + sq y
    where
        sq::Int->Int
        sq 0 = 0
        sq z = z^pow

        pow=2
{% endhighlight %}

being generated.

The transformation is achieved by doing a generic traversal over the
GHC `RenamedSource` AST

There are (at least) three possible strategies to achieve this.

1. Generic match at an appropriate level to be able to perform the
   transformation. This is the approach used in the original HaRe, but
   the GHC AST is different from the original so a much deeper match
   is required.

1. Create a generic traversal that has access to the surrounding
   zipper. This requires a modification of the standard SYB `extM`
   etc.

1. Split the transformation into opening the zipper via a generic
   query and then transforming it.

The third option is similar to what is happening in the HaRe
[TokenUtils][] using `Data.Tree.Zipper`.

  [TokenUtils]: https://github.com/alanz/HaRe/blob/hp2013-2/src/Language/Haskell/Refact/Utils/TokenUtils.hs

#### Preliminaries

Before `Data.Generics.Zipper` from [syz][] can be used for traversals,
it must be modified to deal with the undefined areas in the GHC AST.

  [syz]: http://hackage.haskell.org/packages/archive/syz/0.2.0.0/doc/html/Data-Generics-Zipper.html

There are a number of types in the AST that are populated with error
messages so that they blow up if they are traversed at an
inappropriate stage of GHC compilation. This unfortunately means that
specific care must be taken to avoid them in any generic code.

e.g.

{% highlight haskell %}
placeHolderType :: PostTcType	-- Used before typechecking
placeHolderType  = panic "Evaluated the place holder for a PostTcType"

placeHolderKind :: PostTcKind	-- Used before typechecking
placeHolderKind  = panic "Evaluated the place holder for a PostTcKind"
{% endhighlight %}

A further wrinkle is that the set of invalid types changes depending
on what phase (Parser,Renamer,TypeChecker) of the AST is being
examined.

We first define a test, parameterised on the phase, to determine if
the type may blow up. This just attempts to cast the current zipper
hole to the dangerous type, and if the cast succeeds returns True.

{% highlight haskell %}
checkZipperStaged :: Stage -> Zipper a -> Bool
checkZipperStaged stage z
  | isJust maybeNameSet    = checkItemStage stage (fromJust maybeNameSet)
  | isJust maybePostTcType = checkItemStage stage (fromJust maybePostTcType)
  | isJust maybeFixity     = checkItemStage stage (fromJust maybeFixity)
  | otherwise = False
  where
    maybeNameSet ::  Maybe NameSet
    maybeNameSet = getHole z

    maybePostTcType :: Maybe PostTcType
    maybePostTcType = getHole z

    maybeFixity :: Maybe GHC.Fixity
    maybeFixity = getHole z
{% endhighlight %}

The above function makes use of an existing (in HaRe) test for an item against
the specific stage

{% highlight haskell %}
-- | Checks whether the current item is undesirable for analysis in the current
--   AST Stage.
checkItemStage :: Typeable a => Stage -> a -> Bool
checkItemStage stage x = (const False `extQ` postTcType 
                                      `extQ` fixity 
                                      `extQ` nameSet) x
  where 
    nameSet :: NameSet        -> Bool
    nameSet    = const (stage `elem` [Parser,TypeChecker])
 
    postTcType :: GHC.PostTcType -> Bool
    postTcType = const (stage < TypeChecker)

    fixity :: GHC.Fixity     -> Bool
    fixity     = const (stage < Renamer)
{% endhighlight %}

With this test in hand, we can modify the existing traversals to make
use of it. So

{% highlight haskell %}
-- | Apply a generic transformation everywhere in a bottom-up manner.
zeverywhere :: GenericT -> Zipper a -> Zipper a
zeverywhere f z = trans f (downT g z) where
  g z' = leftT g (zeverywhere f z')
{% endhighlight %}

becomes

{% highlight haskell %}
-- | Apply a generic transformation everywhere in a bottom-up manner.
zeverywhereStaged :: (Typeable a) 
  => Stage -> GenericT -> Zipper a -> Zipper a
zeverywhereStaged stage f z
  | checkZipperStaged stage z = z
  | otherwise = trans f (downT g z)
  where
    g z' = leftT g (zeverywhereStaged stage f z')
{% endhighlight %}

Given that it is a transformation, we simply return the existing item
if it cannot be evaluated.

#### Opening the zipper

A traversal over a zipper does not really add much over existing SYB
traversals, unless we have access to the zipper while transforming or
querying the AST.

So we create a function which takes a generic query, and returns the
point in the zipper where the query matches. The zipper can then be
manipulated as required. 

{% highlight haskell %}
-- | Open a zipper to the point where the Geneneric query passes.
-- returns the original zipper if the query does not pass (check this)
zopenStaged :: (Typeable a) 
  => SYB.Stage -> SYB.GenericQ Bool -> Z.Zipper a -> [Z.Zipper a]
zopenStaged stage q z
  | checkZipperStaged stage z = []
  | Z.query q z = [z]
  | otherwise = reverse $ Z.downQ [] g z 
  where
    g z' = (zopenStaged stage q z') ++ (Z.leftQ [] g z')
{% endhighlight %}

Note that we are using a `GenericQ Bool`, so the standard `extQ`
combinators can be used to build up a compound query.

The above code is a variant of `zmapQ` in [syz][].

##### zopenStaged in use

Returning to our opening example, we first create a function to look
for the definition of the function to be lifted, which occurs in a
[Match][] buried in a [FunBind][] by way of a `MatchGroup`.

  [Match]: http://www.haskell.org/ghc/docs/7.6.3/html/libraries/ghc-7.6.3/HsExpr.html#t:Match
  [FunBind]: http://www.haskell.org/ghc/docs/7.6.3/html/libraries/ghc-7.6.3/HsBinds.html#v:FunBind

{% highlight haskell %}
liftToMatchQ :: GHC.Match GHC.Name -> Bool
liftToMatchQ (match@(GHC.Match pats mtyp (GHC.GRHSs rhs ds))::GHC.Match GHC.Name)
    = nonEmptyList (definingDeclsNames [n] (hsBinds  ds) False False) ||
      nonEmptyList (definingDeclsNames [n] (hsBinds rhs) False False)
{% endhighlight %}

The [Match][] has the patterns matched for the function, a possible
type annotation and then the RHS with optional local binds (ds).

In our example the `pow = 2` declaration occurs in the ds portion of
the second match of a MatchGroup.

So, if `renamed` holds the RenamedSource for the example,

{% highlight haskell %}
let [z] = zopenStaged Renamer (False `mkQ` liftToMatchQ) (toZipper renamed)
{% endhighlight %}

will provide a zipper `z` open to the Match containing the declaration.
The list will be empty if the query does not find a target.

#### Operating on the open zipper

We now wish to make a Monadic transformation of the zipper, and since
we have potentially passed in a multi-part GenericQ to open the
zipper, it must be a generic transform presented as a GenericM so that
it matches the appropriate item in the focus of the zipper.

We start with a function which first re-does the GenericQ to ensure
that it is in the right place, and runs the zipper level
transformation function where it matches.

{% highlight haskell %}
-- | Monadic transform of a zipper opened with a given generic query
transZM :: Monad m
  => SYB.Stage
  -> SYB.GenericQ Bool
  -> (SYB.Stage -> Zipper a -> m (Zipper a))
  -> Zipper a
  -> m (Zipper a)
transZM stage q t z
  | query q z = t stage z
  | otherwise = return z
{% endhighlight %}

The `transZM` function returns the zipper unchanged if it does not
match, so it can be chained in a fold.

Also, the transformation function has access to the AST stage and the
zipper, so can navigate further to find the point higher up in AST as
the target for the move.









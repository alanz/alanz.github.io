---
layout: post
title: "Haskell Refactorer Token Management"
description: ""
category: 
tags: []
---
{% include JB/setup %}

One of the most important aspects of a refactorer is that it must work
with real world source code. This means that when a user requests a
particular change, they expect ONLY that change to be made, and no
other changes to the source file or its layout.

On the other side, in order to reliably make the changes, the
refactorer needs to make use of the AST, also using the analysis the
compiler has done to resolve which names are the same and what the
types are.

Most compilers throw away the tokens once the AST is built.

GHC, via the GHC API, makes the rich token stream available. For each
token, this provides its location in the original source file in terms
of line,col start and end, and the original string in the source file
making up the token. This token stream includes non-trivial whitespace
such as comments.

Passing this rich token list to the appropriate function recreates the
original source file (except for a minor bug:
<http://hackage.haskell.org/trac/ghc/ticket/7351> which is easily
worked around.

The AST is well annotated with `SrcSpan`s tying it up to the original
rich token stream.

Theoretically it should be possible to perform the refactoring based
on the located AST, and relatively easily tie up the corresponding
tokens.

### Problems

There are some problems with this.

The first problem we face is that the locations in the AST refer to
the start and end of the specific tokens making up the syntactic
features parsed to create the AST element, e.g. a function or data
declaration. They do not take into account the comment tokens that are
also in the stream. This becomes particularly important if we are
moving things around, as people get upset if there comments suddenly
disappear.

The second problem is that Haskell is a layout sensitive language.
This means white space is important, and a module can fail to compile if
the layout is not handled properly.

Renaming a variable can make the resulting token and hence locations larger or
smaller, and influence layout.  e.g. in

{% highlight haskell %}
--Layout rule applies after 'where','let','do' and 'of'
--In this Example: rename 'sq' to 'square'.

sumSquares x y= sq x + sq y where sq x= x^pow
  --There is a comment.
                                  pow=2
{% endhighlight %}

the entire subordinate where clause needs to shift right to compensate
for the longer name.

{% highlight haskell %}
sumSquares x y= square x + square y where square x= x^pow
  --There is a comment.
                                          pow=2
{% endhighlight %}

The third problem is that the very close tie up between the AST and
the token stream only holds before any changes are made, and the token
output function expects the tokens to be in increasing order.  This
means that if a declaration is lifted to the top level, or demoted
from the top level to inside the declaration where it is used, the
locations in the AST and in the tokens need to be updated.

The fourth problem is more of a design goal. It should be possible to
perform refactorings by making changes to the AST, and the tokens
should automatically be taken care of. This is particularly important
as the detailed token and layout management is some of the fiddliest
code in the base, it needs to be taken care of once comprehensively
rather than generating an endless stream of stupid niggles.

### Solutions

Before presenting solutions, a brief disclaimer: I am more of a
practical programmer (like Michael Snoyman) than a theoretical one
(like Gabriel Gonzales), particularly when I am
still feeling my way through the problem space.

#### Annotated AST

The most obvious solution is to annotate the AST with the tokens.
Although I would love to do this, it requires some deep dives into
some pretty hairy type level programming to achieve, mainly because
the AST is a large set of mutually recursive types, rather than a
single type.

There is some promise in using the `Annotations` package, based on the
paper 'Generic Selections of Subexpressions' by M van Steenbergen,
which I have been experimenting with here
<https://github.com/alanz/annotations-play>, the problem is coming up
with a means of actually generating all the required boilerplate.

Some attempts at generating the required multirec instances are here
<https://github.com/alanz/ghc-multirec>.

In the longer term I think this approach holds promise, but I have
taken a more pragmatic approach in the mean time.

#### Automatically maintain AST and tokens

blah

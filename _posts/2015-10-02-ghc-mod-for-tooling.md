---
layout: post
title: "ghc-mod for tooling"
description: ""
category: "Haskell Refactorer"
---
{% include JB/setup %}

The Haskell Refactorer ([HaRe](https://github.com/alanz/HaRe)) makes use of
[ghc-mod](https://github.com/kazu-yamamoto/ghc-mod) to provide the low-level
interface to the Haskell source code being refactored.

This has a number of advantages

* it isolates HaRe from having to have a lot of fiddly code to deal with the
  mechanics of having an environment to load a project

* ghc-mod is a widely used tool, and so is kept up to date with all the changes
  in the surrounding ecosystem. For example, the recent Cabal changes, and
  providing support for stack

* ghc-mod can detect the specific environment it is running in, and make sure
  the correct GHC options are used when loading a particular source module for
  processing. This includes stack vs Cabal vs single file. It keeps track of the
  specific GHC options for a given module as defined in the Cabal target
  specification. If a cabal/stack configuration step has not been done since
  anything changed in the config, it will be done automatically to generate the
  component mappings and GHC options for each.

* ghc-mod target loading automatically sets flags appropriately if template
  haskell, quasi quotes or pattern synonyms are used.

* ghc-mod is able to decouple the version of GHC/Cabal used in the tooling from
  the one being used in the project being processed by the tooling. To do this
  decoupling, it makes use of the `cabal-helper` library to manage the
  appropriate version of Cabal for the specific project.
  
I think of ghc-mod as the Haskell tooling BIOS.

## How to use ghc-mod in this role

The ghc-mod source code itself is a bewildering array of custom Monads, all
being used to make sure the backward compatibility works, performance is
adequate for IDE usage, and so on. This makes it difficult to see which parts to
use for tooling.

The key to it all is that the `GhcModState` manages both the GHC session and the
mapping from Cabal targets to source components, and the GHC options to use for
each target.

{% highlight haskell %}
data GhcModState = GhcModState {
      gmGhcSession   :: !(Maybe GmGhcSession)
    , gmComponents   :: !(Map ChComponentName (GmComponent 'GMCResolved (Set ModulePath)))
    , gmCompilerMode :: !CompilerMode
    , gmCaches       :: !GhcModCaches
    , gmMMappedFiles :: !FileMappingMap
    }
{% endhighlight %}

The only part a tool-writer needs to care about is that there is a GHC session
in the ghc-mod state, and that loading a target file sets up this session with
the correct GHC options, regardless of the underlying project configuration.

`GhcModT` is defined as a state transformer wrapped around various other monad
transformers that are not important here.

{% highlight haskell %}
type GhcModT m = GmT (GmOutT m)

newtype GmT m a = GmT {
      unGmT :: StateT GhcModState
                 (ErrorT GhcModError
                   (JournalT GhcModLog
                     (ReaderT GhcModEnv m) ) ) a
    } deriving ( Functor
               , Applicative
               , Alternative
               , Monad
               , MonadPlus
               , MTL.MonadIO
#if DIFFERENT_MONADIO
               , GHC.MonadIO
#endif
               , MonadError GhcModError
               )
{% endhighlight %}

This allows a tool writer to simply use `GhcModT` in their monad stack, as in
HaRe, where the `GM` qualifier is used on ghc-mod imports

{% highlight haskell %}
newtype RefactGhc a = RefactGhc
    { unRefactGhc :: GM.GhcModT (StateT RefactState IO) a
    } deriving ( Functor
               , Applicative
               , Alternative
               , Monad
               , MonadPlus
               , MonadIO
               , GM.GmEnv
               , GM.GmOut
               , GM.MonadIO
               , ExceptionMonad
               )
{% endhighlight %}

The instances which cannot be automatically derived are

{% highlight haskell %}
instance GM.GmOut (StateT RefactState IO) where

instance GM.MonadIO (StateT RefactState IO) where
  liftIO = liftIO

instance MonadState RefactState RefactGhc where
    get   = RefactGhc (lift $ lift get)
    put s = RefactGhc (lift $ lift (put s))
{% endhighlight %}

Note that `GM.GmOut` is required to be defined, but is never used in HaRe so the
instance methods are not implemented.

In order to use the GHC API in HaRe the appropriate instances need to be defined, which simply route down into ghc-mod

{% highlight haskell %}
instance GHC.GhcMonad RefactGhc where
  getSession     = RefactGhc $ GM.unGmlT GM.gmlGetSession
  setSession env = RefactGhc $ GM.unGmlT (GM.gmlSetSession env)

instance GHC.HasDynFlags RefactGhc where
  getDynFlags = GHC.hsc_dflags <$> GHC.getSession
{% endhighlight %}

### Setting up a target session

With this setup the `setTargetSession` function will use all the ghc-mod
machinery to make sure that the right version of cabal is available if needed,
the project is configured using it (or stack is used for the equivalent), the
appropriate options are set in the GHC session and the target is loaded by GHC.

{% highlight haskell %}
setTargetSession :: FilePath -> RefactGhc ()
setTargetSession targetFile = RefactGhc $ GM.runGmlT' [Left targetFile] setDynFlags (return ())

setDynFlags :: GHC.DynFlags -> GHC.Ghc GHC.DynFlags
setDynFlags df = return (GHC.gopt_set df GHC.Opt_KeepRawTokenStream)
{% endhighlight %}

In the HaRe case the `DynFlags` need to be tweaked to ensure that the parser
does not discard comments in the API Annotations, hence `setDynFlags`.

### Done

The GHC API can now be used freely in the HaRe monad, with the session
configured precisely for the specific target file being processed.

If a different target file is required, calling `setTargetSession` is all that
is needed to ensure the correct session is available.

ghc-mod caches the sessions, as well as the cabal/stack configuration
information so this is not an expensive call unless something has to actually
change.


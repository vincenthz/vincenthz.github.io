+++
title = "Listing licenses with cabal-db"
date = 2014-03-29
draft = false
[taxonomies]
tags = ["cabal","license","cabal-db"]
categories = ["programming"]
+++

Following discussions with fellow haskellers, regarding the need to be careful
with adding packages that could depends on GPL or proprietary licenses, it
turns out it's not easy to get your dependencies's licenses listed.

<!-- more -->

It would be convenient to be able to ask the hackage database those things,
and this is exactly what cabal-db usually works with.

[cabal-db](http://hackage.haskell.org/package/cabal-db), for the ones that
missed the previous
[annoucement](http://tab.snarc.org/posts/haskell/2013-03-13-cabal-db.html), is a
simple program that using the index.tar.gz file, is able to recursively display
or search into packages and dependencies.

The license subcommand is mainly to get a summary of the licenses of a packages
and its dependencies, but it can also display the tree of licenses. Once
a package has been listed, it would not appears again in the tree even if
another package depend on it.

A simple example is better than many words:

~~~~ {.shell}
$ cabal-db license -s -t BNFC
BNFC: GPL
  process: BSD3
    unix: BSD3
      time: BSD3
        old-locale: BSD3
          base: BSD3
        deepseq: BSD3
          array: BSD3
      bytestring: BSD3
    filepath: BSD3
    directory: BSD3
  pretty: BSD3
  mtl: BSD3
    transformers: BSD3
  containers: BSD3
== license summary ==
BSD3: 14
GPL: 1
~~~~

cabal-db is only using the license listed in the license field in cabal files,
so if the field is incorrectly set, cabal-db would have no idea.



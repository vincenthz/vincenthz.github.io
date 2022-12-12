+++
title = "cabal-db : simple tool for cabal database queries"
date = 2013-03-13
draft = false
[taxonomies]
tags = ["cabal","cabal-db","graph","API","library","dependencies","dot"]
categories = ["programming"]
+++

Following previous experiment with Cabal library and querying the
state of the hackage world [here](http://tab.snarc.org/posts/haskell/2013-03-03-playing-with-cabal-lib.html),
I've extended and wrapped the tool into a cabal package.

<!-- more -->

A simple tool for cabal database query
======================================

* [hackage](http://hackage.haskell.org/package/cabal-db)
* [github](http://github.com/vincenthz/cabal-db/)

In addition to the graph feature, i've add and added some
others commands that i consolidated from some misc scripts:

* diff: run the diff command between two different versions of a package.
* revdeps: print all reverse dependencies of a package.
* info: print all available versions of a package and some misc information.
* search-author: search all the database for match in the author field.
* search-maintainer: search all the database for match in the maintainer field.

The revdeps feature of cabal-db can be also found in a web friendly yesod app with the
very useful [packdeps](http://packdeps.haskellers.com/)
or using the source from [Michael Snoyman's packdeps repository](https://github.com/snoyberg/packdeps)

The diff feature of cabal-db can be found using a gitweb frontend on [hdiff](http://hdiff.luite.com/)

Dependency Graph
----------------

After my previous [experiment](http://tab.snarc.org/posts/haskell/2013-03-03-playing-with-cabal-lib.html) to
display graph, i've added some nice improvements to make it even more useful.

* color: red for the packages queried, and green for the package in the platform. Easier to see
         how "far" you are from just the platform packages.
* hiding packages from the output: useful to hide pervasive packages like base or bytestring,
  that doesn't necessarily add information to your graph (e.g. every one depends on base)

Running on a single package (cryptohash):

~~~~
    $ cabal-db graph --hide base --hide bytestring cryptohash
    ...
~~~~

![cryptohash deps rendered by dot](/pictures/posts/2013-03-03-graph-cryptohash.png)

With this i can produce the graph of all the package i maintain with a single line:

~~~~
    $ cabal-db graph --hide base --hide bytestring $(cabal-db search-maintainer Hanquez | xargs)
    ...
~~~~

[all my pkgs rendered by dot](http://tab.snarc.org/pictures/posts/2013-03-03-graph-all.png)

Diff
----

Running diff between hit 0.4.2 and hit 0.4.3:

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
$ cabal-db diff hit 0.4.2 0.4.3
diff -Naur hit-0.4.2/Data/Git/Storage/FileWriter.hs hit-0.4.3/Data/Git/Storage/FileWriter.hs
--- hit-0.4.2/Data/Git/Storage/FileWriter.hs    2013-03-12 11:08:25.453936222 +0000
+++ hit-0.4.3/Data/Git/Storage/FileWriter.hs    2013-03-12 11:08:25.963936224 +0000
@@ -20,6 +20,14 @@
 
 defaultCompression = 6
 
+-- this is a copy of modifyIORef' found in base 4.6 (ghc 7.6),
+-- for older version of base.
+modifyIORefStrict :: IORef a -> (a -> a) -> IO ()
+modifyIORefStrict ref f = do
+    x <- readIORef ref
+    let x' = f x
+    x' `seq` writeIORef ref x'
+
 data FileWriter = FileWriter
         { writerHandle  :: Handle
         , writerDeflate :: Deflate
@@ -42,7 +50,7 @@
 postDeflate handle = maybe (return ()) (B.hPut handle)
 
 fileWriterOutput (FileWriter { writerHandle = handle, writerDigest = digest, writerDeflate = deflate }) bs = do
-        modifyIORef' digest (\ctx -> SHA1.update ctx bs)
+        modifyIORefStrict digest (\ctx -> SHA1.update ctx bs)
         (>>= postDeflate handle) =<< feedDeflate deflate bs
 
 fileWriterClose (FileWriter { writerHandle = handle, writerDeflate = deflate }) =
diff -Naur hit-0.4.2/hit.cabal hit-0.4.3/hit.cabal
--- hit-0.4.2/hit.cabal 2013-03-12 11:08:25.460602889 +0000
+++ hit-0.4.3/hit.cabal 2013-03-12 11:08:25.973936225 +0000
@@ -1,5 +1,5 @@
 Name:                hit
-Version:             0.4.2
+Version:             0.4.3
 Synopsis:            Git operations in haskell
 Description:
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Revdeps
-------

Or running revdeps on tls:

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
$ cabal-db revdeps tls
yesod-platform: tls (==1.1.2)
warp-tls: tls (>=1.1)
tls-extra: tls (>=1.1.0 && <1.2.0)
tls-debug: tls (>=1.1 && <1.2 && >=1.1 && <1.2 && >=1.1 && <1.2 && >=1.1 && <1.2)
pontarius-xmpp: tls (>=1.0.0)
network-conduit-tls: tls (>=0.9)
network-api-support: tls (>=0.9)
kevin: tls (==1.1.*)
imm: tls (-any)
http-proxy: tls (>=0.9 && <0.10)
http-enumerator: tls (>=0.9 && <0.10)
http-conduit-downloader: tls (-any)
http-conduit-browser: tls (-any)
http-conduit: tls (>=1.1.0)
ez-couch: tls (-any)
dropbox-sdk: tls (==0.9.*)
connection: tls (>=1.0)
azure-service-api: tls (>=1.0 && <1.1)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~


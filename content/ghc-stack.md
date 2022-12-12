+++
title = "Compiling GHC with Stack for Stack"
date = 2017-07-20
draft = false
[taxonomies]
tags = ["haskell","ghc","stack"]
categories = ["programming"]
+++

While Stack is really good at magically summoning all the compilers you need,
adding your own compiled compiler is not quite documented. For testing specific
version that doesn't have a release, or testing your own compiler modification,
it's useful to add your own compiler in a build tool that by default works
in a multi compiler settings.

<!-- more -->

First make sure you have the right building environment (a C compiler, the make
tool, etc.).  Also alex and happy are required, which you can get with:

    $ stack install alex happy

One of the useful thing that Stack does, is there's no system compiler anymore,
which manifest itself by not having ghc on the $PATH.  GHC requires one to
bootstrap itself, so we put the default stack one in the $PATH by starting a
new shell environment:

    $ stack exec --no-ghc-package-path bash 

Build GHC
---------

Clone the sources needed:

    $ git clone --recursive https://github.com/ghc/ghc

This will obviously give you the latest HEAD, so if you want
to rewind to a specific version, do it here.

Now we just build GHC; apply variant of those steps if you want
specific configuration here (see GHC building guide):

    $ cd ghc
    $ ./boot
    $ ./configure
    $ make

Make a binary dist
------------------

It's possible to skip this step by having ./configure called with the right
prefix above and doing `make install` now, but in the spirit of caching &
re-use, and also to adopt the exact same procedure that stack is doing when
installing a GHC compiler, we will create a binary dist.

To create a binary dist:

    $ make binary-dist

Now if everything went according to plan, you have a tarball on the root of the
ghc build repository in a format vaguely of `ghc-$VERSION-$ARCH-$SYSTEM.tar.xz`.
at this stage if you plan to reuse, you can cache it somewhere, make it
available for your company, etc..

Install for stack
-----------------

Unpack the bindist in a temp dir (don't forget to replace the variables):

    $ mkdir $TMPDIR
    $ cp $TARBALL $TMPDIR
    $ tar xvJf $TARBALL

Then run the bindist to install itself to the right place (again replacing the variables):

    $ cd $TARBALLDIR
    $ ./configure --prefix=$(stack path --programs)/$GHCVER
    $ make install
    $ echo -e "installed" > $(stack path --programs)/$GHCVER.installed

Configure your compiler
-----------------------

Now create a new `my.yaml` file to use your new compiler:

    compiler: $GHCVER
    compiler-check: match-exact
    resolver: $RESOLVER
    allow-newer: true

Make sure it works:

    $ stack --stack-yaml my.yaml ghc -- --version
    $GHCVER

That's it, it's all ready.

    $ stack --stack-yaml my.yaml build
    ...


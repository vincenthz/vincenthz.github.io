+++
title = "Combining Rust and Haskell"
date = 2015-09-28
draft = false
[taxonomies]
tags = ["haskell", "rust"]
categories = ["programming"]
+++

Rust is a pretty interesting language, in the area of C++ but more modern /
better. The stated goal of rust are: "a systems programming language focused
on three goals: safety, speed, and concurrency". Combining Rust with Haskell
could create some interesting use cases, and could replace use of C in some
projects while providing a more high level and safer approach where Haskell
cannot be used.

<!-- more -->

One of my reason for doing this, is that writing code targetting low-level
features is simpler in Rust than Haskell. For example, writing inline assembly
or some lowlevel OS routines. Also the performance of Rust is quite close to
C++, and I could see this being useful in certain case where Haskell is not as
optimised.

In this short tutorial, let's call Rust functions from Haskell.

The Rust library
----------------

First we start with an hypothetical rust library that takes a value, print to
console and return a value.

Our entry point in Rust is a simple rust\_hello, in a `src/lib.rs` file:

```rust
#[no_mangle]
pub extern fn rust_hello(v: i32) -> i32 {
    println!("Hello Rust World: {}", v);
    v+1
} 
```

One of the key thing here is the presence of the `no_mangle` pragma, that allow
exporting the name of the function as-is in the library we're going to generate.

Rust uses Cargo to package library and executable, akin to Cabal for haskell.
We can create the `Cargo.toml` for our test library:

```ini
[package]
name = "hello"
version = "0.0.1"
authors = ["Vincent Hanquez <vincent@snarc.org>"]

[lib]
name = "hello"
crate-type = ["staticlib"]
```

The only special trick is that we ask Cargo to build a static library in the
crate-type section, instead of the default rust lib (.rlib).

Haskell doesn't know other calling / linking convention like C++ (yet) or
Rust, which is why we need to go through those hoops.

We can now build with our Rust library with:

    cargo build

If everything goes according to plan, you should end up with a `target` directory where you can find the `libhello.a` library.

The haskell part
----------------

Now the haskell part is really easy, as this point there's no much difference
than linking with some static C library; first we create a `src/Main.hs`:

```haskell
{-# LANGUAGE ForeignFunctionInterface #-}
module Main where

import Foreign.C.Types

foreign import ccall unsafe "rust_hello" rust_hello :: CInt -> IO CInt

main = do
    v <- rust_hello 1234
    putStrLn ("Rust returned: " ++ show v)
```

Nothing special if you've done some C bindings yourself, otherwise I suggest
having a look at the [FFI](https://en.wikibooks.org/wiki/Haskell/FFI) article.

we can try directly linking with ghc:

    ghc -o hello-rust --make src/Main.hs -lhello -Ltarget/debug

and running:

```sh
$ ./hello-rust
Hello Rust World: 1234
Rust returned: 1235
```

That achieve the goal above. From there we can polish things and add this to a cabal file:

```
name:                hello-rust
version:             0.1.0.0
license:             PublicDomain
license-file:        LICENSE
author:              Vincent Hanquez
maintainer:          vincent@snarc.org
category:            System
build-type:          Simple
cabal-version:       >=1.10
extra-source-files:  src/lib.rs

executable hello-rust
  main-is:             Main.hs
  other-extensions:    ForeignFunctionInterface
  build-depends:       base >=4.8
  hs-source-dirs:      src
  default-language:    Haskell2010
  extra-lib-dirs:      target/release, target/debug
  extra-libraries:     hello
```

Note: The `target/release` path is to support building with the `-release` flag of cargo build.
by listing the `target/release` and then `target/debug`, it should allow you to pickup
the release library in preference to the debug library. It can also create some confusion,
and print a warning on my system when one of the directory is missing.

The missing step is either adding some pre-build rules to cabal `Setup.hs` to run cargo build, or
some more elaborate build system, both which are left as exercice to the interested reader.

Where this could go
-------------------

Going forward, this could lead to having another language from Haskell to
target that is not as lowlevel as C, but offer stellar performance and more
high level constructs (than C) without imposing any other runtime system. This
is interesting where you need to complete some tasks that Haskell is not quite
ready to handle (yet).

For example, a non exhaustive list:

* Writing cryptographic bindings in `rust&asm` instead of `C&asm`
* Heavily Vector-Optimised routines
* Operating system routines (e.g. page table handling) for a hybrid and safer operating system kernel.
* [Servo](https://github.com/servo/servo) embedding

Let me know in the comments anything else that might be of interests !

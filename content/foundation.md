+++
title = "Foundation"
date = 2016-09-09
draft = false
[taxonomies]
tags = ["haskell", "foundation", "text", "vector", "bytestring"]
categories = ["programming"]
+++

A new hope. Foundation is a new library that tries to define a new modern Haskell framework.
It is also trying to be more than a library: A common place for
the community to improve things and define new things

<!-- more -->

It started as a thought experiment:

**What would a modern Haskell base looks like if I could start from scratch ?**

What would I need to complete my Haskell projects without
falling into traditional pitfalls like inefficient String, all-in-one Num,
un-productive packing and unpacking, etc.

One of the constraints, that was set early on, was not depending on any
external packages, instead depending only on what GHC provides (itself, base, and libraries like ghc-prim).
While it may sound surprising, especially considering the usually high quality and precision of
libraries available on hackage, there are many reasons for not depending on anything;
I’ll motivate the reason later in the article, so hang on.

A very interesting article from Stephen Diehl on [production](http://www.stephendiehl.com/posts/production.html), that details
well some of the pitfalls of Haskell, or the workaround for less than
ideal situation/choice, outline pretty well some of the reasons for this effort.

Starting with the basic
=======================

One of the few basic things that you'll find in any modern haskell project, is
ByteArray, Array and packed strings.  Usually in the form of the `bytestring`,
`vector` and `text` packages.

We decided to start here. One of the common problem of those types is
their lack of inter-operability.  There's usually a way to convert one into
another, but it's either exposed in an `Unsafe` or `Internal` module, or has a
scary name like `unsafeFromForeignPtr`.

Then, if you're unlucky you will see some issues with unpinned and pinned
(probably in production settings to maximise fun); the common `ByteString` using
the pinned memory, `ShortByteString` and `Text` using unpinned memory, and
`Vector`, well, it's complicated (there's 4 different kind of Vectors).

Note: pinned and unpinned represent whether the memory is allowed to move by
the GC. Unpinned usually is better as it allows the memory system to reduce
fragmentation, but pinned memory is crucial for dealing with Input/Output with
the real world, large data (and some other uses).


Unboxed Array
-------------

Our corner stone is the unboxed array. The unboxed array is a native Haskell
ByteArray (represented by the `ByteArray#` primitive type),
and it is allowed to be unpinned or pinned (at allocation time). To also support
further interesting stuff, we supplement it with another constructor to make it
able to support natively a chunk of memory referenced by a pointer.

In simplified terms it looks like:

```haskell
data UArray ty = UArrayBA (Offset ty) (Size ty) PinnedStatus ByteArray#
               | UArrayAddr (Offset ty) (Size ty) (Ptr ty)
```

With this capability, we have the equivalent of `ByteString`,
`ShortByteString`, Unboxed `Vector` and (Some) Storable `Vector`,  implemented
in one user friendly type. This is a really big win for users, as suddenly all
those types play better together; they are all the same thing working
the same way.

Instead of differentiating `ByteString` and `Vector`, now `ByteString` disappears
completely in favor of *just* being a `UArray Word8`. This has been tried
before with the current ecosystem with [vector-bytestring](http://hackage.haskell.org/package/vector-bytestring).

String
------

String is a big pain point. Base represents it as a list of Char `[Char]`, which
as you can imagine is not efficient for most purpose. `Text` from the popular `text`
package implements a packed version of this, using UTF-16 and unpinned native `ByteArray#`.

While `text` is almost a standard in haskell, it’s very likely you’ll need
to pack and unpack this representation to interact with base functions, or
switch representation often to interact with some libraries.

Note on Unicode: UTF-8 is an encoding format where unicode codepoints are
encoded in sequence of 1 to 4 bytes (4 different cases). UTF-16 represent unicode
sequences with either 2 or 4 bytes (2 different cases).

Foundation’s String are packed UTF-8 data backed by an unboxed vector of bytes.
This means we can offer a lightweight type based on `UArray Word8`:

```haskell
newtype String = String (UArray Word8)
```

So by doing this, we inherit directly all the advantages of our vector types; namely
we have a `String` type that is unpinned or pinned (depending on needs), and supports
native pointers. It is extremely lightweight to convert between the two: provided
UTF8 binary data, we only validate the data, without re-allocating anything.

There’s no perfect representation of unicode; each representation has it own
advantages and disadvantages, and it really depends on what types of data you’re
actually processing. One of the easy rules of thumb is that the more your representation
has cases, the slower it will be to process the highest unicode sequences.

By extension, it means that choosing a unique representation leads to compromise.
In early benchmarks against text we are consistently outperforming `Text` when
the data is predominantly ASCII (i.e. 1-byte encoding).
In other type of data, it really depends; sometimes we’re faster still,
sometimes slower, and sometimes par.

Caveat emptor: benchmarks are far from reliable, and only been run on 2 machines
with similar characteristic so far.

Other Types
-----------

We also support already:

* **Boxed Array**. This is an array to any other Haskell types. Think of it as
array of pointers to another Haskell value
* **Bitmap**. 1 bit packed unboxed array

In the short term, we expect to add:

* **tree like structure**.
* **hash based structure**.

Unified Collection API
----------------------

Many types of collections support the same kind of operations.

For example, commonly you have very similar functions defined with different types:

```haskell
    take :: Int -> UArray a -> UArray a
    take :: Int -> [a]      -> [a]
    take :: Int -> String   -> String

    head :: UArray a -> a
    head :: String   -> Char
    head :: [a]      -> a
```

So we tried to avoid monomorphic versions of common Haskell functions and instead
provide type family infused versions of those functions. In foundation we have:

```haskell
    take :: Int -> collection -> collection

    head :: collection -> Element collection

```

Note: `Element collection` is a type family. this allow from a type "collection"
to define another type. for example, the Element of a `[a]` is `a`, and the Element of
`String` is `Char`.

Note2: `head` is not exactly defined this way in foundation: This was the simplest example
that show Type families in action and the overloading. foundation's `head` is not partial and defined:
`head :: NonEmpty collection -> Element collection`

The consequence is that the same `head` or `take` (or other generic functions) works
the same way for many different collection types, even when they are monomorphic (e.g. String).

For another good example of this approach being taken, have a look at the
[mono-traversable](https://hackage.haskell.org/package/mono-traversable) package

For other operations that are specific to a data structure, and hard to generalize,
we still expose dedicated operations.

The question of dependencies
============================

If you’re not convinced by how we provide a better foundation to the standard
Haskell types, then it raises the question: why not depend
on those high quality libraries doing the exact same thing ?

**Consistency**. I think it's easier to set a common direction, and have a consistent
approach when working in a central place together, than having N maintainers
working on M packages independently.

**Common place**. An example speaks better than words sometime: I have this X thing,
that depends on the A package, and the B package. Should I add it to A, to B, or
create a new C package ?

**Atomic development**. We don't have to jump through hoops to improve our types
or functions against other part of foundation. Having more things defined in a
same place, means we can be more aggressive about improving things faster,
while retaining an overall package that make sense.

**Versions, and Releases**. Far easier to depends on a small set of library
than depends on hundreds of different versions. Particularly in an industrial
settings, I will be much more confident tracking 1 package, watch 1 issue tracker
and deal with a set of known people, than having to deal with N packages, N issues trackers
(possibly in different places), and N maintainers.

Some final notes
================

**A fast iterative release schedule**. Planning to release early, release often.
and with a predictable release schedule.

**We're still in the early stage**. While we're at an exciting place,
don't expect a finish product right now.

**You don't need to be an expert to help**. anyone can help us shape foundation.

**Join us**. If you want to get involved: all Foundation works
take place in the open, on the [haskell-foundation organisation](https://github.com/haskell-foundation)
with code, proposals, issues and voting, questions.

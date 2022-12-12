+++
title = "Efficient CStruct"
date = 2017-03-20
draft = false
[taxonomies]
tags = ["haskell", "typelit", "compilation", "cstruct"]
categories = ["programming"]
+++

Dealing with complex C-structure-like data in haskell often
force the developer to have to deal with C files, and create
a system that is usually a tradeoff between efficiency, modularity
and safety.

The `Foreign` class doesn't quite cut it, external program needs C files,
binary parsers (binary, cereal) are not efficient or modular.

Let's see if we can do better using the advanced haskell type system.

<!-- more -->

First let define a common like C structure that we will re-use to compare
different methods:

```C
struct example {
    uint64_t a;
    uint32_t b;
    union {
        uint64_t addr64;
        struct {
            uint32_t hi;
            uint32_t low;
        } addr32;
    } addr;
    uint8_t data[16];
};
```

Dealing with C structure
------------------------

The offset of each field is defined as a displacement (in bytes) from the
beginning of the structure to point at the beginning of the field memory
representation. For example here we have:

* `a` is at offset 0 (relative to the beginning of the structure)
* `b` is at offset 8
* `addr.addr64` is at offset 12
* `addr.addr32.hi` is at offset 12
* `addr.addr32.low` is at offset 16
* `data` is at offset 20

The size of primitives is simply the number of bits composing the type; so a
`uint64_t`, composed of 64 bits is 8 bytes.  Union is a special construction
where the different option in the union are overlayed on each other and the
biggest element define its size.  The size of a struct is defined recursively
as the sum of all its component.

* field pointed by `a` is size 8
* field pointed by `b` is of size 4
* field pointed by `addr` is size 8
* field pointed by `data` is size 16
* the whole structure is size 36

What's wrong with Foreign
-------------------------

Here's the usual Foreign definition for something equivalent:

```haskell
data Example = Example
    { a    :: {-# UNPACK #-} !Word64
    , b    :: {-# UNPACK #-} !Word32
    , u    :: {-# UNPACK #-} !Word64
    , data ::                !ByteString
    } 

peekBs p ofs len = ...

instance Foreign Example
    sizeof _ = 36
    alignment _ = 8
    peek p = Example <$> peek (castPtr p)
                     <*> peek (castPtr (p `plusPtr` 8))
                     <*> peek (castPtr (p `plusPtr` 12))
                     <*> peekBs p 20 16
    poke p _ = ...
```

Given a (valid) Ptr, we can now get element in this by creating a new `Example`
type by calling `peek`. This will materalize a new haskell data structure in
the haskell GC-managed memory which have a copy of all the fields from the Ptr.

In some cases, copying all this values on the haskell heap is wasteful and not
efficient.  A simple of this use case, would be to quickly iterate over a block
of memory to check for a few fields values repeatedly in structure.

The `Foreign` type classes and co is only about moving data between the foreign
boundary, it's not really about efficiency dealing with this foreign boundary.

In short:

* Materialize values on the haskell side
* Not modular: whole type peeking/poking or nothing.
* Size and alignment defined on values, not type.
* No distinction between constant size types and variable size types.
* Often passing `undefined :: SomeType` to sizeof and alignment.
* Usually manually created, not typo-proof.

What about binary parsers
-------------------------

There's many binary parser on the market: [binary](http://hackage.haskell.org/package/binary)
, [cereal](http://hackage.haskell.org/package/cereal), [packer](http://hackage.haskell.org/package/packer),
[store](http://hackage.haskell.org/package/store).

Most of a binary parser job is taking a stream of bytes and efficiently turning
those bytes into haskell value. One added job is dealing with chunking, since
you may not have all the memory for parsing, you need to deal with values that
are cut between memory boundaries and have to deal with resumption.

Here's an example of a binary parser for example:

```haskell
getExample :: Get Example
getExample =
    Example <$> getWord64Host
            <*> getWord32Host
            <*> getWord64Host
            <*> getByteString 16
```

However, intuitively this has the exact same problem as `Foreign`, you can't
selectively and modularly deal with the data, and this create also
full copy of the data on the haskell side. This is clearly warranted when
dealing with memory that you want processed in chunks, since you
can't hold on to the data stream to refer to it later.

Defining a C structure in haskell
---------------------------------

Dealing with memory directly is error prone and it would be nice to able
to simulate C structures overlay on memory without having to deal with
size, offset and composition manually and to remain as efficient as possible.

First we're gonna need a recent GHC (at least 8.0) and the following extensions:

```haskell
{-# LANGUAGE DataKinds #-}
{-# LANGUAGE TypeOperators #-}
{-# LANGUAGE UndecidableInstances #-}
{-# LANGUAGE ScopedTypeVariables #-}
{-# LANGUAGE FlexibleContexts #-}
{-# LANGUAGE AllowAmbiguousTypes #-}
```

Then the following imports:

```haskell
import GHC.TypeLits
import Data.Type.Bool
import Data.Proxy
import Data.Int
import Data.Word
```

We define a simple ADT of all the possible elements that you can find, and their compositions:

```haskell
data Element =
      FInt8
    | FWord8
    | FInt16
    | FWord16
    | FInt32
    | FWord32
    | FInt64
    | FWord64
    | FFloat
    | FDouble
    | FLong
    | FArray Nat Element          -- size of the element and type of element
    | FStruct [(Symbol, Element)] -- list of of field * type
    | FUnion [(Symbol, Element)]  -- list of field * type
```

now `struct example` can be represented with:

```haskell
type Example = 'FStruct
    '[ '( "a"   , 'FWord64)
     , '( "b"   , 'FWord32)
     , '( "addr", 'FUnion '[ '( "addr64", 'FWord64)
                           , '( "addr32", 'FStruct '[ '( "hi", 'FWord32)
                                                    , '( "low", 'FWord32) ])
                           ])
     , '( "data", 'FArray 16 'FWord8 )
     ]
```

Calculating sizes
-----------------

Size is one of the key thing we need to be able to do on element.

Using a type family we can define the Size type which take an `Element` and returns a `Nat`
representing the size of the element. 

```haskell
type family Size (t :: Element) where
```

This is very easy for our primitives types:

```haskell
    Size ('FInt8)       = 1
    Size ('FWord8)      = 1
    Size ('FInt16)      = 2
    Size ('FWord16)     = 2
    Size ('FInt32)      = 4
    Size ('FWord32)     = 4
    Size ('FInt64)      = 8
    Size ('FWord64)     = 8
    Size ('FFloat)      = 4
    Size ('FDouble)     = 8
    Size ('FLong)       = 8 -- hardcoded for example sake, but would be dynamic in real code
```

The array is simply the Size of the element multiplied by the number of elements:

```haskell
    Size ('FArray n el) = n * Size el
```

For the constructed elements, we need to define extra recursive type families.
The structure is recursively defined to be the sum of its component Size, and
the union is recursively defined as the biggest element in it,.

```haskell
    Size ('FStruct ls)  = StructSize ls
    Size ('FUnion ls)   = UnionSize ls

type family StructSize (ls :: [(Symbol, Element)]) where
    StructSize '[]            = 0
    StructSize ('(_,l) ': ls) = Size l + StructSize ls

type family UnionSize (ls :: [(Symbol, Element)]) where
    UnionSize '[] = 0
    UnionSize ('(_,l) ': ls) = If (Size l <=? UnionSize ls) (UnionSize ls) (Size l)
```

Almost there, we only need a way to materialize the `Size` type, to have a
value that we can use in our haskell code:

```haskell
getSize :: forall el . KnownNat (Size el) => Integer
getSize = natVal (Proxy :: Proxy (Size el))
```

This looks a bit magic, so let's decompose this to make clear what happens;
first `getSize` is a constant Integer, it doesn't have *any* parameters. Next
the `el` type variable represent the type that we want to know the size of,
and the contraint on `el` is that applying the Size type function, we
have a `KnownNat` (Known Natural). In the body of the constant function we use
natVal that takes a Proxy of a KnownNat to materialize the value.

Given this signature, despite being a constant value, `getSize` need to determine
the element on which it is applied. We can use the Type Application to effectively
force the `el` element to be what we want to resolve to:

```haskell
> putStrLn $ show (getSize @Example)
36
```

Zooming with accessors
----------------------

One first thing we need to have an accessor types to represent how we represent
part of data structures.  For example in C, given the `struct example`, we want
to be able to do:

```C
    .a
    .addr.addr32.hi
    .data[3]
    .data
```

in a case of a structure or a union, we use the field name to dereference the structure,
but in case of an array, we use an integral index. This is really straighforward:

```haskell
data Access = Field Symbol | Index Nat
```

A List of `Access` would represent the zooming inside the data structures. The previous
example can be written in haskell with:

```haskell
    '[ 'Field "a" ]
    '[ 'Field "addr", 'Field "addr32", 'Field "hi" ]
    '[ 'Field "data", 'Index 3 ]
    '[ 'Field "data" ]
```

Calculating Offset
------------------

Offset of fields is the next important step to have full capabilities in this system

We define a type family for this that given an `Element` and `[Access]` would get back an offset in Nat.
Note that due to the recurvise approach we add the offset `ofs` to start from.

```haskell
type family Offset (ofs :: Nat) (accessors :: [Access]) (t :: Element) where
```

When the list of accessors is empty, we have reach the element, so we can just return the offset we have calculated

```ruby
    Offset ofs '[]          t                = ofs
```

When we have a non empty list we call to each respective data structure with:

* the current offset
* the name of field searched or the index searched
* either the dictionary of symbol to element (represented by `'[(Symbol, Element)]`) or the array size and inner Element

```haskell
    Offset ofs ('Field f:fs) ('FStruct dict) = StructOffset ofs f fs dict
    Offset ofs ('Field f:fs) ('FUnion dict)  = UnionOffset ofs f fs dict
    Offset ofs ('Index i:fs) ('FArray n t)   = ArrayOffset ofs i fs n t
```

Being a type enforced definition, it also mean that with this you can mix up
trying to `Index` into a Structure, or trying to dereference a `Field` into an
Array. the type system will (too) emphatically complain.

Both the Structure and Union will recursely match in the dictionary of symbol to find
a matching field. If we reach the empty list, we haven't found the right field
and the developper is notified with a friendly TypeError, at compilation time, that the field is
not present in the structure.

Each time an field is skipped in the structure the size of the element being skipped, is added to the current offset.

```haskell
type family StructOffset (ofs :: Nat)
                         (field :: Symbol)
                         (rs :: [Access])
                         (dict :: [(Symbol, Element)]) where
    StructOffset ofs field rs '[]                =
        TypeError ('Text "offset: field "
             ':<>: 'ShowType field
             ':<>: 'Text " not found in structure")
    StructOffset ofs field rs ('(field, t) ': _) = Offset ofs rs t
    StructOffset ofs field rs ('(_    , v) ': r) = StructOffset (ofs + Size v) field rs r

type family UnionOffset (ofs :: Nat)
                        (field :: Symbo)
                        (rs :: [Access])
                        (dict :: [(Symbol, Element)]) where
    UnionOffset ofs field rs '[]                 =
        TypeError ('Text "offset: field "
             ':<>: 'ShowType field
             ':<>: 'Text " not found in union")
    UnionOffset ofs field rs ('(field, t) ': _)  = Offset ofs rs t
    UnionOffset ofs field rs (_            : r)  = UnionOffset ofs field rs r
```

In the case of the array, we can just make sure, at compilation time, that the user is accessing
a field that is within bounds, otherwise we also notify the developer with a friendly TypeError.

```haskell
type family ArrayOffset (ofs :: Nat)
                        (idx :: Nat)
                        (rs :: [Access])
                        (n :: Nat)
                        (t :: Element) where
    ArrayOffset ofs idx rs n t =
        If (n <=? idx)
            (TypeError ('Text "out of bounds : index is "
                  ':<>: 'ShowType idx
                  ':<>: 'Text " but array of size "
                  ':<>: 'ShowType n))
            (Offset (ofs + (idx * Size t)) rs t)
```

A simple example of how the machinery works:

```haskell
> Offset ofs '[ 'Field "data", 'Index 2 ]) Example
> StructOffset ofs "data" ['Index 2 ]
               '[ '("a", ..), '("b", ..) , '("addr", ..), '( "data", ..) ]
> StructOffset (ofs + Size 'Word64) "data" ['Index 2]
               '[ '("b", ..) , '("addr", ..), '( "data", ..) ]
> StructOffset (ofs + 8 + Size 'Word32) "data" ['Index 2]
               '[ '("addr", ..), '( "data", ..) ]
> StructOffset (ofs + 12 + Size ('Union ..)) "data" ['Index 2 ]
               '[ '( "data", 'Farray 16 'FWord8) ]
> Offset (ofs + 20) ['Index 3] ('FArray 16 'FWord8)
> Offset (ofs + 20 + 3 * Size 'FWord8) [] 'FWord8
> ofs + 23
```

Now we can just calculate Offset of accessors in structure, we just need something to use it.

```haskell
getOffset :: forall el fields . (KnownNat (Offset 0 fields el)) => Integer
getOffset = natVal (Proxy :: Proxy (Offset 0 fields el))
```

Again same magic as `getSize`, and we also define a constant by construction.
We also start counting the offset at 0 since we want to calculate absolute
displacement, but we could start at some other points depending on need, and
prevent a runtime addition if we were to know the starting offset at compilation
for example.

```haskell
> putStrLn $ show (getOffset @Example @('[])
0
> putStrLn $ show (getOffset @Example @('[ 'Field "a"]))
0
> putStrLn $ show (getOffset @Example @('[ 'Field "b"]))
8
> putStrLn $ show (getOffset @Example @('[ 'Field "addr, 'Index "addr32", 'Field "lo" "]))
16
> putStrLn $ show (getOffset @Example @('[ 'Field "data, 'Index 3 ]))
23
```

Conclusion
-----------

One nice aspect on this is that you can efficiently nest structure, and you can
without a problem re-use the same field names for structure.

You can also define at compilation all sorts of different offsets and sizes
that automatically recalculate given their structures, and combine together.

With this primitive machinery, it's straighforward to define an efficient,
safe, modular accessors (e.g. peek & poke) functions on top of this.

Code
----

You can find the code:

* [Code Gist](https://gist.github.com/vincenthz/9c840ec99172c495a811b9e50c15c788)
* [Experimental Example Usage Gist](https://gist.github.com/vincenthz/34f0dc42128491317329b42f00fe5294)

Notes
-----

1. Packing & Padding

In all this code I consider the C structure packed, and not containing any
padding. While the rules of alignment/padding could be added to the calculation
types, I chose to ignore the issue since the developper can always from a
packed structure definition, add the necessary padding explicitely in the
definition.  It would also be possible to define special padding types that
automatically work out their size given how much padding is needed.

2. Endianness

I completely ignore endianness for simplicity purpose, but a real library would
likely and simply extend the definitions to add explicit endianness for all
multi-bytes types.

3. Nat and Integer

It would be nice to be able to generate offset in machine Int or Word, instead
of unbounded Integer.  Sadly the only capability for Nat is to generate Integer
with `natVal`.  The optimisation is probably marginal considering it's just a
constructor away, but it would prevent an unnecessary unwrapping and possibly
even more efficient code.

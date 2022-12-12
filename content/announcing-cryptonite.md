+++
title = "Announcing: cryptonite"
date = 2015-06-02
draft = false
[taxonomies]
tags = ["haskell", "crypto", "cryptography"]
categories = ["programming", "crypto"]
+++

For the last 5 years, I've worked intermittently on cryptographic related packages for Haskell.
Lately, I've consolidated it all in one single package. Announcing [cryptonite](http://hackage.haskell.org/package/cryptonite)

<!-- more -->

This new package merges the following packages:

* [cryptohash](http://hackage.haskell.org/package/cryptohash)
* [cryptocipher](http://hackage.haskell.org/package/cryptocipher)
* [crypto-random](http://hackage.haskell.org/package/crypto-random)
* [crypto-pubkey-types](http://hackage.haskell.org/package/crypto-pubkey-types)
* [crypto-pubkey](http://hackage.haskell.org/package/crypto-pubkey)
* [crypto-numbers](http://hackage.haskell.org/package/crypto-numbers)
* [crypto-cipher-types](http://hackage.haskell.org/package/crypto-cipher-types)
* [crypto-cipher-tests](http://hackage.haskell.org/package/crypto-cipher-tests)
* [crypto-cipher-benchmarks](http://hackage.haskell.org/package/crypto-cipher-benchmarks)
* [cprng-aes](http://hackage.haskell.org/package/cprng-aes)
* [cipher-rc4](http://hackage.haskell.org/package/cipher-rc4)
* [cipher-des](http://hackage.haskell.org/package/cipher-des)
* [cipher-camellia](http://hackage.haskell.org/package/cipher-camellia)
* [cipher-blowfish](http://hackage.haskell.org/package/cipher-blowfish)
* [cipher-aes](http://hackage.haskell.org/package/cipher-aes)
* [afis](http://hackage.haskell.org/package/afis)

Also this package adds support for the following features:

* [ChaCha](http://cr.yp.to/chacha.html)
* [Poly1305](http://cr.yp.to/mac.html)
* [Salsa](http://cr.yp.to/snuffle.html)
* [PBKDF2](http://tools.ietf.org/html/rfc2898)
* [Scrypt](http://www.tarsnap.com/scrypt.html)
* [Curve25519](http://cr.yp.to/ecdh.html)
* [Ed25519](http://ed25519.cr.yp.to/papers.html)
* A faster and more secure NIST P256 ECC support (through Google P256 implementation)

Why this new packaging model ?
------------------------------

This is mostly rooted in three reasons:

* Discoverability
* Cryptographic taxonomy
* Maintenance

Discovering new packages in our current world of hackage is not easy.
Unless you communicate heavily on new packages, there's a good chance that most
people would not know about a new package, leading to re-implementation,
duplicated features, and inconsistencies.

Cryptography taxonomy is hard, and getting harder; cryptographic primitives
are re-used creatively for making hash from cipher primitive, or random
generator from cipher, or authentification code from galois field operations.
This does create problems into where the code lives, how the code is tested,
etc. 

Then, finally, if I have to choose a unique reason for doing this, it will be
maintenance.  Maintenance of many cabal packages is costly in time: lower
bounds, upper bounds, re-installation, compatibility modules, testing framework, benchmarking
framework.

My limited free time has been siphoned into doing unproductive cross packages
tasks, for example:

* Upgrading bounds
* Sorting out ghc database of installed-packages when reinstalling packages for testing features
* Duplicating compatibility modules for supporting many compilers and library versions
* Maintaining meta data for many packages (e.g. LICENSE, CHANGELOG, README, .travis, .cabal)
* Tagging and releasing new versions

Doing everything in one package, simplifies the building issues, gives a better
ability to test features easily, makes a more consistent cryptographic
solution, and minimizes meta data changes.

What happens to other crypto packages ?
---------------------------------------

Cryptonite should be better in almost every aspect: better features, better testing.
So there are no real reasons to maintain any of the old packages anymore, so in
the long run, I expect most of those packages to become deprecated. I encourage
everyone to move to the new package.

I'll try to answer any migration questions as they arise, but most of the migration
should be straightforward in general.

I'm committed to maintain cryptohash for now, as it is very widely used. I'll
try to maintain the rest of the packages for now, but don't expect this to last
for long.

Otherwise, If some people are interested in keeping certain other pieces
independent and maintained, come talk to me directly with motivated arguments.

Contributing
------------

I hope this does bring contributions, and this becomes a more
community-maintained package, and specially that cryptonite becomes the
canonical place for anything cryptography related in Haskell.

Main things to look out, for successful contributions:

* respect the coding style
* do not introduce dependencies

Also you don't need to know every little thing in cryptography to help
maintain and add feature in cryptonite.

PS: I'm also looking forward to more cryptography related discussions
about timing attacks, what source of random is truly random, etc. :-þ

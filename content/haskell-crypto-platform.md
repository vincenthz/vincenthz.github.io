+++
title = "haskell crypto platform"
date = 2013-10-25
draft = false
[taxonomies]
tags = ["haskell","crypto","platform"]
categories = ["crypto", "programming"]
+++

One of my side projects that has been running for couple of years now,
was to get Cryptography up to scratch in haskell. Back when
I started [TLS](http://hackage.haskell.org/package/tls), there were many
various cryptography related projects and libraries. Many were not easy to use,
none were consistent, many had performance problems.

<!-- more -->

Just like the haskell platform, I dreamt of having a go-to set of packages
for cryptography. This is why i've started this project on a side of actual packages.

Currently the platform is made of those features:

* ASN.1: a terrible serialization format, that is unfortunately pervasive in cryptography.
* X.509: public certificate infrastructure based on ASN.1.
* symmetric ciphers (block and stream): RC4, DES (inherited from Crypto), 3DES, Blowfish, Camellia, AES
* cryptographic hashes (SHA1, SHA2, SHA3, MD2, MD4, MD5, ...) and siphash
* assymetric ciphers: RSA, DSA, DH
* cryptographic randomness: entropy gathering, Pseudo and random secure bytes generation
* securemem: auto scrubbing "bytestring".

A side note, about reimplementing cryptography
----------------------------------------------

Many people would think that it is a foolish project to reimplement cryptography.
It's undeniable, that cryptography is a hard subject and it's easy to get
some stuff horribly wrong. And many people would rather use references
implementations out there (openssl, nacl, ..).

There's still some values in this, despite not having always the best and most audited implementations:

* portability: all the code available run on all 3 platforms without having to install external libraries
* some cryptography code doesn't require implementation security: for example verifying RSA signature, decrypting your local files for your own purpose, etc.
* More diversity: monoculture is dangerous.
* It's much more haskell friendly :-)
* A common API framework that alternative implementation could/can use.

Despite not necessarily being the best cryptography code out there,
I do want to stress that I still consider many of the libraries
available throught this platform to be good in many contexts.

We want you ..
--------------

Despite my end goals, not everything is as good as I would want,
some ciphers inherited from Crypto are slows, some API not ideal, etc.

Open a discussion somewhere on one of the tracker, send some pull requests,
send me email about supporting your favorite feature, etc..

The organisational part of the platform is not yet defined, while in the process of making stuff
up, i do value suggestions on how to organize. 

Also, more documentation would be nice, and while i'm trying to document everything, it would
be better to have end-user trying to document pieces that is not documented well enough.
As an author, it's sometimes difficult to judge where more or better documentation would be
needed.

Other implementations
---------------------

One of the side goals of the platform, that I would like to see developed,
would be to also provide implementation alternatives with the same API infrastructure.

Thanks to Stefan Bühler there's already one package that wander in this direction:

[nettle haskell bindings](http://hackage.haskell.org/package/nettle)

I'm looking forward to similar packages covering other parts of the platform.

License
-------

For maximum usability, everything in the crypto-platform is under the BSD3 license.
All additions will be required to be under the same (or very similar) license too.

Links
-----

* [The main repository](https://github.com/vincenthz/hs-crypto-platform)
* [platform README](https://github.com/vincenthz/hs-crypto-platform/blob/master/README.md)
* [individual packages list](http://hackage.haskell.org/packages/tag/crypto-platform)

There's also a google group where to ask question about the packages:

[google group - haskell crypto platform](https://groups.google.com/forum/#!forum/haskell-crypto-platform)

And of course, the issue tracker of either specific packages or the platform one on github.

Enjoy,

+++
title = "Downloading safely"
date = 2015-05-30
[taxonomies]
tags = ["security", "download", "digest", "tls"]
categories = ["security"]
+++

All too often, things are downloaded without safety from hosts and mirrors.
Here's a practical guide to know where you stand and improve the situation.

<!-- more -->

I've seen countless example of script, Docker file, etc, that
will do something akin to:

~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~ {.shell}
wget http://my-url/package (or curl)
install package
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

For people that don't understand unix like shell script, that download a
package on a HTTP server using a plain text connection, then try to install it.

It's very likely that if you have some piece of automated machinery or a script,
you want this, to download a very specific data (for example a specific version of a compiler).

What can possibly go wrong ?
----------------------------

Pretty much everything:

* Your query could have been intercepted and modified: you don't know which server you're talking to.
* The data could have tempered on the destination (for example, adding a malware in a .tar.gz)
* An unauthorized person could have modified the data (e.g. someone manage to do a release but its not the 

Can't I just use a secure transport and be done ?
-------------------------------------------------

This is a good idea, but secure transport only secure the transport part.

A secure transport make sure you know that what you are sending and receiving
data comes from and go to a specific server without fear of eavedropping and
meddling.

But it doesn't gives you any guarantees that the data has not been changed on the server,
e.g. the server was compromised and the package you're downloading has been tempered.

Signature to the rescue !
-------------------------

Usually using GPG signatures, one can make sure that the data has been released
by the correct person/team. For example, the developers in the team release
a software signing with their personal keys.

Now, we can detect that data has been released by a specific person (or team),
which prevent the data being tempered by third party.

But still not enough: You still rely on something you don't have any control
for your security; the gpg key could have been compromised, or the people
owning the key could have been compromised.

Knowing what to expect
----------------------

A simple technique to represent an arbitrary sized piece of data into a finite
fingerprint is based on digest supported by a [cryptographic hash function](https://en.wikipedia.org/wiki/Cryptographic_hash_function).

More specifically we're looking for those 2 properties in the algorithm generating the digest in this case:

> it is infeasible to modify a message without changing the hash.
> it is infeasible to find two different messages with the same hash.

By embedding a digest of what you expect to download in your script, 
you know that no one can modify it without you knowing about it on the
other side.

Which means that the security of the transport doesn't matters anymore,
nor that the security of GPG keys of some people (or group), to determine
that you have downloaded what you expect.

It doesn't make anything secure by itself, but it allow you to vet what you
install, for example by looking at the source, or sandboxing the app, to see what it does,
in a restricted context. Then you only (or your organisation) become in charge
of rubber-stamping this specific version with a digest.

Finalizing
----------

To conclude, hashing is one of the most powerful and simple technique
to make sure the data has not changed and it is the exact same thing
you expect.

Here a short table on the risks related to transport and validation method used:

|Validation Method|Transport  |       MITM |  Tempering| Author Validated|
|-----------------|-----------|------------|-----------|-----------------|
|None             |Plain      |Unprotected |   Possible|               No|
|None             |Secure     |  Protected |   Possible|               No|
|GPG              |Plain      |  Protected |   Possible|              Yes|
|GPG              |Secure     |  Protected |   Possible|              Yes|
|Hash             |Plain      |  Protected | Impossible|               No|
|Hash             |Secure     |  Protected | Impossible|               No|
|Hash+GPG         |Plain      |  Protected | Impossible|              Yes|
|Hash+GPG         |Secure     |  Protected | Impossible|              Yes|

Note: it's assumed that the algorithm used for each specific purpose are perfect regarding to the properties they provide.

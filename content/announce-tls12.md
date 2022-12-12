+++
title = "announcement: tls-1.2 is out"
date = 2014-02-14
draft = false
[taxonomies]
tags = ["haskell", "tls"]
categories = ["programming"]
+++

One year ago, I've started some big changes on the [tls](http://hackage.haskell.org/package/tls) package.
I've finally manage to wrap it up in something that people can use straight out of hackage.

<!-- more -->

State improvements
------------------

One major limitation of previous tls versions, was that you wouldn't be able to
read and write data at the same time, since all the state was behind a big-lock
single mvar. Now the state is separated between multiple smaller states with
can be concurrently used:

* the RX state for receiving data.
* the TX state for sending data.
* the handshake state for creating new security parameters to replace RX/TX state when the handshake is finished
* Misc state for other values.

For many protocols like HTTP, this was never an issue as the reading and
writing are disjoints. But some others protocols that do intertwined read and
write (AMQP, IMAP, SMTP, ..) were rightfully having difficulties to use tls.

This provide a more scalable implementation, and optimise the structure changes
to the minimum needed.

Certificate improvements
------------------------

The second pharaonic change was a major rewrite of ASN.1, X509 and the handling
of certificate. The support libraries are now splitted in more logical units, and
provide all the necessary foundations for a much simplified handling of
certificates.

ASN.1 that was previously all in [asn1-data](http://hackage.haskell.org/package/asn1-data) is splitted
into [asn1-types](http://hackage.haskell.org/package/asn1-types) for the high level ASN.1 Types,
[asn1-encoding](http://hackage.haskell.org/package/asn1-encoding) for BER and DER binary encoding support,
and [asn1-parse](http://hackage.haskell.org/package/asn1-parse) to
help with parsing ASN.1 representation into high level types. Generally,
the code is nicer and able to support more cases, and also more stress tested.

Certificate [certificate](http://hackage.haskell.org/package/certificate) is splitted too and is now deprecated in favor of:

* [x509](http://hackage.haskell.org/package/x509): Contains all the format
  parser and writer for certificate, but also now support CRL. The code has
  been made more generic and better account certificate formats from the real
  world.
* [x509-store](http://hackage.haskell.org/package/x509-store): Contains some
  routines to store and access certificates on disk; this is not very different
  than what was in [certificate](http://hackage.haskell.org/package/certificate").
* [x509-system](http://hackage.haskell.org/package/x509-system): Contains all
  routines to access system certificates, mainly the trusted CA certificates
  supported. The code is not different from [certificate](http://hackage.haskell.org/package/certificate")
  package, except there's now Windows supports for accessing the system
  certificate store.
* [x509-validation](http://hackage.haskell.org/package/x509-validation): One of
  the main security aspect of the TLS stack, is certificate validation, which
  is a complicated and fiddly process. The main fiddly aspect is the many input
  variables that need to be considered, combined with errata and extensions.
  The reason to have it as a separate package it to make it easy to debug,
  while also isolating this very sensitive piece of code. The feature is
  much more configurable and tweakable.

On the TLS side, previous version was leaving the whole validation process to a
callback function. Now that we have a solid stack of validation and support for
all main operating systems, tls now automatically provide the validation
function enabled and with the appropriate security parameters by default.  Of
course, It's still possible to change validation parameters, add some hooks and
add a validation cache too.

The validation cache is a way to take a fingerprint and get cached yes/no
answer about whether the certificate is accepted. It's a generic lookup
mechanism, so that it could work with any storage mechanism. The same mechanism
can be overloaded to support Trust-on-first-use, and exceptions fingerprint
list.

Exceptions list a great way to use self-signed certificates without
compromising on security; You have to do the validation process out-of-band to
make sure the certificate is really the one, and then store a tuple of the name
of the remote accessed associated with a fingerprint. The fingerprint is a
simple hash of the certificate, whereas the name is really just a simple
(hostname, service) tuple.

Key exchange methods
--------------------

Along with RSA signature, there's now DSA signature support for certificate.

Previous versions only supported the old RSA key exchange methods. After a bit
of refactoring, we now have DHE-RSA and DHE-DSS support. DHE is ephemereal
Diffie Hellman, and provide [Forward Secrecy](http://en.wikipedia.org/wiki/Forward_secrecy).

In the future, with this refactoring in place, ECDHE based key exchange methods
and ECDSA signature will be very easy to add.

API and parameters changes
--------------------------

The initialization parameters for a context is now splitted into multiples smaller structures:

* one for the supported parameters (versions, ciphers methods, compressions methods, ..)
* one for shared access structures (x509 validation cache, x509 CA store, session manager, certificate and keys)
* the client and server parameters are now 2 distinct structures. this is not anymore a common structure with a role part.

All this change allow better separation of what parameters are for the client
and the server, and should also make it easier to setup better default, and allow
tweaking of the configuration to be more self contain. The aim is only to have
to set your "Shared" structure, and for the remaining structures uses default.

 [tls-extra](http://hackage.haskell.org/package/tls-extra) has been merged in tls.

TLS Protocol Versions
---------------------

Previous tls packages were not able to downgrade protocol version. This is now completely fixed, and
by default tls will try to use the maximum supported version (by default, TLS 1.2)
instead of the version specified by the user (by default, TLS 1.0).

Client use
----------

For client connection, I recommend to use [connection](http://hackage.haskell.org/package/connection)
instead of tls directly.

Closing note
------------

And finally this is the extents of the modifications just in tls:

~~~~~~
 82 files changed, 5528 insertions(+), 4568 deletions(-)
~~~~~~

Enjoy,

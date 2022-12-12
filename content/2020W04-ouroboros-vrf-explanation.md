+++
title = "Ouroboros Verifiable Random Function"
date = 2020-01-26
draft = true
[taxonomies]
tags = ["cryptography", "blockchain", "VRF", "benchmarks"]
categories = ["cryptography"]
[extra]
math = true
diagram = true
+++

Verifiable Random Function (VRF) are one of the key cryptographic primitive for
[Ouroboros Praos](https://eprint.iacr.org/2017/573.pdf), that allows to participate
in the block creation lottery. Let's dig in the detail of the tech

<!-- more -->

# What is a VRF

VRF can be thought as the asymmetric key equivalent to keyed cryptographic hash.
In a nutshell, it uniquely allows the secret key owner to make a output for a given seed value
and anyone with the public key can verify that it was generated honestly from the right seed.

It's easy to see the similarily with a Message Authentication Code (MAC) construction, except
adding that the capability to generate and verify for different part of the key.

So, the VRF is mathematically defined as such:

* \\( \tt{Output} \in \lbrace 0,1 \rbrace^{vrf} , \tt{Seed} \in \lbrace 0,1 \rbrace^* \\)
* \\( \tt{generate} : SecretKey \to \lbrace 0,1 \rbrace^\* \to \lbrace 0,1 \rbrace^{vrf} \\)
* \\( \tt{verify} : PublicKey \to \lbrace 0,1 \rbrace^\* \to \lbrace 0,1 \rbrace^{vrf} \to \lbrace True,False \rbrace \\)

You can see the similarity compared to the MAC construction:
* \\( \tt{generate} : MacKey \to \lbrace 0,1 \rbrace^\* \to \lbrace 0,1 \rbrace^{mac} \\)
* \\( \tt{verify} : MacKey \to \lbrace 0,1 \rbrace^\* \to \lbrace 0,1 \rbrace^{mac} \to \lbrace True,False \rbrace \\)

There's plenty of online resources for more thorough and formal explanations of VRFs, and the ouroboros specific use:

* [Wikipedia](https://en.wikipedia.org/wiki/Verifiable_random_function)
* [Verifiable Random Functions 1999 - PDF](https://people.csail.mit.edu/silvio/Selected%20Scientific%20Papers/Pseudo%20Randomness/Verifiable_Random_Functions.pdf)
* [Ouroboros Praos - PDF](https://eprint.iacr.org/2017/573.pdf)

# How is it used ?

Each slot will be individually evaluated by each owner of stake in secret.
When succesfully evaluated, it gives the authorisation on a per slot basis
to generate a header.

Given the VRF output, we can map it to a range where the minimum of this range represent
0% of stake, and the maximum of the range represent 100% of stake:

$$ number : \lbrace 0,1 \rbrace^{vrf} \to \mathbb{N} $$

And we effectively evaluate this number to be under the owner threshold:

$$ \lbrace 0,1 \rbrace^{vrf} \ as\  \mathbb{N} \lt \tt{StakeThreshold} $$ 

This realise the major proposition of proof of stake, given stake in the
system, allow to speak a new block on the network proportional to the
stake you have in the system.

{% diagram() %}
graph TB
    A["Generate(epoch_random ++ slot_number)"]
    A -->|Output| B{"compare stake"}
    B -->|< Threshold| D["allowed for this slot"]
    B -->|>= Threshold| E["not allowed for this slot"]
{% end %}

So for example, given a 25% stake holder, with a number domain mapped to 8 bits
(0..255), the threshold is 64, the schedule given to this secret key will be:

| Slot number | VRF Output (decimal) | Authorisation |
| ----------- | -------------------- | ------------- |
| 1           | 0                    | ✅             |
| 2           | 124                  | ❌             |
| 3           | 180                  | ❌             |
| 4           | 65                   | ❌             |
| 5           | 80                   | ❌             |
| 6           | 63                   | ✅             |
| ...         | ...                  | ...           |

Every authorisation, give the ability for this stake holder to broadcast a
block to the network. It doesn't force the stake holder to create this block,
nor that it means that the stake holder is uniquely able to broadcast for
this slot either.

# Benchmark

The actual implementation is using 2HashDH-NIZK, as described in the ouroboros paper.
The curve choice for the discrete log was narrowed down to curve25519, as it allow
a secure implementation and is also blazingly fast:

On x86-64:

| Curve       | Function | Speed  |
| ----------- | -------- | ------ |
| Curve25519  | generate | 83 µs  |
| NIST-P256R1 | generate | 211 µs |
| Curve25519  | verify   | 103 µs |
| NIST-P256R1 | verify   | 227 µs |

On Aarch64:

| Curve       | Function | Speed   |
| ----------- | -------- | ------- |
| Curve25519  | generate | 1086 µs |
| NIST-P256R1 | generate | 1685 µs |
| Curve25519  | verify   | 1411 µs |
| NIST-P256R1 | verify   | 1529 µs |

Note: The Curve25519 implementation is using curve25519-dalek and is all in
rust, while the NIST-P256R1 is using haskell, and bindings to OpenSSL; The
aarch64 implementation of curve25519-dalek is using normal operation, and
probably could go faster with some low level optimisation.

It might sounds insignifiant to optimise at the hundreds of µs scale, but
given an epoch of 100000 slots, this means the evaluation budget for the
whole epoch schedule between Curve25519 to P256, move from 8.3s to 21s on x64,
and 108s to 168s on aarch64.

Also the verification is done everytime that there's a network block received,
so it's important to reduce the cost of those operation.

Last but not least, the pure rust implementation allow seamless production
ready compilation to webassembly, which allow to run in the context of a
browser / javascript.

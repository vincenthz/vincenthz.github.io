+++
title = "Cryptoxide perf (SHA2 / Blake2)"
date = 2021-01-17
draft = false
[taxonomies]
tags = ["cryptography", "performance", "benchmark"]
categories = ["cryptography"]
[extra]
graphs = true
+++

Related to some engine rewrite and SSE, AVX, AVX2 cpu optimisation I did last
year on [cryptoxide](https://github.com/typed-io/cryptoxide/) :

* [SHA2 optimisation](https://github.com/typed-io/cryptoxide/pull/8)
* [Blake2 optimisation](https://github.com/typed-io/cryptoxide/pull/9)

<!-- more -->

## History of cryptoxide

Cryptoxide is a fork of the initial [rust-crypto](https://github.com/DaGenix/rust-crypto)
one-stop cryptography package that went unmaintained.

In 2018, we needed a pure rust version to construct rust-wasm based
web-applications when this use case was in its infancy; rust-crypto was an
interesting starting point, as all the algorithms were written in pure rust,
and it was also easier to construct something than the exploded version which
would have required lots more time to port.

Many other cryptographic packages are now wasm friendly also.

## Benchmarks setup

* cpu: 3.6 GHz 8-Core Intel Core i9 (I9-9900K)
* rust compiler: stable 1.49
* cryptoxide: 0.3.0
* rust-crypto: blake2 0.9.1, sha2 0.9.1
* ring: 0.16.19

The benchmark code itself consist of benchmarking few time the main costly
part of each algorithm over a 10 megabytes array and taking the average of
the run. It's possible that the number reported could be buggy, but it should
be consistently buggy, so here we're more interested by the relative values
than the absolute values.

This benchmark is only looking at the function I was interested about also, thus
only compare Sha256, Sha512, Blake2b and Blake2s.

Finally benchmarks should always be taken with a grain of salt, as different
cpu and environment can lead to different results.

To play with the benchmark on your own machine, have a look at [rcc](https://github.com/vincenthz/rcc)

## Raw numbers

Let's start with the raw number in release mode;
This show the average (lower better) with standard deviation (the lower, the better for reliability of benchmark),
and the speed of processing (higher better):

Using the default target_cpu:

| Algorithm | Crate      | Avg(ms) | Std Dev(ms) | Speed(mb/s) |
| --------- | ---------- | ------- | ----------- | ----------- |
| blake2b   | cryptoxide | 10.18   | 0.174       | 981         |
| blake2b   | blake2     | 10.28   | 0.260       | 972         |
| blake2s   | cryptoxide | 15.97   | 0.264       | 625         |
| blake2s   | blake2     | 17.07   | 0.150       | 585         |
| sha256    | cryptoxide | 30.51   | 0.220       | 327         |
| sha256    | sha2       | 35.66   | 0.277       | 280         |
| sha256    | ring       | 19.17   | 0.293       | 521         |
| sha512    | cryptoxide | 20.86   | 0.319       | 479         |
| sha512    | sha2       | 21.10   | 0.422       | 473         |
| sha512    | ring       | 13.29   | 0.296       | 752         |

Using the native target_cpu `target_cpu=native`:

| Algorithm | Crate      | Avg(ms) | Std Dev(ms) | Speed(mb/s) |
| --------- | ---------- | ------- | ----------- | ----------- |
| blake2b   | cryptoxide | 6.72    | 0.229       | 1486        |
| blake2b   | blake2     | 9.95    | 0.388       | 1004        |
| blake2s   | cryptoxide | 11.27   | 0.232       | 886         |
| blake2s   | blake2     | 17.23   | 0.136       | 580         |
| sha256    | cryptoxide | 20.71   | 0.243       | 482         |
| sha256    | sha2       | 28.31   | 0.365       | 353         |
| sha256    | ring       | 19.74   | 0.283       | 506         |
| sha512    | cryptoxide | 17.13   | 0.184       | 583         |
| sha512    | sha2       | 17.50   | 0.339       | 571         |
| sha512    | ring       | 13.17   | 0.133       | 759         |

## In Graphs

Putting in graphical form, comparing the default and native runs:

{{ canvasdata(path="content/2021-01-17-cryptoxide-performance.json") }}

## Sha256

{{ canvas(id=1, width="90%", height="300px") }}

## SHA512

{{ canvas(id=2, width="90%", height="300px") }}

## BLAKE2B

{{ canvas(id=3, width="90%", height="300px") }}

## BLAKE2S

{{ canvas(id=4, width="90%", height="300px") }}


## Conclusion

Ring is the uncontested winner in term of performance (and probably safety);
Most or all algorithms are implemented in assembly and using the best level
of optimisation all the time; which explains default and native being
virtually identical.

Related to Sha256 algorithm, with native optimisation cryptoxide reach very close
to the very optimised ring implementation and have a noticeable difference with
the pervasive sha2 crate.

Related to Sha512 algorithm, there's no significant difference between cryptoxide and sha2,
which is not particularly surprising considering that I didn't take time to write
an SIMD optimised version of Sha512 in cryptoxide.

Both SHA256 and SHA512 algorithms are only partially optimisable with SIMD.

Related to Blake2b and Blake2s algorithm, while at the default level
performance is mostly equivalent, the true difference happens at the AVX/AVX2
level, where cryptoxide manage a massive boost compared to blake2b. This is enabled
by the really nice design of [BLAKE2](https://www.blake2.net/).

With time permitting, the next step is to add more SIMD optimisation with different
algorithms and as new architecture achieved tier1 and wide support in rust,
hopefully getting other type of SIMD optimisations.

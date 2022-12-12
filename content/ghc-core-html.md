+++
title = "ghc core with style"
date = 2013-04-22
draft = false
[taxonomies]
tags = ["ghc","core","html","css","javascript"]
categories = ["programming"]
+++

After reading one too many time ghc core's output,
i've been itching to have a more interactive output.

<!-- more -->

ghc-core-html is the result of scratching my itch, and i
think it could be useful in general to anyone. It creates
a html output similar to what ghc-core does in a terminal,
but with also the following benefits:

* Symbols index at the beginning of the file
* Clickable symbols.
* Some hover popup: extra informations displayed on symbol.
* Foldable structures: hide what you don't need.
* Core output is (coarsely) parsed, not regex matched: better extensibility.

An example is worth thousand words:
[Example 1](http://tab.snarc.org/misc/ghc-core-html-example1.html)

It's really simple to use, and very similar to the well known ghc-core:

    > ghc-core-html Program.Hs > program.html
    > $browser program.html

There's lots of other things that can be added,
and style and javascript can easily be improved.
Pull requests gladly accepted at: [github repository](http://github.com/vincenthz/ghc-core-html)

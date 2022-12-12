+++
title = "unix memory"
date = 2014-02-25
draft = false
[taxonomies]
tags = ["haskell","unix","memory","mmap"]
categories = ["programming"]
+++

On unix system, we get access to syscalls that maps files or devices into
memory. The main syscall is mmap, but there's also some others syscalls in the
same family to handle mapped memories like mlock, munlock, mprotect, madvise,
msync.

<!-- more -->

Some limited mmap access is available through the
[mmap](http://hackage.haskell.org/package/mmap) or
[bytestring-mmap](http://hackage.haskell.org/package/bytestring-mmap) packages,
but both provide a high level access to those API.

To the rescue, I've released
[unix-memory](http://hackage.haskell.org/package/unix-memory).  This provide
low level access to all those syscalls. In some place, the API presented is
slightly better than the raw API.

This package is supposed to be ephemeral; The goal is to fold this package to
the venerable [unix](http://hackage.haskell.org/package/unix) package when this
becomes less experimental, more stable and is known to work on different
unixoid platforms.  If and when this happens, then this package will just
provide compatibility for old versions and eventually be deprecated.

Manipulating memory is unsafe in general, so don't expect any safety from this
package, by design. Also if you don't know what you're doing, don't use those
APIs; It's difficult to get right.

But it also allow interesting patterns when you cooperate with the operating system
to efficiently map files, and devices as virtual memory.

A simple example opening the "/dev/zero" device first memory page, and reading 4096 bytes from it:

~~~~~~~~~~~~ {.haskell .numberLines}
import System.Posix.IO
import System.Posix.Memory
import Control.Monad
import Control.Exception (bracket)
import Foreign.C.Types
import Foreign.Storable

bracket (openFd "/dev/zero" ReadWrite Nothing defaultFileFlags) closeFd $ \fd ->
  bracket (memoryMap Nothing 4096 [MemoryProtectionRead] MemoryMapPrivate (Just fd) 0)
          (\mem -> memoryUnmap mem 4096)
          (\mem -> mapM (peekElemOff mem) [0..4095])
~~~~~~~~~~~~


---
layout: post
title: redis and hiredis added to Gentoo Overlay
---

While I was writing some more code for [songride](https://github.com/xhochy/songride/)
I felt that I should have the latest redis version installed. After running
`emerge '>=dev-db/redis-2.4.17'` I was confronted with the following error message:

<pre>Re-emerging redis fails too with:
    LINK redis-benchmark
zmalloc.o: In function `zmalloc':
zmalloc.c:(.text+0x1a): undefined reference to `jmalloc'
zmalloc.c:(.text+0x2c): undefined reference to `jmalloc_usable_size'
zmalloc.o: In function `zcalloc':
zmalloc.c:(.text+0xd2): undefined reference to `jcalloc'
zmalloc.c:(.text+0xe4): undefined reference to `jmalloc_usable_size'
zmalloc.o: In function `zrealloc':
zmalloc.c:(.text+0x1a5): undefined reference to `jmalloc_usable_size'
zmalloc.c:(.text+0x1b5): undefined reference to `jrealloc'
zmalloc.c:(.text+0x1ed): undefined reference to `jmalloc_usable_size'
zmalloc.o: In function `zfree':
zmalloc.c:(.text+0x31b): undefined reference to `jmalloc_usable_size'
zmalloc.c:(.text+0x35f): undefined reference to `jfree'
collect2: ld returned 1 exit status
make[1]:  [redis-benchmark] Error 1
make[1]: Leaving directory `/var/tmp/portage/dev-db/redis-2.4.17/work/redis-2.4.17/src'
make:  [all] Error 2
* ERROR: dev-db/redis-2.4.17 failed (compile phase):
*   emake failed
</pre>

As I this is a problem affecting all Gentoo installations since the upgrade
from jemalloc from 3.1 to 3.2 the [Gentoo Bug #444796](https://bugs.gentoo.org/show_bug.cgi?id=444796)
was already solved and contained an ebuild for redis-2.4.17 that would install
fine. Sadly this ebuild is not yet available in the main portage tree so I
added it to [my overlay](https://github.com/xhochy/gentoo-overlay](https://github.com/xhochy/gentoo-overlay)).

Another useful project when using redis is [hiredis](https://github.com/redis/hiredis),
a minimalistic C client library for which an ebuild is available in
[Gentoo Bug #440226](https://bugs.gentoo.org/show_bug.cgi?id=440226) but not
in the main portage tree. The ebuild in my overlay contains [the modification](https://bugs.gentoo.org/show_bug.cgi?id=440226#c4)
suggested by alexbr.

Both of these ebuilds are GPLv2 licensed so there was no problem in adding
them to the overlay.

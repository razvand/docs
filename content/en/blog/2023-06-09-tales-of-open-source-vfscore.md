---
title: "Tales of Open Source: vfscore"
date: "2023-06-08T22:00:00+01:00"
author: "Răzvan Deaconescu"
tags: ["open source", "vfscore", "Unikraft", "OSv", "filesystem"]
---

{{< figure
    src="/assets/imgs/open-source.png"
    position="center"
>}}

The beauty of open source is the immediate availability of source code that you can use to create or improve your software.
Of course, this relies on [compatible licensing](https://en.wikipedia.org/wiki/License_compatibility).

Access to existing and, ideally, well tested and maintained software reduces design, development, and testing time, while giving software designers and developers the bandwidth to focus on the relevant parts of the software they are creating.

Unikraft, being inherently modular and configurable, is built with the idea of (open source) software reusability in mind.
This is obviously the case with external libraries such as [`lib-musl`](https://github.com/unikraft/lib-musl), [`lib-openssl`](https://github.com/unikraft/lib-openssl) or [`lib-boost`](https://github.com/unikraft/lib-boost);
and with applications such as [`app-sqlite`](https://github.com/unikraft/app-sqlite), [`app-nginx`](https://github.com/unikraft/app-nginx) or [`app-python3`](https://github.com/unikraft/app-python3).
But, more than patching and configuring existing libraries to be built with Unikraft, the Unikraft core itself (i.e. [the `unikraft` repository](https://github.com/unikraft/unikraft)) is reusing software components from other projects with compatible licensing (i.e. compatible with [BSD 3-clause license used by Unikraft](https://github.com/unikraft/unikraft/blob/staging/COPYING.md)).

One such component is [`vfscore`](https://github.com/unikraft/unikraft/tree/staging/lib/vfscore) (_Virtual Filesystem Core_).
`vfscore` is an internal library of Unikraft that provides a generic interface of system calls related to filesystem management.
It presents the API and support functions to handle typical filesystem-related concepts: files, directory entries, file operations (`open`, `read`, `write`), filesystem operations (`mount`, `umount`, `statfs`).

Looking at the copyright notices of files in `vfscore` (such as [`syscalls.c`](https://github.com/unikraft/unikraft/blob/staging/lib/vfscore/syscalls.c)) gives us a glimpse of how it came to be:

```text
 * Copyright (c) 2005-2007, Kohsuke Ohtani
 * Copyright (C) 2014 Cloudius Systems, Ltd.
 * Copyright (c) 2019, NEC Europe Ltd., NEC Corporation.
```

But let's look deeper.
Let's take a trip through the commit history of `vfscore` to find out how it came to be part of Unikraft.

## Finding the Source of `vfscore`

In order to check the commit history, let's first clone the source code of Unikraft:

```console
$ git clone https://github.com/unikraft/unikraft

$ cd unikraft/

$ ls -F
arch/  Config.uk  CONTRIBUTING.md  COPYING.md  include/  lib/  Makefile  Makefile.uk  plat/  README.md  support/  version.mk
```

`vfscore` is implemented in `lib/vfscore/`:

```console
$ cd lib/vfscore/

$ ls -F
Config.uk  eventpoll.c    extra.ld  file.c  include/  main.c       mount.c  README.md  stdio.c     syscalls.c  vfs.h
dentry.c   exportsyms.uk  fd.c      fops.c  lookup.c  Makefile.uk  pipe.c   rootfs.c   subr_uio.c  task.c      vnode.c
```

Now, let's see when the first commit came to be part of `vfscore`.
While inside the `lib/vfscore/` directory, we list the relevant commit history:

```console
$ git log -- .
```

We can scroll through the history and go to the first commit, but it's easiest to just extract the first commit and then show it:

```console
$ git log --oneline -- . | tail -1
146c13e0 lib/vfscore: introduce vfscore

$ git show 146c13e0
commit 146c13e043c56d3f62a1eec323821ad3ff0dd487
Author: Yuri Volchkov <yuri.volchkov@neclab.eu>
Date:   Fri Jun 15 02:35:20 2018 +0200

    lib/vfscore: introduce vfscore

    The initial implementation of the vfs abstraction layer. The task of
    the library is to allocate file descriptors, and redirect generic
    calls to the responsible library.
```

So `vfscore` was introduced in February 2018 by Yuri Volchkov.
We can checkout to that commit to see how `vfscore` looked at that point:

```console
$ ls -F
Config.uk  eventpoll.c    extra.ld  file.c  include/  main.c       mount.c  README.md  stdio.c     syscalls.c  vfs.h
dentry.c   exportsyms.uk  fd.c      fops.c  lookup.c  Makefile.uk  pipe.c   rootfs.c   subr_uio.c  task.c      vnode.c

$ git checkout -b init-vfscore 146c13e0
Switched to a new branch 'init-vfscore'

$ ls -F
export.syms  fd.c  file.c  include/  Makefile.uk
```

These files were written from scratch, nothing is ported from somewhere else.
We need to find other commits.

We get back to the `staging` branch, and we inspect the history of commits again:

```console
$ git checkout staging

$ git log -- .
[...]
commit 54e65bbb61ba8123682aac59dd3f14384c41ed6b
Author: Yuri Volchkov <yuri.volchkov@neclab.eu>
Date:   Mon Feb 18 15:54:22 2019 +0100

    lib/vfscore: Initial import of OSv vfs

    The code is imported as is.

    Commit f1f42915a33bebe120e70af1f32c1a4d92bac780
[...]
```

[Commit `54e65bbb`](https://github.com/unikraft/unikraft/commit/54e65bbb) is the one that introduced the `vfs` from [OSv](https://github.com/cloudius-systems/osv).
This happened in February 2019, one year later than the initial commit.
We also know the commit from [OSv](https://github.com/cloudius-systems/osv): `f1f42915`.
Unfortunately, that commit doesn't seem to be part of the [OSv commit history](https://github.com/cloudius-systems/osv/commits/master).

But we can checkout commit `54e65bbb` to see what `vfscore` looked like at that point:

```console
$ git checkout -b import-osv-vfs 54e65bbb
Switched to a new branch 'import-osv-vfs'

$ ls -F
Config.uk  dentry.c  exportsyms.uk  fd.c  file.c  fops.c  include/  lookup.c  main.c  Makefile.uk  mount.c  stdio.c  subr_uio.c  syscalls.c  task.c  vfs.h  vnode.c
```

And we can see quite a bunch of them are from Cloudius Systems / OSv:

```console
$ grep -r  Cloudius .
./include/vfscore/dentry.h: * Copyright (C) 2014 Cloudius Systems, Ltd.
./include/vfscore/prex.h: * Copyright (C) 2013 Cloudius Systems, Ltd.
./dentry.c: * Copyright (C) 2014 Cloudius Systems, Ltd.
./fops.c: * Copyright (C) 2013 Cloudius Systems, Ltd.
./syscalls.c: * Copyright (C) 2013 Cloudius Systems, Ltd.
./main.c: * Copyright (C) 2013 Cloudius Systems, Ltd.
```

Of course, a lot of things happened since the introduction of `vfs` components from OSv to `vfscore` in Unikraft:

```console
$ git log --oneline HEAD...54e65bbb -- . | wc -l
245

$ git log --shortstat --oneline HEAD...54e65bbb -- .  | grep -o '[0-9]\+ insertions' | cut -d ' ' -f 1 | paste -d '+' -s | bc
9009

$ git log --shortstat --oneline HEAD...54e65bbb -- .  | grep -o '[0-9]\+ deletions' | cut -d ' ' -f 1 | paste -d '+' -s | bc
5204
```

As of this writing, `245` commits have dealt with `vfscore`, with `9009` lines added and `5204` lines removed.
So quite a number of changes have happened since the introduction of `vfs` from OSv.
As is the case with open source, you start from existing code (in our case OSv), and then you adapt it to your needs.

What we know so far:

* In February 2019, Unikraft imported [`vfs` from OSv](https://github.com/cloudius-systems/osv/tree/master/fs/vfs) part of [Unikraft's `vfscore` implementation](https://github.com/unikraft/unikraft/tree/staging/lib/vfscore), with commit [`54e65bbb`](https://github.com/unikraft/unikraft/commit/54e65bbb).
* There is also a tag `Copyright (c) 2005-2007, Kohsuke Ohtani` that we want to find out more about.
  As we'll see below, that was itself imported in `OSv`.

## Digging Deeper: OSv

Let's clone the OSv source code and let's look into its `vfs` implementation:

```console
$ git clone https://github.com/cloudius-systems/osv

$ cd osv/fs/vfs/

$ ls -F
kern_descrip.cc  main.cc      vfs_bdev.cc  vfs_conf.cc    vfs_fops.cc  vfs_id.h       vfs_mount.cc     vfs_task.cc
kern_physio.cc   subr_uio.cc  vfs_bio.cc   vfs_dentry.cc  vfs.h        vfs_lookup.cc  vfs_syscalls.cc  vfs_vnode.cc
```

We can see that the `mount.c`, `vnode.c`, `syscalls.c`, `lookup.c` and other files in [Unikraft's `vfscore`](https://github.com/unikraft/unikraft/tree/staging/lib/vfscore) have been imported from the corresponding files in OSv's `vfs`: `vfs_mount.cc`, `vfs_vnode.cc`, `vfs_syscalls.cc`, `vfs_lookup.cc`.

OSv features a [BSD 3-clause license](https://github.com/cloudius-systems/osv/blob/master/LICENSE) making its source code, such as `vfs`, reusable in Unikraft.

We look for copyright notices in the `vfs` implementation:

```console
$ grep -r Copyright .
./vfs_id.h: * Copyright (C) 2020 Waldemar Kozaczuk
./vfs_dentry.cc: * Copyright (C) 2014 Cloudius Systems, Ltd.
./vfs_dentry.cc: * Copyright (c) 2005-2007, Kohsuke Ohtani
./kern_descrip.cc: * Copyright (C) 2013 Cloudius Systems, Ltd.
./vfs.h: * Copyright (c) 2005-2007, Kohsuke Ohtani
./subr_uio.cc: * Copyright (c) 1982, 1986, 1991, 1993
./main.cc: * Copyright (C) 2013 Cloudius Systems, Ltd.
./main.cc: * Copyright (c) 2005-2007, Kohsuke Ohtani
./vfs_mount.cc: * Copyright (c) 2005-2007, Kohsuke Ohtani
./vfs_task.cc: * Copyright (c) 2007, Kohsuke Ohtani All rights reserved.
./vfs_syscalls.cc: * Copyright (C) 2013 Cloudius Systems, Ltd.
./vfs_syscalls.cc: * Copyright (c) 2005-2007, Kohsuke Ohtani
./vfs_fops.cc: * Copyright (C) 2013 Cloudius Systems, Ltd.
./vfs_bdev.cc: * Copyright (C) 2013 Cloudius Systems, Ltd.
./vfs_conf.cc: * Copyright (C) 2013 Cloudius Systems, Ltd.
./vfs_conf.cc: * Copyright (c) 2005-2007, Kohsuke Ohtani
./vfs_bio.cc: * Copyright (c) 2005-2007, Kohsuke Ohtani
./kern_physio.cc: * Copyright (C) 2013 Cloudius Systems, Ltd.
./vfs_vnode.cc: * Copyright (c) 2005-2008, Kohsuke Ohtani
./vfs_lookup.cc: * Copyright (c) 2005-2007, Kohsuke Ohtani
```

So we see that we have `Kohsuke Ohtani`, also featured in Unikraft's `vfscore`, as an author for parts of OSv's `vfs`.
Let's see how that came to be;
let's look at the commit history of `vfs`, similar to our approach for Unikraft's `vfscore`:

```console
$ git log --oneline -- .

$ git log --oneline -- . | tail -1
47ec18e1 import the prex 0.9.0 VFS
```

[Commit `47ec18e1`](https://github.com/cloudius-systems/osv/commit/47ec18e1), by Christoph Hellwig, is the one that introduced `vfs` in OSv from a project called `Prex`, version `0.9.0`.
This happened in January 2013, 6 years before OSv's `vfs` was introduced in Unikraft's `vfscore`.
Similar to Unikraft's `vfscore`, OSv's `vfs` suffered changes since its introduction of `Prex` source code:

```console
$ git log --oneline -- . | wc -l
449
```

`449` commits related to VFS have been added to the OSv codebase ever since.

We can try to see the commit in the OSv codebase that was used as a snapshot to introduce `vfscore` in Unikraft.
The `vfscore`-related commit importing OSv's `vfs` is [`54e65bbb`](https://github.com/unikraft/unikraft/commit/54e65bbb).
It was done in February 2019, and it mentions a non-existing `OSv` commit: `f1f42915`.
We use the February 2019 date to identify a related commit in OSv.
We find commit [`eb28d3bd`](https://github.com/cloudius-systems/osv/commit/eb28d3bd), in October 2018, which should be the one used for importing OSv's `vfs` in Unikraft's `vfscore`.

Let's checkout that commit in the OSv code, and get information about the source code files in `fs/vfs/`:

```console
$ git checkout -b possible-unikraft-import eb28d3bd9395d25476f8aaa173a308f87b90b263
Switched to a new branch 'possible-unikraft-import'

$ ls
kern_descrip.cc  main.cc      vfs_bdev.cc  vfs_conf.cc    vfs_fops.cc  vfs_lookup.cc  vfs_syscalls.cc  vfs_vnode.cc
kern_physio.cc   subr_uio.cc  vfs_bio.cc   vfs_dentry.cc  vfs.h        vfs_mount.cc   vfs_task.cc

$ ls -l
total 180
-rw-rw-r-- 1 razvan razvan  5565 Jun  9 00:52 kern_descrip.cc
-rw-rw-r-- 1 razvan razvan  3060 Jun  9 00:52 kern_physio.cc
-rw-rw-r-- 1 razvan razvan 54732 Jun  9 00:52 main.cc
-rw-rw-r-- 1 razvan razvan  2655 Jun  9 00:52 subr_uio.cc
-rw-r--r-- 1 razvan razvan  2565 Feb 14  2021 vfs_bdev.cc
-rw-r--r-- 1 razvan razvan  7890 Feb 14  2021 vfs_bio.cc
-rw-rw-r-- 1 razvan razvan  2612 Jun  9 00:52 vfs_conf.cc
-rw-r--r-- 1 razvan razvan  6319 Feb 14  2021 vfs_dentry.cc
-rw-rw-r-- 1 razvan razvan  3834 Jun  9 00:52 vfs_fops.cc
-rw-r--r-- 1 razvan razvan  6181 Feb 14  2021 vfs.h
-rw-r--r-- 1 razvan razvan  9490 Feb 14  2021 vfs_lookup.cc
-rw-rw-r-- 1 razvan razvan 11018 Jun  9 00:52 vfs_mount.cc
-rw-rw-r-- 1 razvan razvan 29814 Jun  9 00:52 vfs_syscalls.cc
-rw-rw-r-- 1 razvan razvan  3941 Jun  9 00:52 vfs_task.cc
-rw-rw-r-- 1 razvan razvan 10558 Jun  9 00:52 vfs_vnode.cc

$ md5sum vfs_lookup.cc vfs_mount.cc vfs_syscalls.cc vfs_vnode.cc vfs_fops.cc vfs_dentry.cc vfs_task.cc vfs.h subr_uio.cc 
db179a920699d75d90f4f28a962788a7  vfs_lookup.cc
c0d6c8e9b4fb1f2630a1860a354780bd  vfs_mount.cc
85212639e5aeb6450c0909a11ff135dd  vfs_syscalls.cc
e83966c6c54b31c626c0cd7dcba5973f  vfs_vnode.cc
568c11a0f14679310dee5a1c4facbdb7  vfs_fops.cc
fa8de7a83f6fb9a8c931c51d5a58f37d  vfs_dentry.cc
69c6ac0438dd233a26a4e095d0d6a55b  vfs_task.cc
a565e675ebe0b35b2423ef3b52618138  vfs.h
bb097cd088409bb5b9e8d0656bc349de  subr_uio.cc
```

And let's do something similar to the files in Unikraft's `vfscore` at the time of the import (commit [`54e65bbb`](https://github.com/unikraft/unikraft/commit/54e65bbb)):

```console
$ git checkout import-osv-vfs
Switched to branch 'import-osv-vfs'

$ ls -l
total 180
-rw-rw-r-- 1 razvan razvan   116 Jun  9 00:52 Config.uk
-rw-rw-r-- 1 razvan razvan  6319 Jun  9 00:52 dentry.c
-rw-rw-r-- 1 razvan razvan    85 Jun  9 00:52 exportsyms.uk
-rw-rw-r-- 1 razvan razvan  3264 Jun  9 00:52 fd.c
-rw-rw-r-- 1 razvan razvan  2874 Jun  9 00:52 file.c
-rw-rw-r-- 1 razvan razvan  3834 Jun  9 00:52 fops.c
drwxrwxr-x 3 razvan razvan  4096 Jun  9 00:52 include
-rw-rw-r-- 1 razvan razvan  9490 Jun  9 00:52 lookup.c
-rw-rw-r-- 1 razvan razvan 54717 Jun  9 00:52 main.c
-rw-rw-r-- 1 razvan razvan   243 Jun  9 00:52 Makefile.uk
-rw-rw-r-- 1 razvan razvan 11018 Jun  9 00:52 mount.c
-rw-rw-r-- 1 razvan razvan  3121 Jun  9 00:52 stdio.c
-rw-rw-r-- 1 razvan razvan  2655 Jun  9 00:52 subr_uio.c
-rw-rw-r-- 1 razvan razvan 29814 Jun  9 00:52 syscalls.c
-rw-rw-r-- 1 razvan razvan  3941 Jun  9 00:52 task.c
-rw-rw-r-- 1 razvan razvan  6181 Jun  9 00:52 vfs.h
-rw-rw-r-- 1 razvan razvan 10558 Jun  9 00:52 vnode.c

$ md5sum lookup.c mount.c syscalls.c vnode.c fops.c dentry.c task.c vfs.h subr_uio.c
db179a920699d75d90f4f28a962788a7  lookup.c
c0d6c8e9b4fb1f2630a1860a354780bd  mount.c
85212639e5aeb6450c0909a11ff135dd  syscalls.c
e83966c6c54b31c626c0cd7dcba5973f  vnode.c
568c11a0f14679310dee5a1c4facbdb7  fops.c
fa8de7a83f6fb9a8c931c51d5a58f37d  dentry.c
69c6ac0438dd233a26a4e095d0d6a55b  task.c
a565e675ebe0b35b2423ef3b52618138  vfs.h
bb097cd088409bb5b9e8d0656bc349de  subr_uio.c
```

As expected, source code files are either identical or very similar (`main.c`).
So, indeed, commit [`eb28d3bd`](https://github.com/cloudius-systems/osv/commit/eb28d3bd) in OSv was used to import `vfs` in Unikraft's `vfscore`, in commit [`54e65bbb`](https://github.com/unikraft/unikraft/commit/54e65bbb).

Getting back to the origin of OSv's `vfs` (Prex version `0.9.0`), searching the web reveals [the page of the Prex project](https://prex.sourceforge.net), authored by Kohsuke Ohtani.
Prex is an open source real-time operating system [designed as a microkernel](https://prex.sourceforge.net/doc/overview.html).

Prex has been developed for close to 5 years:
version `0.1` was released in March 2005;
version `0.9.0`, used as the basis for OSv's `vfs`, was released in October 2009.
So it was more than 3 years after the release of Prex `0.9.0` that some of its parts were introduced in OSv.

We couldn't find a repository for Prex.
So, to find more about the initial code that was imported in OSv, we'll have to do manual download and investigation of [Prex source code](https://prex.sourceforge.net/).

## The Source Material: Prex

We download Prex `0.9.0` and check its VFS implementation in the filesystem server (`usr/server/fs/vfs/`):

```console
$ wget http://prdownloads.sourceforge.net/prex/prex-0.9.0.tar.gz

$ tar xf prex-0.9.0.tar.gz

$ cd prex-0.9.0/

$ ls -F
bsp/  conf/  configure*  doc/  include/  Makefile*  mk/  sys/  usr/

$ cd usr/server/fs/vfs/

$ ls
main.c  Makefile  vfs_bio.c  vfs_conf.c  vfs.h  vfs_lookup.c  vfs_mount.c  vfs_security.c  vfs_syscalls.c  vfs_task.c  vfs_vnode.c
```

We see that a larger part of the source components that were part of Unikraft's `vfscore`, via OSv's `vfs`, are part of Prex's `vfs`: `vfs_mount.c`, `vfs_lookup.c`, `vfs_vnode.c`, `vfs_syscalls.c`.

Prex features a [BSD style license](https://prex.sourceforge.net/doc/license.html), making its code reusable in OSv and then in Unikraft.

Browsing the source code reveals that it was written from scratch.
This is as far as we go with respect to source code origin.
The only additional reference is mentioned in `vfs_bio.c`: the book [The Design of the UNIX Operating System (by Maurice Bach)](https://www.amazon.com/Design-UNIX-Operating-System/dp/0132017997).

What we can determine is when in time did `vfs` get implemented in Prex, by checking [different versions](https://prex.sourceforge.net/downloads.html).
As the description for version `0.5.0` says `File system`, we check that version and the previous one (`0.4.3`):

```console
$ wget http://prdownloads.sourceforge.net/prex/prex-0.5.0.tar.gz

$ tar xf prex-0.5.0.tar.gz

$ ls -F prex-0.5.0/user/
arch/  bin/  include/  lib/  Makefile  sample/  server/  test/

$ ls prex-0.5.0/user/server/fs/vfs/
bio.c  conf.c  lookup.c  main.c  Makefile  mount.c  syscalls.c  tcb.c  vfs.h  vnode.c

$ ls -F prex-0.5.0/user/server/fs/
arfs/  devfs/  Makefile  ramfs/  vfs/

$ wget http://prdownloads.sourceforge.net/prex/prex-0.4.3.tar.gz

$ tar xf prex-0.4.3.tar.gz

$ ls prex-0.4.3/user/
arch/  bin/  include/  lib/  Makefile  sample/  test/
```

We that, indeed, the `vfs` implementation appeared in Prex `0.5.0`, released in June 2007.
Header files with `vfs`-related structure are present in Prex `0.4.3`, but no implementation was yet there.

## Full Story

So we now have the full story of the origin of Unikraft's `vfscore`, based on the amazing process of reusing open source code.
Initially implemented as part of the filesystem server in the [Prex real-time microkernel](https://prex.sourceforge.net/), it was then imported as [OSv's `vfs`](https://github.com/cloudius-systems/osv/tree/master/fs/vfs) and then added to [Unikraft's `vfscore`](https://github.com/unikraft/unikraft/tree/staging/lib/vfscore), over the span of about 14 years.

The timeline is:

* March 2005: Prex `0.1` is released by Kohsuke Ohtani
* June 2007: Initial `vfs` implementation is added to Prex `0.5.0`
* October 2009: Latest version of Prex (`0.9.0`) is released
* January 2013: Prex `0.9.0` `vfs` is imported in `OSv` as commit [`47ec18e1`](https://github.com/cloudius-systems/osv/commit/47ec18e1) by Christoph Hellwig
* February 2019: OSv `vfs` (at commit [`eb28d3bd`](https://github.com/cloudius-systems/osv/commit/eb28d3bd)) is added to Unikraft's `vfscore` as commit [`54e65bbb`](https://github.com/unikraft/unikraft/commit/54e65bbb) by Yuri Volchcov
* June 2023 (present day): 245 new commits with 9009 insertions and 5204 deletions have happened to Unikraft's `vfscore` implementation since the OSv import

{{< figure
    src="/assets/imgs/vfscore-timeline.png"
    position="center"
>}}

This has all been possible by having access to open source code with compatible licensing (BSD 3-Clause).
This code that made its way from Prex, via OSv, to Unikraft.
And, of course, it was also possible with constant features and improvements by Unikraft contributors to better the imported code.

As a noteworthy fact, the debugging facility in `vfscore`, with the [`DEBUG_VFS` macro](https://github.com/unikraft/unikraft/blob/staging/lib/vfscore/vfs.h#L51), the [`VFSDB_*` flags](https://github.com/unikraft/unikraft/blob/staging/lib/vfscore/vfs.h#L51) and [`DPRINTF` macro](https://github.com/unikraft/unikraft/blob/staging/lib/vfscore/vfs.h#L62), inconsistent with debugging mechanisms in Unikraft, is directly inherited from Prex `0.9.0`.
So, while existing source code can save developers time to focus on project-specific tasks, it also means that effort needs to be put to make it consistent (style, design, debugging mechanisms) to project guidelines.
As sometimes discussed in the Unikraft community, a revamp of `vfscore` is something to look into: improving its design (inherited from Prex) and making it consistent with Unikraft source code.

This post is part of a series titled `Tales of Open Source`, that highlight the way open source code makes its way and is improved in the Unikraft project.
Stay tuned for more!

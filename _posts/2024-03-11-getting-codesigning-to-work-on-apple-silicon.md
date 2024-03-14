---
layout: post
title: >
  "Killed: 9" – Getting codesigning to work with conda-pack on Apple Silicon (osx-arm64)
feature_image: "/images/priscilla-du-preez-4hifhaM4pVg-unsplash.jpg"
redirect_from:
 - /2023/03/11/getting-codesigning-to-work-on-apple-silicon.html
---

If your error message is simply "Killed: 9", you know you are in for a long debug session.
Sadly, in this case, even a debugger would not give you any further information.
Thus, this has been a problem that could only be fixed by having good insight and knowledge of the packaging percularities of the particular platform.

One of these programs that failed with the above-mentioned error message was `conda-pack`.
`conda-pack` is a tool to package `conda` environments into an archive that can be extracted later on at an arbitrary location.
Through a specialised activation script, the executable in the packed environment can then ??? from this arbitrary location.

While it has been working quite smoothly for Windows, Linux, and Intel-based macOS systems, it sadly failed with the very brief error message on all Apple Silicon-based macOS installations:

```
 <script>: line 3: 36276 Killed: 9               pip install --disable-pip-version-check --no-build-isolation --no-deps -e
ERROR conda.cli.main_run:execute(47): `conda run pip install --disable-pip-version-check --no-build-isolation --no-deps -e .` failed. (See above for error)
```

The special knowledge needed here to understand the problem is that the "magic" `conda-pack` is doing is replacing the environment prefix in any file (even binaries) where it is hardcoded on its first activation. This in turn means that it will modify file contents. This is an issue with Apple Silicon-based macOS systems as [Apple requires all binaries to be codesigned there](https://eclecticlight.co/2020/08/22/apple-silicon-macs-will-require-signed-code/). None of the other systems `conda-pack` runs on has this requirement.

To the end-user, this failure is reported simply on the command line as `Killed: 9`. One can find more information about this when looking simultaneously at the kernel logs: `sudo log stream --predicate 'sender = "kernel"'`. This produces an output that looks like the following:

```
2023-03-15 21:41:04.947799+0100 0xeee4bd   Default     0x0                  0      0    kernel: CODE SIGNING: cs_invalid_page(0x105048000): p=38578[python3.11] final status 0x23000200, denying page sending SIGKILL
2023-03-15 21:41:04.947811+0100 0xeee4bd   Default     0x0                  0      0    kernel: CODE SIGNING: process 38578[python3.11]: rejecting invalid page at address 0x105048000 from offset 0x224000 in file "/private/var/folders/km/wl0tx9197gz3__0_h7y9gfyr0000gn/T/tmpn5ygqmbc/lib/libcrypto.3.dylib" (cs_mtime:1678912860.971028164 == mtime:1678912860.971028164) (signed:1 validated:1 tainted:1 nx:0 wpmapped:0 dirty:0 depth:0)
```

The above error message states that the process we are executing tries to load a shared library `libcrypto.3.dylib` that has an invalid code signature and is thus aborted.

The creation of the initial conda environment worked smoothly as `conda` already has logic for recreating code signatures when it modifies a binary.
As `conda-pack` comes with its own independent script, it needed also to have the logic to create valid signatures for binaries it modifies.
The resulting fix has been merged as part of [conda-pack#257](https://github.com/conda/conda-pack/pull/257) and was released as `conda-pack==0.7.1`.

The essence of the fix is to run `codesign -s - -f <binary>` on all the binaries that are touched by `conda-pack`'s activation script.

This article is probably only relevant for a very small amount of people.
If I could help you with a problem, please reach out, I'm curious where other people stumble into this issue.

*Title picture: Photo by [Priscilla Du Preez](https://unsplash.com/photos/brown-pie-on-white-ceramic-bowl-4hifhaM4pVg?utm_content=creditCopyText&utm_medium=referral&utm_source=unsplash) on [Unsplash](https://unsplash.com/?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)*



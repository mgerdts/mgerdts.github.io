---
layout: post
title:  uefi debug in bhyve
date:   2018-02-25 20:00:00 -0600
tags: uefi bhyve
commentIssueId: 3
tweetWidgetId: 970353960732299264
---

While debugging a bug in [SmartOS bhyve](https://smartos.org/bugview/OS-6604) seen on Ubuntu guests, I needed to debug two areas that most people avoid: `grub` and `uefi`.  Debugging `grub` is not so bad, as you can at least add print statements and they appear on the console.  `uefi` was not quite as cooperative at first.

There are several things involved at build time:

- While running the uefi-edk2 build script, you need the command line argument `-DDEBUG_ON_SERIAL_PORT=TRUE`.  If building [uefi-edk2 in illumos-extra](https://github.com/joyent/illumos-extra/tree/master/uefi-edk2), this is already taken care of.
- You may need to tweak the `PcdDebugPrintErrorLevel` or `PcdDebugPropertyMask` in `BhyvePkg/BhyvePkgX64.dsc` to filter the particular types of messages you see.  I've not experimented with these.
- In `BhyvePkg/Csm/BhyveCsm16/GNUmakefile`, `DEBUG_PORT` needs to point to the proper [COM port](https://en.wikipedia.org/wiki/COM_(hardware_interface)).
- In `BhyvePkg/Csm/BhyveCsm16/Printf.c`, `DebugLevel` needs to be set to a level that gets the desired verbosity.

These last two bullets are taken care of with [this changeset](https://github.com/mgerdts/uefi-edk2/commit/eeeeb9c01ecab3d0830d2e00e43a1f9d8a408c2a).  Notice that it also changes `PcdDebugIoPort` in `BhyvePkg/BhyvePkg.dec`.  I had expected that that would be sufficient to change the debug port to `COM1`, but it seemed to have no effect.

Once I did this, I wasn't completely out of the woods, as I found that this [caused bhyve to crash](https://smartos.org/bugview/OS-6694) because its uart emulation only handled 1 or 2 byte operations.  uefi works 4 bytes at a time.  [Here's the fix](https://github.com/mgerdts/illumos-joyent/commit/ec04362e4d1964b7135cd7b70545ab97ce33882b).

By the time you are reading this, the `bhyve` bug will probably be long fixed.  To save you the hassle of building a debug uefi image, you can snag [the one I built](https://us-east.manta.joyent.com/mgerdts/public/blog/downloads/uefi-csm-rom-debug-com1-info.bin).  You can activate that for a single bhyve zone with:

```
cp .../uefi-csm-rom-debug-com1-info.bin /zones/$uuid/root
vmadm update $uuid bootrom=/uefi-csm-rom-debug-com1-info.bin
```

---
layout: post
title:  OmniOS bhyve guest
date:   2018-03-14 18:00:00 -0500
tags: omnios bhyve
commentIssueId: 7
tweet: <blockquote class="twitter-tweet"><p lang="en" dir="ltr">Thanks <a href="https://twitter.com/OmniOSce?ref_src=twsrc%5Etfw">@OmniOSce</a> for making it so easy for me to create a bhyve guest on SmartOS. <a href="https://t.co/tQXyl2NsFN">https://t.co/tQXyl2NsFN</a></p>&mdash; Mike Gerdts (@OMGerdts) <a href="https://twitter.com/OMGerdts/status/974089025878454274?ref_src=twsrc%5Etfw">March 15, 2018</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>
---

I had a very pleasant surprise today.  I have some work that I need to
contribute to illumos and figured that a little dogfooding would be good for me.
It seems as though two excellent distros for building illumos are OpenIndiana
and OmniOS.  Since I don't really need a desktop and I've not used OmniOS
before, OmniOS seemed like a great choice for the day.

After a bit of looking around at
[omniosce.org](https://omniosce.org/download.html) I came across something that
was almost too good to be true:
[omnios-bhyve.zfs.xz](https://downloads.omniosce.org/media/bloody/omnios-bhyve.zfs.xz)!

Let's take that for a spin:

```
# curl https://downloads.omniosce.org/media/bloody/omnios-bhyve.zfs.xz | xzcat | zfs recv zones/omnios
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  343M  100  343M    0     0  3374k      0  0:01:44  0:01:44 --:--:-- 4368k
# zfs clone zones/omnios@good zones/o1
# bhyve -m 16g -c 16 -l com1,stdio -P -H -s 1,lpc -s 3,virtio-blk,/dev/zvol/rdsk/zones/o1 -s 4,virtio-net-viona,o1 -l bootrom,/usr/share/bhyve/uefi-csm-rom.bin o1
```

OMG!  A few seconds later and I saw loader, and a few seconds after that I had a
login prompt.  Woot!

Well, let's make this work with the SmartOS bits that are almost ready for you
to take for a spin.  First, I'll turn this into an image that will work well
with `vmadm`.

```
# zfs rename zones/omnios@good zones/omnios@final
# zfs send zones/omnios@final | pigz > /zones/images/6c4da93e-cffd-e809-a352-e4e11de3b067/disk0.zfs.gz
# ls -l disk0.zfs.gz
total 1011562
-rw-r--r--   1 root     root     517860460 Mar 15 00:20 disk0.zfs.gz
-rw-r--r--   1 root     root         918 Mar 15 00:19 manifest.json
# digest -a sha1 disk0.zfs.gz
9d14866b9ec35443adcffc34fed54bd7f1e62700
# vi manifest.json
# imgadm install -m manifest.json -f disk0.zfs.gz
Installing image 6c4da93e-cffd-e809-a352-e4e11de3b067 (omnios-master-39b8863d01@20180301)
...-e809-a352-e4e11de3b067 [=========================================>] 100% 493.87MB  29.32MB/s    16s
Installed image 6c4da93e-cffd-e809-a352-e4e11de3b067 (omnios-master-39b8863d01@20180301)
[root@emy-17 ~]# imgadm list os=illumos
UUID                                  NAME                      VERSION   OS       TYPE  PUB
6c4da93e-cffd-e809-a352-e4e11de3b067  omnios-master-39b8863d01  20180301  illumos  zvol  2018-03-15
```

Next, shamelessly copy an early `vmadm create` payload to create a new vm.

```
# cp test.json omni1.json
# vi omni1.json
# vmadm create -f omni1.json
[some output that I lost in my haste.  Just the normal stuff that says things went well.]
# vmadm console $(vm omni1)
...
Copyright (c) 1983, 2010, Oracle and/or its affiliates. All rights reserved.
Copyright (c) 2017-2018 OmniOS Community Edition (OmniOSce) Association.
Loading smf(5) service descriptions: 131/131
Configuring devices.
NOTICE: vioif0: Got MAC address from host: 72:93:42:ef:32:a0
NOTICE: Csum enabled.
Applying initial boot settings...
Hostname: omniosce

omniosce console login:
```

Log in as `root` with no password, and I'm ready to configure networking, users,
etc.

```
root@omniosce:~# dladm show-link
LINK        CLASS     MTU    STATE    BRIDGE     OVER
vioif0      phys      1500   up       --         --
root@omniosce:~# ipadm create-if vioif0
root@omniosce:~# ipadm create-addr -T static -a 172.26.17.173/24 vioif0/v4
root@omniosce:~# ping 172.26.17.1
172.26.17.1 is alive
root@omniosce:~# route -p add default 172.26.17.1
add net default: gateway 172.26.17.1: entry exists
add persistent net default: gateway 172.26.17.1
root@omniosce:~# vi /etc/resolv.conf
root@omniosce:~# vi /etc/nsswitch.conf
...
```

Thanks much to (andyf?) for creating the image.

---
layout: post
title:  bhyve zones in SmartOS
date:   2018-03-05 20:00:00 -0600
tags: bhyve smartos zones
commentIssueId: 5
tweetWidgetId: 971975452981018624
---

The team at Joyent is bringing [bhyve](http://www.bhyve.org/) - The FreeBSD
Hypervisor - to SmartOS.  This includes
[Patrick Mooney](https://github.com/pfmooney),
[Hans Rosenfeld](https://github.com/hrosenfeld),
[Jerry Jelinek](https://github.com/jjelinek),
[John Levon](https://github.com/jlevon) and [me](https://github.com/mgerdts),
with help and encouragement from many other Joyeurs.

Most of my work up to this time on bhyve has been focused on integration with
zones and other parts of SmartOS.  As such my comments will focus on these areas
that I know well.  I'll (mostly) leave it to others to speak to the areas they
know best.

Much of what is described below is the current state as of early March, 2018.
Many improvements are expected.

## Why bhyve?

To [paraphrase the boss](https://twitter.com/bcantrill/status/966722687102930945),
we are doing this to replace KVM because it is better for us.  More to the
point, our KVM implementation lacks PCI pass through and the network performance
lags the requirements of demanding applications.  As the [twitter thread
says](https://twitter.com/phlm011/status/966824721487712256), FreeBSD and
illumos have a history of cross-pollination.  When examining each hypervisor and
the communities around them, it was determined that we are best served by
putting our efforts into bhyve.

## Why use zones?

To be clear, `bhyve` (the command) works just fine without zones.  Realistically
you need to wrap it with something because you really don't want to have to
manually start a bunch of VMs with a dozen or so command line arguments.  The
wrapper could be a shell script dropped in `/etc/rc*.d/`, an SMF service, a
zone, or something else.  Using zones has a few advantages:

- Each zone's configuration...
  - is a handy place to describe vcpus, memory, virtual disks, networks, serial
    ports, etc., that map to various `bhyve` command line arguments.
  - can contain resource control specifications, ensuring the bhyve instance
    running within a zone doesn't consume more than its share.
  - can define a reduced set of privileges.
- In the event of a guest escape into host user space, the attacker's code runs
  with reduce privilege (e.g. no `fork()` or `exec()`, in a namespace that lacks
  most devices, and has no networking.
- All of the other types of virtualization supported by SmartOS are based on
  zones.  This takes care of:
  - Configuration
  - Installation
  - Booting and shutting down
  - Snapshots
  - Migration (cold only)

## Anatomy of a bhyve zone

A bhyve zone is defined by the
[`config.xml`](https://github.com/joyent/illumos-joyent/tree/master/usr/src/lib/brand/bhyve/config.xml)
and
[`platform.xml`](https://github.com/joyent/illumos-joyent/tree/master/usr/src/lib/brand/bhyve/platform.xml),
which are installed in `/usr/lib/brand/bhyve/`.

In `platform.xml`, you can see there are a number of file systems that are
mounted.  Notably, `/lib` and `/usr` are read-only `lofs` mounts from the global
zone.  There are a number of `tmpfs` file systems mounted because the various
parts of the system do not yet agree on where the system's temporary files
belong.  The list of device files is much longer than it should be.  This
should be fixed via [OS-6728](https://smartos.org/bugview/OS-6728).

In `config.xml`, you will see that `initname` is set to
`/usr/lib/brand/bhyve/zhyve`.  The `zhyve` (notice **z**) process is started in
the zone by the kernel, but without any command line arguments.  The command
line is formed by the `boot` brand hook, implemented by
[`boot.c`](https://github.com/joyent/illumos-joyent/blob/master/usr/src/lib/brand/bhyve/zone/boot.c).
This `boot` program runs in the global zone as a child of `zoneadmd` just before
`zhyve` is started.  It transforms
[a bunch of environment variables](https://github.com/joyent/illumos-joyent/blob/master/usr/src/cmd/zoneadmd/zoneadmd.c#L819)
set by `zoneadmd` into a normal `bhyve` command line.  The command line is
stored in a packed nvlist in `<zoneroot>/tmp/zhyve.cmd`, which is then read by
`zhyve`.

`/usr/lib/brand/bhyve/zhyve` is a hard link to `/usr/sbin/bhyve`.  When the
program is started as `bhyve`, it does the normal command line processing.  When
started as `zhyve` it looks for the aforementioned packed nvlist, then calls the
normal `bhyve` `main()`
([renamed to `bhyve_main()`](https://github.com/joyent/illumos-joyent/blob/master/usr/src/cmd/bhyve/Makefile.com#L91))
with those command line arguments.  See
[`zhyve.c`](https://github.com/joyent/illumos-joyent/blob/master/usr/src/cmd/bhyve/zhyve.c).

Each bhyve zone needs to have access to `/dev/vmmctl`, `/dev/vm/<vmname>`, and
`/dev/viona` (virtio network adapter).  `/dev/vmmctl` and `/dev/viona` are
implemented as `devfsadmd` plugins, which means that they will trigger the load
of the `vmm` and `viona` kernel modules on access.  As `vmm` loads, an `sdev`
(aka devnames) plugin causes `/dev/vmm` to be created.  As is normally the case,
with directories in `/dev`, the contents of `/dev/vmm` are populated on demand.
`/dev/vmm` only contains the entries for a particular zone's VMs.

For now, VM names must be unique.  We accomplish this by setting the VM name to
`SYSbhyve-<debug-id>`.  For those not familiar with debug-id, it is a numeric
identifier that stays with a zone across reboots.

```
> ::walk zone | ::print zone_t zone_name zone_id zone_did
zone_name = 0xfffffffffbf3ff40 "global"
zone_id = 0
zone_did = 0
zone_name = 0xffffd028c5ffbe00 "793c85e9-1dd5-ef28-ae08-e41a5942c946"
zone_id = 0x5
zone_did = 0x31
> !zoneadm -z 793c85e9-1dd5-ef28-ae08-e41a5942c946 reboot
> ::walk zone | ::print zone_t zone_name zone_id zone_did
zone_name = 0xfffffffffbf3ff40 "global"
zone_id = 0
zone_did = 0
zone_name = 0xffffd028d5fa8c00 "793c85e9-1dd5-ef28-ae08-e41a5942c946"
zone_id = 0x6
zone_did = 0x31
```

## Configuration

The essential part of one of my bhyve zones appears below.  I've cut out many
lines for the sake of brevity (haha).

```
# zonecfg -z 1d5b8e7c-c004-4f69-bf0c-c98918e35bd5 export | cat -n
     1	create -b
     2	set zonepath=/zones/1d5b8e7c-c004-4f69-bf0c-c98918e35bd5
     3	set brand=bhyve
     4	set autoboot=true
     5	set limitpriv=default,-file_link_any,-net_access,-proc_fork,-proc_info,-proc_session
     6	set ip-type=exclusive
     7	set uuid=1d5b8e7c-c004-4f69-bf0c-c98918e35bd5
```

Lines 2 - 7 set properties in the global scope.  Setting `brand=bhyve` tells the
zones framework that this is a bhyve zone.  Line 5 takes away some privileges
that this zone does not need - don't get confused between what code running on
on the SmartOS instance can do and what can be done in the guest OS.  Line 6
indicates that each `net` resource will have its own network stack.  The other
value for `ip-type`, `shared`, is not supported with bhyve.

```
     8	add net
     9	set physical=net0
    10	set mac-addr=c2:ba:60:58:bf:78
    11	set vlan-id=3317
    12	set global-nic=external
    13	add property (name=gateway,value="172.26.17.1")
    14	add property (name=gateways,value="172.26.17.1")
    15	add property (name=netmask,value="255.255.255.0")
    16	add property (name=ip,value="172.26.17.211")
    17	add property (name=ips,value="172.26.17.211/24")
    18	add property (name=model,value="virtio")
    19	add property (name=primary,value="true")
    20	end
```

Lines 8 - 20 configure networking.  Lines 9 - 12 provide parameters that are
used during automatic creation of a vnic.  By default, the network disallows
spoofing at layer 2 and layer 3.  Only packets originating from the MAC address
found in line 10 and an IP address listed in line 17 are allowed to pass out of
the guest.

Due to transitional issues in SmartOS there are some extraneous properties (e.g.
`ip`, `gateway`).

```
    21	add device
    22	set match=/dev/zvol/rdsk/zones/1d5b8e7c-c004-4f69-bf0c-c98918e35bd5/disk0
    23	add property (name=boot,value="true")
    24	add property (name=model,value="virtio")
    25	add property (name=media,value="disk")
    26	add property (name=image-size,value="10240")
    27	add property (name=image-uuid,value="e0ef873d-6423-4002-90db-1ecd1d99dff2")
    28	end
```

These next lines describe a disk that resides on a zfs volume.  Of all these
lines, only lines 22 and 23 are currently used by the bhyve brand.  Currently
`boot.c` turns each `device` resource into a virtio disk.  Two other disks are
not shown as they don't show anything unique.

```
    45	add rctl
    46	set name=zone.cpu-shares
    47	add value (priv=privileged,limit=100,action=none)
    48	end
```

Various resource controls can be specified, just as they are for any other zone.
I show only one of several that are defined automatically by `vmadm create`.

```
   105	add attr
   106	set name=resolvers
   107	set type=string
   108	set value=8.8.8.8,8.8.4.4
   109	end
```

Some networking properties that are common to all `net`s are defined with an
`attr` resource.

```
   120	add attr
   121	set name=ram
   122	set type=string
   123	set value=2048
   124	end
   135	add attr
   136	set name=vcpus
   137	set type=string
   138	set value=1
   139	end
```

Likewise, the amount of memory (MiB), and number of vcpus are also defined in
attr resources.  I've rearranged some lines to provide a more logical grouping
here.

```
   125	add attr
   126	set name=com1
   127	set type=string
   128	set value=/dev/zconsole
   129	end
   130	add attr
   131	set name=com2
   132	set type=string
   133	set value=socket,/tmp/vm.ttyb
   134	end
```

We also [ab]use `attr` resources to map the guest's first two serial ports to
paths in the host (but within the zone).  If the guest redirects its console to
the first serial port, `zlogin -C` or `vmadm console` in the global zone can
be used to access the guest's console.

The second serial port is set to a path that is connected to the SmartOS
metadata server.

The other LPC device, `bootrom` can also be configured with a `attr
name=bootrom`.  If not specified, the system defaults to the UEFI CSM bootrom.

## Guest network configuration

When kvm/qemu is used on SmartOS, a DHCP server that is built into qemu provides
guest networking configuration.  No such DHCP server is built into bhyve and we
don't have immediate plans to change that.  Instead, we rely on guests to use
the [metadata API](https://docs.joyent.com/private-cloud/instances/using-mdata)
to retrieve configuration from `sdc:hostname`, `sdc:nics`, and `sdc:resolvers`.

The metadata approach is already used by recent Ubuntu images through their use
of the SmartOS Data Source in cloud-init.  As part of my work on bhyve, I've
fixed a [few bugs](https://github.com/mgerdts/cloud-init/commits/bug-1667735)
in `DataSourceSmartOS` to make it more tolerant of line noise and various other
errors.

## Future work

As a reminder, this blog entry is focused on items related to the brand.  The
list of items for `bhyve`, `vmm`, etc. is in addition to what is mentioned
below.

### Real soon now, please

At this time, the bhyve code is not fully integrated into SmartOS.  That should
be complete in the first half of March, 2018.  There are a number of small
fixes that various developers have waiting in the wings that will follow this
integration.

Perhaps not waiting in the wings, but needed soon, is a way to configure PCI
pass through (PPT).  It is not clear whether this will fit into the existing
`device` resource or if it will be shoehorned into yet another `attr` resource.
Other features that are likely needed involve being able to express dedicated
host cpus and binding vcpu threads to those host cpus.

### Helpful, but not a blocker

Debugging failures is sometimes difficult, as the failures can happen in
`zoneadmd`, `zhyve`, the `vmm` driver, and perhaps other places.  This can be
made much easier by being a bit more verbose (e.g. `vmm` logs no messages for
most failures) and having messages appear in time-stamped logs.  I previously
did [analogous work](https://blogs.oracle.com/zoneszone/zones-console-logs).
See [OS-6718](https://smartos.org/bugview/OS-6718).

### Upstream to illumos

Prior to upstreaming to illumos, we need to transition from being a consumer of
the zones framework to improving it.  The work that has been done on the bhyve
brand copies a lot of what has been done for the kvm brand, an unabashed
consumer of the zones framework.  That means that the bhyve brand has
implemented the minimum that is needed to work on SmartOS and is not integrated
into the zones framework in a way that is suitable for inclusion in illumos.

Specific areas that need work prior to upstreaming include:

- The `attr` resources used to configure `vcpus`, `ram`, LPC devices, etc. need
  to transitioned to appropriate resources and properties.
  - This means we need to alter [zonecfg.dtd.1](https://github.com/joyent/illumos-joyent/blob/master/usr/src/lib/libzonecfg/dtd/zonecfg.dtd.1)
    to add new resource types and/or properties.
  - And that implies that we need to introduce a way to selectively enable
    resource types and properties for particular brands.  See
    [RFD 122](https://github.com/joyent/rfd/blob/master/rfd/0122/README.md).
- The automatic creation of VNICs is handled in SmartOS-specific brand hooks -
  shell scripts that are called during state changes.  This work should be done
  in `zoneadmd` via calls into `libdladm` and related libraries.
- Support booting from iso or usb installation media.
- Most brand hooks need to be rewritten to not be specific to SmartOS.  In
  particular, we need to support `zoneadm install` in a meaningful way.
- Improve detection of resource (e.g. RAM) allocation failures during bhyve
  initialization.  See [OS-6717](https://smartos.org/bugview/OS-6717).
- The bhyve brand relies on other OS components (e.g. `sdev` (aka devnames)
  plugins) that are not yet in illumos.

Watch [RFD 121](https://github.com/joyent/rfd/blob/master/rfd/0121/README.md),
others for details in this space.

## Collaboration with FreeBSD

We are striving to keep the code that we get from FreeBSD as close to the
FreeBSD code as possible.  Both communities will benefit from being able to stay
in sync with the latest features and fixes.

The following are some of the areas where we have diverged and will look for
opportunities to contribute to FreeBSD after we catch our breath from our
initial pushes into [illumos-joyent](https://github.com/joyent/illumos-joyent).

### Specialized startup and ongoing interaction

There are some times where we need unique functionality. One key example is
`zhyve` for special handling of command line arguments, described above.
`zhyve` is also responsible for redirecting `stdout` and `stderr` to a log file.

To meet a variety of needs ([OS-6718](https://smartos.org/bugview/OS-6718), get
rid of bouncing command line into `zhyve.args` via
[`boot.c`](https://github.com/joyent/illumos-joyent/blob/master/usr/src/lib/brand/bhyve/zone/boot.c),
eventual hot plug of devices, live migration, etc.),
we would like for `zoneadmd` to be able to interact with `zhyve`.  It may be
that extensions of the existing `bhyvectl` mechanisms is the right way, or it
could be that `zhyve` gains a door server which is used by `zoneadmd` to pass
commands and file descriptors.

### State change notifications.

As described in [OS-6717](https://smartos.org/bugview/OS-6717), `zoneadmd` needs
to know when `zhyve` shifts from allocating resources for the guest to running
guest code.  There are other state transitions that are interesting as well.
For instance, if we see that `zhyve` exited, it would be helpful to know whether
that was due to an explicit request from within the guest, due to receiving a
signal, or due to a failure in `zhyve`.

If `zoneadmd` (or something else) is notified about pending and completed state
changes, this could help in ensuring that we take appropriate action - or no
action at all - when `zhyve` exits.

The workaround for the immediate problem is for `zhyve` to [set a
callback](https://github.com/joyent/illumos-joyent/commit/c34f7c2c834dc5a9b871356c0f85214dc25c5881#diff-c469bedead3260c7298f80c05adcbda8R166)
that is called once `zhyve` initialization is complete.

### SMBIOS hacking

For `cloud-init` to recognize that it should use `DataSourceSmartOS`, we need to
set the [SMBIOS product to "SmartDC HVM"](https://github.com/mgerdts/illumos-joyent/commit/38d374d18abcd1f7e26f0ae49f5599b2667d4bbf).
We may also wish to set the version and/or serial number, as illustrated in
[OS-6727](https://smartos.org/bugview/OS-6727).

### UNIX domain sockets for COM1 and COM2

Since SmartOS does not have `nmdm`, we need more flexibility with use of UNIX
domain sockets to connect to LPC COM devices.  This is also a fairly
straight-forward change that may benefit FreeBSD users.

### mevent tests

To gain confidence in `mevent.c` [porting
efforts](https://github.com/mgerdts/illumos-joyent/commits/OS-6720), `mevent`
unit tests have been developed.  These may be helpful for FreeBSD as well.

### UEFI EDK2 extended write support

Ubuntu guests, which use grub and do not have a small `/boot` partition complain
`error: attempt to read or write outside of disk `hd0'` during boot.  This was
tracked down to lack of extended write support.  See
[OS-6604](https://smartos.org/bugview/OS-6604) and [the
fix](https://github.com/mgerdts/uefi-edk2/commit/f7714dd1907ad2ad09d254bda3c2dac064baba33).

While debugging this, it was discovered that the `bhyve` uart emulation code
does not handle 4-byte writes which are performed by grub debug statements.  See
[OS-6694](https://smartos.org/bugview/OS-6694) and [the
fix](https://github.com/mgerdts/illumos-joyent/commit/ec04362e4d1964b7135cd7b70545ab97ce33882b).

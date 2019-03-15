---
layout: post
title:  Mutual exclusion of bhyve and kvm on SmartOS
date:   2018-03-20 09:00:00 -0500
tags: omnios bhyve
commentIssueId: 8
tweet: <blockquote class="twitter-tweet"><p lang="en" dir="ltr">Mutual exclusion of bhyve and kvm on SmartOS <a href="https://t.co/f8g6K7W57Z">https://t.co/f8g6K7W57Z</a></p>&mdash; Mike Gerdts (@OMGerdts) <a href="https://twitter.com/OMGerdts/status/976131256986554368?ref_src=twsrc%5Etfw">March 20, 2018</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>
---

**Note**: This post is of historical interest only.  With the [fix](https://github.com/joyent/illumos-joyent/commit/befffd577ca6c3a090d7d3c72d267a383c3a3c45) for [OS-7080](https://smartos.org/bugview/OS-7080) in July of 2018 bhyve and kvm can coexist in peace.

If two hypervisors are not able to coordinate with each other, they must not
both use hardware viritualization.  Within SmartOS, there is a pair of
functions, `hvm_excl_hold()` and `hvm_excl_rele()`, described by [this
comment](https://github.com/joyent/illumos-joyent/blob/master/usr/src/uts/i86pc/os/pc_hvm.c#L24):

```
/*
 * HVM Exclusion Interface
 *
 * To avoid VMX/SVM conflicts from arising when multiple hypervisor providers
 * (eg. KVM, bhyve) are shipped with the system, this simple advisory locking
 * system is presented for their use.  Until a proper hypervisor API, like the
 * one in OSX, is shipped in illumos, this will serve as opt-in regulation to
 * dictate that only a single hypervisor be allowed to configure the system and
 * run at any given time.
 */
```

This advisory lock is taken as the first VM is starting and is released when the
last one is stopped.  This allows the use of either hypervisor, but prevents the
concurrent use of both.  Notably, the lock is not taken at module load time.

### Who has the lock?

You can use `mdb` to see which hypervisor currently has the lock:

```
# mdb -k
Loading modules: [ ... ]
> hvm_excl_holder::print
0
```

After starting a kvm instance, it says:

```
> hvm_excl_holder::print
0xfffffffff8224185 "SmartOS KVM"
```

### How is the lock taken and released?

Using a little dtrace, we can see the call stack involved in taking the lock.
For some reason, `dtrace` couldn't resolve the symbols so I used `::whatis` in
`mdb` to fill in the missing symbols.

```
# dtrace -n 'hvm_excl_*:entry { stack() }'
dtrace: description 'hvm_excl_*:entry ' matched 2 probes
CPU     ID                    FUNCTION:NAME
 48   8666              hvm_excl_hold:entry 
              0xfffffffff82aec97 (vmmdev_mod_incr+0x37)
              0xfffffffff82aee23 (vmmdev_do_vm_create+0x73)
              0xfffffffff82afbd3 (vmm_ioctl+0x193)
              genunix`cdev_ioctl+0x39
              specfs`spec_ioctl+0x60
              genunix`fop_ioctl+0x55
              genunix`ioctl+0x9b
              unix`sys_syscall+0x19f

 72   9030              hvm_excl_rele:entry 
              0xfffffffff82aed11 (vmmdev_mod_decr+0x31)
              0xfffffffff82af6b6 (vmm_do_vm_destroy_locked+0xe6)
              0xfffffffff82af70d (vmm_do_vm_destroy+0x3d)
              0xfffffffff82c9f9a (vmm_zsd_destroy+0x3a)
              genunix`zsd_apply_destroy+0x1d1
              genunix`zsd_apply_all_keys+0x5f
              genunix`zone_zsd_callbacks+0xe6
              genunix`zone_destroy+0x124
              genunix`zone+0x1e7
              unix`sys_syscall+0x19f
```

This release output shows that the bhyve instance I was running is in a zone and
the destroy of the VM state was taken care of by the *zone specific data*
`destroy` callback.

### And now for something unexpected

Despite what I said above, it is possible to run bhyve and kvm instances
simultaneously.

```
[root@emy-17 /root]# vmadm start $(vm test)
Successfully started VM 79062669-e229-e55d-960d-9b18d0fed8d0
[root@emy-17 /root]# vmadm start $(vm kvm-debian8)
Successfully started VM 57409636-927a-cc7a-d744-e0f92b73f4e4
[root@emy-17 /root]# vmadm list state=running
UUID                                  TYPE  RAM      STATE             ALIAS
57409636-927a-cc7a-d744-e0f92b73f4e4  KVM   1024     running           kvm-debian8
79062669-e229-e55d-960d-9b18d0fed8d0  BHYV  2048     running           test
```

It turns out that bhyve has the lock.

```
# echo hvm_excl_holder::print | mdb -k
0xfffffffff82caded "bhyve"
```

So, what's happening here?  In this case, kvm was not able to get the lock to
use hardware virtualization.  It is falling back to QEMU's software
virtualization, which is much slower.

Had the kvm instance been started first, kvm would have gotten the lock and the
bhyve instance would have failed to boot.  Now that both of these VMs are set to
being enabled, on the next reboot of the host, they will race to get the lock.

To state the obvious, trying to run bhyve and kvm VMs simultaneously is not
recommended.

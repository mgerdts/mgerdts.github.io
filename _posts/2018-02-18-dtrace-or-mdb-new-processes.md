---
layout: post
title:  dtrace or mdb new processes
date:   2018-02-18 19:57:23 -0600
tags: dtrace mdb bhyve
commentIssueId: 2
tweetWidgetId: 970343435155755008
---

I have a process that will be started later in a special context that I need to
trace with `truss(1M)`.  An easy way to do this is with a little help from `dtrace`:

In real life I did this as a one-liner on the command line.  I expand it here for the sake of comments

```dtrace
#! /usr/sbin/dtrace -ws

/*
 * Watch for the first system call performed by the zhyve program and
 * then trace that program with truss.
 */
syscall:::entry
/execname == "zhyve"/
{
	/* Stop the program */
	stop();
        /* Use truss to start tracing the program.  This causes the program to continue. */
	system("truss -t all -v all -w all -p %d", pid);
        /* Tell dtrace to exit once truss starts */
        exit(0);
}
```

This same approach works for attaching a debugger.

```
# dtrace -wn 'syscall:::entry / execname == "zhyve" / {stop(); system("mdb -p %d", pid); exit(0);}'
dtrace: description 'syscall:::entry ' matched 235 probes
dtrace: allowing destructive actions
CPU     ID                    FUNCTION:NAME
  0    795                 systeminfo:entry Loading modules: [ ld.so.1 ]
>
```

From the stack trace we can see that we caught it very early.  This is even before `main()` starts.

```
> $C
ffffbf7fffdff970 ld.so.1`sysinfo+0xa()
ffffbf7fffdffcb0 ld.so.1`setup+0xebd(ffffbf7fffdffe78, ffffbf7fffdffe80, 0, ffffbf7fffdfffd0, 1000, ffffbf7fef3a99c8)
ffffbf7fffdffdd0 ld.so.1`_setup+0x282(ffffbf7fffdffde0, 190)
ffffbf7fffdffe60 ld.so.1`_rt_boot+0x6c()
0000000000000001 0xffffbf7fffdfffc0()
```

This example was taken from some work I am doing to bring bhyve to SmartOS.  In
this case, `zhyve` is the replacement for the `init` process in a `bhyve`
branded zone.

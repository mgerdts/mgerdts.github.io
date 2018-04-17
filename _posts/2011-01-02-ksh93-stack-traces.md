---
layout: post
title:  "ksh stack trace"
date:   2011-01-02 19:57:23 -0600
tags: shell
commentIssueId: 1
tweet: .
---

It is often times handy to get a
[stack trace](http://en.wikipedia.org/wiki/Stack_trace) to help you understand
how a program got to an error condition.  Unfortunately, shell scripting
languages tend to not provide an easy mechanism to get a stacktrace.  The
following examines how it can be accomplished with ksh93.

```
#! /bin/ksh

function backtrace {
        typeset -a stack 
        # Use "set -u" and an undefined variable access in a subshell
        # to figure out how we got here.  Each token of the result is
        # stored as an element in an indexed array named "stack".
        set -A stack $(exec 2>&1; set -u; unset __unset__; echo $__unset__)

        # Trim the last entries in stack array until we find the one that
        # matches the name of this function.
        typeset i=0
        for (( i = ${#stack[@]} - 1; i >= 0; i-- )); do
                [[ "${stack[i]}" == "${.sh.fun}:" ]] && break
        done

        # Print the name of the function that called this one, stripping off
        # the [lineno] and appending any arguments provided to this function.
        print -u2 "${stack[i-1]/\[[0-9]*\]} $*"
        # Print the backtrace.
        for (( i--; i >= 0; i-- )); do
                print -u2 "\t${stack[i]%:}"
        done
}

# A couple functions to illustrate the output
function a {
        b "$@"
}

function b {
        c "$@"
}

function c {
        # Trigger a backtrace and exit if no arguments were passed
        (( $# == 0 )) && backtrace "missing arg" && exit 1
        print -- "$@"
}

a "$@"
```

A couple example runs:

```
$ ./backtrace.ksh hello world!
hello world!
$ ./backtrace.ksh
c: missing arg
        c[37]
        b[32]
        a[28]
        ./backtrace.ksh[41]
```

This post first appeared on a [legacy
platform](http://mgerdts.blogspot.com/2011/01/ksh93-backtraces.html)

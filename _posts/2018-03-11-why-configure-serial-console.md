---
layout: post
title:  "Why must we manaually configure a serial console?"
date:   2018-03-11 13:00:00 -0500
tags: acpi console bhyve
commentIssueId: 6
tweetWidgetId: 972902431272169472
---

I was looking at ACPI tables in a bhyve guest and stumbled across this:

```
root@test:/tmp# acpidump -b spcr
root@test:/tmp# iasl -d spcr.dat

Intel ACPI Component Architecture
ASL+ Optimizing Compiler version 20160831-64
Copyright (c) 2000 - 2016 Intel Corporation

Input file spcr.dat, Length 0x50 (80) bytes
ACPI: SPCR 0x0000000000000000 000050 (v01 BHYVE  BVSPCR   00000001 BHYV 00000001)
Acpi Data Table [SPCR] decoded
Formatted output:  spcr.dsl - 2599 bytes
root@test:/tmp# cat spcr.dsl
/*
 * Intel ACPI Component Architecture
 * AML/ASL+ Disassembler version 20160831-64
 * Copyright (c) 2000 - 2016 Intel Corporation
 *
 * Disassembly of spcr.dat, Sun Mar 11 17:44:37 2018
 *
 * ACPI Data Table [SPCR]
 *
 * Format: [HexOffset DecimalOffset ByteLength]  FieldName : FieldValue
 */

[000h 0000   4]                    Signature : "SPCR"    [Serial Port Console Redirection table]
[004h 0004   4]                 Table Length : 00000050
[008h 0008   1]                     Revision : 01
[009h 0009   1]                     Checksum : 78
[00Ah 0010   6]                       Oem ID : "BHYVE "
[010h 0016   8]                 Oem Table ID : "BVSPCR  "
[018h 0024   4]                 Oem Revision : 00000001
[01Ch 0028   4]              Asl Compiler ID : "BHYV"
[020h 0032   4]        Asl Compiler Revision : 00000001

[024h 0036   1]               Interface Type : 00
[025h 0037   3]                     Reserved : 000000

[028h 0040  12]         Serial Port Register : [Generic Address Structure]
[028h 0040   1]                     Space ID : 01 [SystemIO]
[029h 0041   1]                    Bit Width : 08
[02Ah 0042   1]                   Bit Offset : 00
[02Bh 0043   1]         Encoded Access Width : 00 [Undefined/Legacy]
[02Ch 0044   8]                      Address : 00000000000003F8

[034h 0052   1]               Interrupt Type : 01
[035h 0053   1]          PCAT-compatible IRQ : 04
[036h 0054   4]                    Interrupt : 00000000
[03Ah 0058   1]                    Baud Rate : 07
[03Bh 0059   1]                       Parity : 00
[03Ch 0060   1]                    Stop Bits : 01
[03Dh 0061   1]                 Flow Control : 03
[03Eh 0062   1]                Terminal Type : 02
```

Notice that it seems to have the address for `COM1` (0x3F8), IRQ 4, etc., which
all seems legit for my setup.  If all of this available when a machine redirects
a serial console, why have I been manually configuring serial consoles all these
years?

Let me predict the most common reasons:

1. It is somethign "new" that Microsoft came up with.  By "new" I mean sometime
in the past decade.
1. The BIOS implementations that support it are so buggy that the information is
not reliable

FWIW, [the specification](https://docs.microsoft.com/en-us/previous-versions/windows/hardware/design/dn639132(v=vs.85)) is available in `.docx` format from Microsoft.

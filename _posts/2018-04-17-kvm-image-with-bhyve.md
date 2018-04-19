---
layout: post
title:  Using kvm images with bhyve
date:   2018-04-17 17:00:00 -0500
tags: bhyve zones
commentIssueId: 9
tweet: <blockquote class="twitter-tweet"><p lang="en" dir="ltr">I&#39;ve been learning more about <a href="https://twitter.com/hashtag/cloud?src=hash&amp;ref_src=twsrc%5Etfw">#cloud</a>-init and wrote up a blog post showing how kvm Ubuntu images can be updated to work well with <a href="https://twitter.com/hashtag/bhyve?src=hash&amp;ref_src=twsrc%5Etfw">#bhyve</a> too. <a href="https://t.co/FLvVOgpX4W">https://t.co/FLvVOgpX4W</a></p>&mdash; Mike Gerdts (@OMGerdts) <a href="https://twitter.com/OMGerdts/status/986374311790501890?ref_src=twsrc%5Etfw">April 17, 2018</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>
---

So far, most of my use of bhyve has been with kvm images that have been
customize to work better with bhyve.  For Ubuntu images, those customizations
have generally been:

- Configure grub to use the serial console
- Install a version of cloud-init that has some fixes

So long as nothing is putting garbage on the second serial line, the version of
cloud-init that comes with Ubuntu works well enough that it can be used to
configure the serial console.  Early on, I was having troubles with garbage on
the serial line, but that seems to have been an ephemeral issue.

Cloud-init supports `user-data` with a variety of content.  I'll use
`cloud-config` to accomplish what I'm after.  This is not as straight-forward as
we would like - ideally we would just splat some more json in
`customer_metadata`.  But `cloud-init` really likes yaml.  And even if it
didn't, `customer_metadata` does not support nested structures.  So, we need
some yaml wrapped in json.

The cloud-config that's needed looks like this:

```yaml
#cloud-config
write_files:
  - content: |
      GRUB_DEFAULT=0
      GRUB_TIMEOUT=5
      GRUB_DISTRIBUTOR=`lsb_release -i -s 2> /dev/null || echo Debian`
      GRUB_CMDLINE_LINUX_DEFAULT="tsc=reliable earlyprintk"
      GRUB_CMDLINE_LINUX=""
      GRUB_TERMINAL="serial console"
      GRUB_CMDLINE_LINUX="console=tty0 console=ttyS0,115200n8"
      GRUB_SERIAL_COMMAND="serial --speed=115200 --unit=0 --word=8 --parity=no --stop=1"
    path: /etc/default/grub
runcmd:
  - update-grub2 >/var/tmp/update-grub2.log 2>&1
  - shutdown -r +1
```

To get that into json, I use a modified version of a [helper script found in the
cloud-init docs](http://cloudinit.readthedocs.io/en/latest/topics/format.html#helper-script-to-generate-mime-messages).

```python
#!/usr/bin/python

import json
import sys

from email.mime.multipart import MIMEMultipart
from email.mime.text import MIMEText

if len(sys.argv) == 1:
    print("%s input-file:type ..." % (sys.argv[0]))
    sys.exit(1)

combined_message = MIMEMultipart()
for i in sys.argv[1:]:
    (filename, format_type) = i.split(":", 1)
    with open(filename) as fh:
        contents = fh.read()
    sub_message = MIMEText(contents, format_type, sys.getdefaultencoding())
    sub_message.add_header('Content-Disposition', 'attachment; filename="%s"' % (filename))
    combined_message.attach(sub_message)

print(json.dumps({'cloud-init:user-data': str(combined_message)}))
```

The `customer_metadata` fodder is generated with:

```
$ userdatagen /tmp/kvm-to-bhyve.yaml:cloud-config
{"cloud-init:user-data": "From nobody Tue Apr 17 16:40:55 2018\nContent-Type: multipart/mixed; boundary=\"===============8760689715383894136==\"\nMIME-Version: 1.0\n\n--===============8760689715383894136==\nMIME-Version: 1.0\nContent-Type: text/cloud-config; charset=\"us-ascii\"\nContent-Transfer-Encoding: 7bit\nContent-Disposition: attachment; filename=\"/tmp/kvm-to-bhyve.yaml\"\n\n#cloud-config\nwrite_files:\n  - content: |\n      GRUB_DEFAULT=0\n      GRUB_TIMEOUT=5\n      GRUB_DISTRIBUTOR=`lsb_release -i -s 2> /dev/null || echo Debian`\n      GRUB_CMDLINE_LINUX_DEFAULT=\"tsc=reliable earlyprintk\"\n      GRUB_CMDLINE_LINUX=\"\"\n      GRUB_TERMINAL=\"serial console\"\n      GRUB_CMDLINE_LINUX=\"console=tty0 console=ttyS0,115200n8\"\n      GRUB_SERIAL_COMMAND=\"serial --speed=115200 --unit=0 --word=8 --parity=no --stop=1\"\n    path: /etc/default/grub\nruncmd:\n  - update-grub2 >/var/tmp/update-grub2.log 2>&1\n  - shutdown -r +1\n\n--===============8760689715383894136==--\n"}
```

That gets copied into the `customer_metadata` section of my `vmadm` payload.

```json
{
  "alias": "kvm2bhyve",
  "brand": "bhyve",
  "resolvers": [
    "8.8.8.8",
    "8.8.4.4"
  ],
  "ram": "1024",
  "vcpus": "2",
  "nics": [
    {
      "nic_tag": "admin",
      "ip": "10.88.88.207",
      "netmask": "255.255.255.0",
      "gateway": "10.88.88.2",
      "model": "virtio",
      "primary": true
    }
  ],
  "disks": [
    {
      "image_uuid": "429bf9f2-bb55-4c6f-97eb-046fa905dd03",
      "boot": true,
      "model": "virtio"
    }
  ],
  "zlog_max_size": 1048576,
  "zlog_keep_rotated": 5,
  "customer_metadata": {
    "cloud-init:user-data": "From nobody Tue Apr 17 16:40:55 2018\nContent-Type: multipart/mixed; boundary=\"===============8760689715383894136==\"\nMIME-Version: 1.0\n\n--===============8760689715383894136==\nMIME-Version: 1.0\nContent-Type: text/cloud-config; charset=\"us-ascii\"\nContent-Transfer-Encoding: 7bit\nContent-Disposition: attachment; filename=\"/tmp/kvm-to-bhyve.yaml\"\n\n#cloud-config\nwrite_files:\n  - content: |\n      GRUB_DEFAULT=0\n      GRUB_TIMEOUT=5\n      GRUB_DISTRIBUTOR=`lsb_release -i -s 2> /dev/null || echo Debian`\n      GRUB_CMDLINE_LINUX_DEFAULT=\"tsc=reliable earlyprintk\"\n      GRUB_CMDLINE_LINUX=\"\"\n      GRUB_TERMINAL=\"serial console\"\n      GRUB_CMDLINE_LINUX=\"console=tty0 console=ttyS0,115200n8\"\n      GRUB_SERIAL_COMMAND=\"serial --speed=115200 --unit=0 --word=8 --parity=no --stop=1\"\n    path: /etc/default/grub\nruncmd:\n  - update-grub2 >/var/tmp/update-grub2.log 2>&1\n  - shutdown -r +1\n\n--===============8760689715383894136==--\n"
  }
}
```

Notice that the `image_uuid` is for an `ubuntu-certified-16.04` image.

If all goes well, a couple minutes after running `vmadm create`, you will notice
the grub menu on the console.

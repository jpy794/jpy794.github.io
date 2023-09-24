---
title: "Final Blog Post - GSoC '23"
date: 2023-09-03T16:23:29+08:00
draft: false
tags: []
series: []
featured: true
---

This is my final blog post for GSoC '23. For me, the past few months have been both challenging and rewarding. Although I didn't achieve all the goals I set out to do, I learned a lot in the process.

<!-- more -->

## What Was Done

As mentioned in the last post, we created a new project, boot2ipxe, to build netboot disk image for different platforms. With that as the base image, ipxe-boot-server could generate disk image that boots boot2container from a HTTPS server. Thanks to my mentor who took the time to add ARM64 support to valve-infra container, we can now launch valve-infra on RPi!

## Boot Valve Infra on RPi

There're a few steps to take.

- Configure the HTTPS server and build your RPi SD card image according to readme of ipxe-boot-server.
- Download prebuilt kernel and initramfs for RPi from boot2container release put them in the correct place according to readme of ipxe-boot-server.
- Write a boot script to run valve-infra container with boot2container 

Here's a simple example script, which requires manually creating another partition on the SD card as cache partition for valve-infra container image. (Or boot would fail due to lack of disk space.)

This script also requires kernel and initramfs to be placed in the same directory as the script. (`files/${fingerprint}/`)

``` shell
#!ipxe

kernel /files/linux-arm64-6.5.1 initrd=b2c b2c.run="-ti registry.freedesktop.org/mupuf/valve-infra/valve-infra-container:aarch64" b2c.ntp_peer=auto b2c.cache_device=/dev/mmcblk0p2
initrd --name b2c /files/initramfs.linux_arm64.cpio.xz
boot
```

Most of the steps above have been explained in previous blog posts. Check them out if any step is not clear.

If everything goes well, you should see something like
```
[root@boot2container ~]# /usr/local/lib/dashboard/dashboard.py --services 'avahi-daemon' 'executor' 'minio' 'salad' 'gitlab-runner' '*-registry' 'influxdb' 'influxdb_configure' 'telegraf' 'nftables' 'vpdu'
```
And then you're in the dashboard of valve-infra.

## Thoughts

This is my first time participating in a program like GSoC. I should have realized earlier that workload of GSoC isn't small. Especially it's my junior year. The class, competition, application season, internship and starting new life in another city, those things can be overwhelming enough. So I think I should learn to let go of some tasks and focus on one thing. Maybe it will work out better that way.

Anyway, GSoC is a great experience for me. And I would like to express my sincere gratitude to my amazing mentor mupuf for his care and guidance.

---
title: "Run Boot2container on RPi - GSoC '23"
date: 2023-09-03T16:23:29+08:00
draft: false
tags: []
series: []
featured: true
---

Long time no see! It's time to share my progress on the GSoC project.
Last time we manually built a netboot SD card image for RPi 3B+ and left some questions to be answered.
Now let's dig into those questions and show you what I've got.

<!-- more -->

## Boot2ipxe

First, a new project, [Boot2ipxe](https://gitlab.freedesktop.org/mupuf/boot2ipxe), is created for building and testing iPXE netboot disk image for both SBCs and x86 PC,
which used to be part of [ipxe-boot-server](https://gitlab.freedesktop.org/mupuf/valve-infra/-/tree/master/ipxe-boot-server)'s job.

Boot2ipxe allows you to build and customize iPXE netboot disk image easily with a single `make` command.
If you're not familiar with your SBC's boot process and just need a working iPXE disk image, make sure to check
the link above. There're both prebuilt images and guides to built them yourself with more flexibility.

Besides, ipxe-boot-server now depends on boot2ipxe to generate disk image for gateways,
which means it should support any device that boot2ipxe supports.

## Run Boot2container with Boot2ipxe

> Ref: https://gitlab.freedesktop.org/mupuf/netboot2container

With boot2ipxe, we can boot [boot2container](https://gitlab.freedesktop.org/mupuf/boot2container) with netboot easily. All we need to do is to set up a netboot server, which can be achived in many ways, such as
- Reuse the https server in the last post
- Set up a local http server
- Set up a DHCP server

Here I choose the last one, as it's boot2ipxe's default behavior to boot from local DHCP server.
We can use prebuilt boot2ipxe image instead of builing our own to embed our http/https server address in the image.

About how to flash the image, please refer to README of boot2ipxe.

Now let's set up the DHCP server.

Install `dnsmasq` before moving on.

In the following steps, I'll assume you have a RPi connected to the network interface naming `${lan_if}`,
and your gateway interface is `${wan_if}`.

The first step is to assign an IP address to `${lan_if}`.

``` shell
sudo ip addr add dev ${lan_if} 10.0.0.1/24
```

Now let's set up IP forward and NAT.

``` shell
sudo sysctl net.ipv4.ip_forward=1

sudo nft add table nat
sudo nft add chain nat postrouting { type nat hook postrouting priority 100 \; }
sudo nft add rule nat postrouting ip saddr 10.0.0.0/24 oifname ${wan_if} masquerade
```

You need create a `tftp` folder under the path you gonna run `dnsmasq`, as root folder of the netboot server.
In the `tftp` folder, create an iPXE script `boot.ipxe` and put boot2container kernel and initramfs there
(You can download them from [boot2container release](https://gitlab.freedesktop.org/mupuf/boot2container/-/releases)).
Remember to replace `${kernel}` and `${b2c_initramfs}` with correct filename.

``` shell
#!ipxe

kernel /${kernel} initrd=b2c b2c.run="-ti docker.io/library/alpine:latest" b2c.ntp_peer=auto
initrd --name b2c /${b2c_initramfs}
boot
```

Then we're ready to start DHCP server.

``` shell
sudo dnsmasq --port=0 \
    --conf-file=/dev/null \
    --dhcp-leasefile=$(pwd)/dnsmasq.leases \
    --dhcp-boot=/boot.ipxe \
    --dhcp-range=10.0.0.2,10.0.0.254 \
    --dhcp-option=option:dns-server,1.1.1.1 \
    --interface=${lan_if} \
    --enable-tftp=${lan_if} \
    --tftp-root=$(pwd)/tftp \
    --no-daemon
```

You can boot up your RPi now. After spending time pulling Docker image etc., the RPi should output something in the end like
```
+ podman start -a 5d85155c5ed80ebc85badf45766db7e33be2b5c12c0292025fed28f6d7fb1391
/ # 
```
You've booted your RPi into boot2container with a prebuilt iPXE disk image from boot2ipxe!

If you're using RPi 3B+, don't worry if boot2container complains about no available network interface and fails to start Apline Linux.
There's a known [mainline kernel issue](https://lore.kernel.org/all/d04bcc45-3471-4417-b30b-5cf9880d785d@i2se.com/) for RPi 3B+.
[Mupuf]((http://www.mupuf.org/contact/mupuf.html)) found a [workaround](https://gitlab.freedesktop.org/gfx-ci/boot2container/-/merge_requests/22) to temporarily disable usb_onboard_hub before it's properly fixed.
But currently the kernel downloaded from boot2container release above is still broken.

---
title: "The Project - GSoC '23"
date: 2023-05-14T20:41:37+08:00
draft: false
tags: [gsoc]
series: [gsoc]
featured: true
---

I received the email saying that my proposal for GSoC '23 was accepted by [X.Org Foundation](https://www.x.org) on May 5th. It’s my first time working with open-source communities and I’m really excited about it. My mentor [Martin Roukala](http://www.mupuf.org/contact/mupuf.html) gave me some great advice and helped me understand the project and my tasks better. Now it’s time to dive into the project details.

<!--more-->

By the way, I guess I'm not that good at writing. So let's keep this blog short :P.

## Valve CI Infrastructure

[Valve CI Infrastructure](https://gitlab.freedesktop.org/mupuf/valve-infra)(valve-infra) targets at building a maintenance-free gateway for a bare metal CI farm. The gateway is a machine that interacts with the user over the internet using RESTful APIs and manages the device under test (DUT). 'Maintenance-free' means that even if we accidentally load a wrong kernel to the gateway, it could be reset to a correct state with a simple reboot instead of flashing an OS. Therefore, [iPXE](https://ipxe.org/) is used in valve-infra for netboot support so that we can replace the kernel image remotely and use a WiFi smart plug to reboot it.

Ideally, We want the gateway to be plug-and-play so that people can share their spare machine as a CI runner with ease. However, due to the wide range of platforms available, it is difficult to find a way like some SD card image that boots on all devices. To solve this problem, we require machine owners to provide at least a prebuilt disk image with a working bootloader and iPXE. Then we could add other things to the prebuilt image, such as a new iPXE with the private key for identification when the gateway connects to the iPXE server, and generates a netboot image compatible with valve-infra.

To make maintenance of the gateway even easier, we use containers to keep the environment clean between different versions of gateway. [Boot2container](https://gitlab.freedesktop.org/mupuf/boot2container) initramfs was used to fetch and boot directly into a valve-infra container, which is also highly configurable by kernel cmdline.

## The Project

My GSoC project is to port valve-infra to a single board computer (SBC), specifically [Raspberry Pi 3B+](https://www.raspberrypi.com/products/raspberry-pi-3-model-b-plus/).

Currently valve-infra only supports building an ISO that runs on x86 UEFI platform and lacks generalization as the boot process of SBCs differs vastly. At the first stage, I’ll try to figure out a proper specification for the prebuilt disk image. This should make it clear where the iPXE binary and the tmp partition for boot2container are.

Then I should adapt valve-infra's ipxe-boot-server according to the specification above. By adding iPXE related files to the prebuilt disk image correctly, it will generate a netboot image. The netboot image should be able to fetch the kernel, boot2container and finally boot a basic container like `alpine:latest`.

After that, we need to build an ARM64 version of valve-infra gateway container. Hope we can simply relpace the base image and get everything work fine.

Finally, a gitlab CI pipeline that automates the disk image building process could be created so that it can be done with a single click.

If there's still time left, I may as well try porting valve-infra to other SBCs. :)


---
title: "Getting Started - GSoC '23"
date: 2023-06-02T17:43:42+08:00
draft: false
tags: [gsoc]
series: [gsoc]
featured: true
---

After a week of setting up the iPXE server, my Raspberry Pi 3B+ can now boot Linux kernel from it with TLS mutual authentication enabled. In this post I'll introduce how I built the SDcard image for my RPi and set up the iPXE server. Both of them are currently only for testing purposes so the process could be kind of messy. Anyway, let's just get it running and leave the optimization for future. Also, like before I'll append the TODO list to the end of this blog. Take a look if this post interests you. :D

<!-- more -->

## The SDcard Image

We use [iPXE](https://ipxe.org) as the netboot firmware. To be able to boot iPXE modernly, a UEFI compatible bootloader is needed. For RPi, there's two bootloaders as our candidates, [EDK2](https://github.com/tianocore/edk2-platforms/tree/master/Platform/RaspberryPi/RPi3) and [U-Boot](https://u-boot.readthedocs.io/en/latest/develop/uefi/uefi.html). As the former seems way too heavy for our use and the latter has better support among ARM SBCs, I chose U-Boot, hoping that could make it easier for us to port the image to SBCs other than RPi.

[Here](https://github.com/jpy794/rpi-uboot-ipxe) is a demo script I wrote to build the SDcard image. Feel free to have a look if there's any ambiguity in my description below.

### RPi3 Boot Flow

- When powered on, the VideoCore (VC) runs the BootROM stored in SoC, which loads `bootcode.bin` from SDcard to L2 cache and runs it, while the ARM core is resetted.
- `bootcode.bin` initializes SDRAM and then reads `fixup.dat` and `start.elf` from SDcard.
    - `fixup.dat` is a linker file that helps spliting the SDRAM between VC and ARM.
    - `start.elf` is the VC firmware, which then reads `config.txt` from SDcard and carries out rest of the boot process.
- `start.elf` is able to load kernel, device tree blob (DTB) and apply configurations from `config.txt`. We'll configure it to load U-Boot.
- Finally, the reset signal for ARM core is cleared. U-Boot executes iPXE, which fetches kernel and initrd from remote server. Boom, our RPi boots!

> [RPi firmwares](https://www.raspberrypi.com/documentation/computers/configuration.html#boot-folder-contents) \
> [fixup.dat use](https://forums.raspberrypi.com/viewtopic.php?t=173308) \
> [RPi boot process](https://forums.raspberrypi.com//viewtopic.php?f=63&t=6685)

Now, let's build up the SDcard image step by step.

### Firmware

`bootcode.bin`, `fixup.dat` and `start.elf` are all firmwares provided by the vendor, which can be downloaded from [RPi firmware repository](https://github.com/raspberrypi/firmware/tree/master/boot). The repo also contains DTB for the [RPi kernel](https://github.com/raspberrypi/firmware/tree/master/boot). For RPi3B+, the file name is `bcm2710-rpi-3-b-plus.dtb`. If you use RPi kernel instead of mainline kernel, it's convenient to download DTB from here directly.

Now it time to grab a SDcard and copy the firmwares to it. RPi requires the first partition on the SDcard to be FAT16 or FAT32. That's the where `bootcode.bin`, `fixup.dat`, `start.elf` shoud be placed.

However, FAT32 seems to not work well with my RPi. I still don't know why but switching to FAT16 did indeed solve my problem. I wrote it down here in case someone could have a similar issue.

Here's how I created the partition.
``` shell
# suppose your sdcard is /dev/sda
parted --align optimal /dev/sda mklabel msdos mkpart primary fat16 0M 32M

# now create the filesystem
mkfs.fat -F 16 /dev/sda1
```

> [RPi partitioning explained](https://github.com/raspberrypi/noobs/wiki/Standalone-partitioning-explained)

Then just copy the firmwares to root directory of the partition we've just created.

### Config.txt

Apart from those blobs, we also need to provide a `config.txt`. In following example, some of the parameters are necessary for boot process, some are set for the convenience of debug.

``` shell
# we want to start u-boot in 64-bit mode
arm_64bit=1

# this is the device tree blob start.elf should load
device_tree=bcm2710-rpi-3-b-plus.dtb

# we'll build u-boot and place the binay here later
kernel=u-boot.bin

# miniuart is disabled by default, enable it
enable_uart=1

# set frequency of VC core to a fixed value so that miniuart has a stable baud rate
core_freq=250

# enable logging for bootcode.bin and start.elf
uart_2ndstage=1
```

Among configurations above, the trickiest part is to get the UART working. RPi3B+ has 2 UART, PL011 and mini UART. By default, the former is used by bluetooth and the latter is mapped to GPIO. However, the clock of mini UART is linked to clock of VC core. If frequency of VC core is not fixed, we'll never get valid output from mini UART due to the changing baud rate.

> [documentation of `config.txt`](https://www.raspberrypi.com/documentation/computers/config_txt.html)

Now, simply plug the SDcard into the RPi, connect the serial, power it up and you should see some logs printed out like

```
Raspberry Pi Bootcode
Read File: config.txt, 209
Read File: start.elf, 2975744 (bytes)
Read File: fixup.dat, 7266 (bytes)
MESS:00:00:01.212454:0: brfs: File read: /mfs/sd/config.txt
MESS:00:00:01.216673:0: brfs: File read: 209 bytes
...
```

### U-Boot

There's RPi support in mainline U-Boot. So it's pretty straightforward to boot it on our board.

First, let's build for RPi3B+ target.

``` shell
git clone https://github.com/u-boot/u-boot.git

cd u-boot
make ARCH=arm CROSS_COMPILE=aarch64-linux-gnu- rpi_3_b_plus_defconfig

# this should generate u-boot.bin
make ARCH=arm CROSS_COMPILE=aarch64-linux-gnu- -j`nproc`
```

Then, we need to create a boot script (`boot.cmd`) which executes the iPXE binay (`efi/tools/snp.efi`).

`boot.cmd`
``` shell
load mmc 0:1 $kernel_addr_r efi/tools/snp.efi
bootefi $kernel_addr_r
```

``` shell
# this should generate boot.scr
./tools/mkimage -A arm -T script -d boot.cmd boot.scr
```

Copy `u-boot.bin` and `boot.scr` to root directory of the boot partition and power the RPi up. There should be U-Boot logs printed out.

```
U-Boot 2023.04-rc4-00053-g8be7b4629e (Jun 03 2023 - 22:19:18 +0800)

DRAM:  948 MiB
RPI 3 Model B+ (0xa020d3)
Core:  66 devices, 14 uclasses, devicetree: embed
MMC:   mmc@7e202000: 0, mmc@7e300000: 1
Loading Environment from FAT... Unable to read "uboot.env" from mmc0:1...
In:    serial
Out:   vidconsole
Err:   vidconsole
Net:   No ethernet found.
starting USB...
Bus usb@7e980000: USB DWC2
scanning bus usb@7e980000 for devices... 4 USB Device(s) found
       scanning usb for storage devices... 0 Storage Device(s) found
Hit any key to stop autoboot:  0
```

### iPXE

Before building iPXE, enable a few features we'll need later firstly.

``` shell
git clone https://github.com/ipxe/ipxe.git
cd ipxe/src
```

`config/local/general.h`
``` c
/* general.h */
#define NSLOOKUP_CMD            /* Name resolution command */
#define PING_CMD                /* Ping command */
#define NTP_CMD                 /* NTP commands */
#define VLAN_CMD                /* VLAN commands */
#define IMAGE_EFI               /* EFI image support */
#define DOWNLOAD_PROTO_HTTPS    /* Secure Hypertext Transfer Protocol */
#define DOWNLOAD_PROTO_FTP      /* File Transfer Protocol */
#define DOWNLOAD_PROTO_NFS      /* Network File System Protocol */
#define DOWNLOAD_PROTO_FILE     /* Local file system access */
```

Like U-Boot, we need to create a boot script (`embed.ipxe`).

`embed.ipxe`
``` shell
#!ipxe

:retry
dhcp || goto retry
ntp pool.ntp.org || goto retry

# we'll replace the url with our ipxe server later
chain -a https://example.com || goto retry

# unreachable
goto retry
```

According to [iPXE documentation](https://ipxe.org/appnote/buildtargets), we should select `snp` target for ARM devices. I didn't notice that and tried several times before finding the right one. :P

``` shell
# this should generate bin-arm64-efi/snp.efi
make CROSS_COMPILE=aarch64-linux-gnu- EMBED=embed.ipxe bin-arm64-efi/snp.efi -j`nproc`
```

Next, copy `snp.efi` to `/efi/tools/snp.efi` and we should get a working iPXE.

```
iPXE 1.21.1+ (gb00935) -- Open Source Network Boot Firmware -- https://ipxe.org
Features: DNS FTP HTTP HTTPS iSCSI NFS TFTP VLAN AoE EFI Menu
lan78xx_eth Waiting for PHY auto negotiation to complete....... done
Configuring (net0 b8:27:eb:ab:cb:dc)....... ok
```

## The iPXE Server

In Valve Infra, a gateway fetches its kernel and initrd through public network. To protect our netboot from man-in-the-middle attack and unauthorized access, we need a HTTPS iPXE server with client certificate enabled. Basically, we need to get the following things done.

- Get a SSL certificate for the domain.
- Set up a HTTPS server with Nginx.
- Create a CA and enable client verification in Nginx.
- Issue client certificates using the CA we've just generated. Embed the ceritficate in our iPXE binary.

Here are some references.
> [mupuf's blog](https://mupuf.org/blog/2022/01/10/setting-up-a-ci-system-part-3-provisioning-your-ci-gateway/) \
> [valve infra - ipxe boot server](https://gitlab.freedesktop.org/mupuf/valve-infra/-/blob/master/ipxe-boot-server/README.md) \
> [ipxe - crypto](https://ipxe.org/crypto) \
> [nginx - http_ssl_module](http://nginx.org/en/docs/http/ngx_http_ssl_module.html#ssl_trusted_certificate)

### HTTPS Server

Thanks to [ACME](https://en.wikipedia.org/wiki/Automatic_Certificate_Management_Environment) protocol and service providers like Let's Encrypt, ZeroSSL, etc, we can get a SSL ceritificate in a few minutes. I used [acme.sh](https://github.com/acmesh-official/acme.sh) to automate the process.

Remember to get a RSA certificate as iPXE does not support ECC certificate currently.

There used to be size limit for the server certificate because iPXE didn't support fragmented TLS handshake. But now the issue has already been [fixed](https://github.com/ipxe/ipxe/pull/930). So feel free to get a SSL certificate of any length. :D

Now simply install and configure Nginx. Create a new configuration at `/etc/nginx/sites-available` and link it to `/etc/nginx/sites-enabled`. Add a server block with a normal SSL configuration. Set `ssl_certificate` to the SSL certificate and `ssl_certificate_key` to your private key. This should give you a working HTTPS server.

iPXE doesn't support all cipher suites though. So take a look at their [documentation](https://ipxe.org/crypto) and check your Nginx settings twice. I also found [Mozilla SSL configuration generator](https://ssl-config.mozilla.org/#server=nginx&config=intermediate) useful. It generates Nginx configuration that works out of the box with iPXE when set to `Intermediate` mode.

You can use `nmap` to detect cipher suites your server allows.
``` shell
nmap -script ssl-enum-ciphers -p 443 example.com
```

If all above are set correctly, you can replace the url in the iPXE boot script with your server address and host a simple iPXE script with Nginx, like

``` shell
#!ipxe

echo hello from server
```

The RPi should be able to output `hello from server`.

### Client Certificate

About how to create CA and issue client certificates, explanation from [Valve Infra](https://gitlab.freedesktop.org/mupuf/valve-infra/-/blob/master/ipxe-boot-server/README.md) and [iPXE](https://ipxe.org/crypto) are clear enough. It's basically some OpenSSL thing. Let's simply skip it.

I suppose you've already got a CA, a client certificate and corresponding private key.

#### Server

For server side, adding two lines to the server block of the Nginx configuration created above is enough.

``` shell
ssl_client_certificate /path/to/ca
ssl_verify_client on;
```

Now, the resources of the server shouldn't be accessible from client without certificates, like your browser.

#### Client

For client side, we need to rebuild iPXE to embed the client certificate and private key.

``` shell
make bin-arm64-efi/snp.efi -j`nproc` \
    CROSS_COMPILE=aarch64-linux-gnu- \
    EMBED=embed.ipxe \
    PRIVKEY=client.key CERT=client.crt
```

Be sure to keep the generated `snp.efi` in a safe place. It contains unencryped private key of your client certificate.

Now, only your RPi could download scripts/binaries from your iPXE server!

#### An iPXE Issue

When tring to embed the private key in the iPXE binary, I encountered a weird issue. When doing a SSL handshake, iPXE complained that no private key was found corresponding to the client certificate, which I had definitely passed to `make` by `PRIVKEY=...`.

I enabled more debug output and found iPXE failed parsing the private key in [`rsa_parse_mod_exp()`](https://github.com/ipxe/ipxe/blob/6a7f560e60837fc2ce82a7aa976035656f7d231e/src/crypto/rsa.c#L159). The private key is stored in ASN.1 structure. So I used `openssl asn1parse` to decode my private key. The output didn't match the structure expected by the parsing function at all. At that point I realized something could be wrong with my private key.

It didn't take me long to find that the private key I generated was in PKCS#8 format and the parsing function assumed it's PKCS#1. I  went to check OpenSSL documentation. Guess what? OpenSSL 3.0 and above outputs private key in PKCS#8 by default instead of PKCS#1, which explained exactly the problem I was having.

Adding a `-traditional` parameter should make OpenSSL output private key in the right format. After converting format of my private key to PKCS#1, I tried again. But it still didn't work.

At last, I found iPXE also called OpenSSL in its [Makefile](https://github.com/ipxe/ipxe/blob/6a7f560e60837fc2ce82a7aa976035656f7d231e/src/Makefile.housekeeping#L691) to convert the private key to DER binary format, which, of course, converted my PKCS#1 input back to PKCS#8 again!

So the solution is simple. When OpenSSL version >= 3, append a `-traditional` to that command in Makefile. I made a small patch and created a [PR](https://github.com/ipxe/ipxe/pull/963). The maintainer of iPXE responded pretty fast and chose to add PKCS#8 formatted private key [support](https://github.com/ipxe/ipxe/pull/964) instead. That's way more better than my workaround!

## What's Next

After learning all those above and setting up the test environment, now it's time to do some real work.

I'll create a repository that generates SDcard image with UEFI bootloader for RPi (hope there'll be more SBCs supported in the future). That image will serve as a base image so that we could ship different iPXE with it for different use, like a HTTP-only version for test machines and a HTTPS with client verification version for gateways. Actually, generating SDcard image for gateways from base image is also on my TODO list, which involves some changes in [ipxe-boot-server](https://gitlab.freedesktop.org/mupuf/valve-infra/-/tree/master/ipxe-boot-server).

There's also a lot of questions left, such as
- The way we load device tree
    - Currently by RPi firmware or U-Boot.
    - Should we load it by iPXE so that we could switch kernel and device tree together remotely?
- The specification of the base image for both MBR and GPT
    - So that we could put the iPXE binary in the right place when building the SDcard image.

Hope in my next blog there'll be answers to all of them. :D

That's all. Thank you for reading until the end! Hope to see you again soon.

---
title: "让 Linux 在我的笔记本上正常休眠/睡眠"
date: 2023-03-06T14:54:12+08:00
draft: false
tags: []
series: []
featured: true
---

去年买笔记本, 为了满足轻便长续航, 高扩展的需求, 笔者购入了机械革命无界16Pro, 成为一名~~光荣的~~革命军. 尽管市面上有诸如 Framework, ThinkPad 一类的提供官方 Linux 支持的产品, 但笔者就是要享受 DIY 的乐趣(~~绝对不是因为贵~~), 这里记录一下自己让笔记本在 Linux 下能(比较)正常休眠/睡眠的全过程.

<!--more-->

## suspend 唤醒后屏幕点不亮

同时 i915 在启动和 suspend 时会在 kernel log 里打 calltrace.

问题的原因是厂商 BIOS 提供了错误的 VBT, 把 HDMI 输出描述成了 eDP 输出, i915 不能正确处理这种软硬件不一致, 导致屏幕点不亮, 此外 HDMI 也因此不能使用(虽然我并没有测试过).

我遇到这个问题是在 2022.10, 已经有人提前踩坑了, 被[@taoky](https://github.com/taoky)学长指点, 少走了很多弯路.

这里附上相关链接
> [taoky's blog](https://blog.taoky.moe/2022-07-21/i915-debug-on-tpt14gen3.html)
>  
> [i915 issue](https://gitlab.freedesktop.org/drm/intel/-/issues/5531)

ThinkPad 很快就提供了 BIOS 更新解决了上述问题. 至于机械革命的机器, 目前有些 workaround 可以临时用下, 等上面 issue 的 patch 合入主线就至少可以正常 suspend 了(可能差不多6.3?), HDMI 还要多等一等.

## 开盖唤醒一会后再次 suspend

在我开开心心打上 workaround 重新编译内核后, 终于可以在 suspend 后点亮屏幕了, 但正常使用 ~15s 后又会再次 suspend.

多次尝试后我发现, 只有合盖会导致这种问题, `systemctl suspend` 则可以正常唤醒, 于是开始怀疑盖子的开关. 用 `acpi_listen` 监听 ACPI 事件发现, 打开盖子唤醒笔记本后, 固件没有报告盖子打开(而在不进入 suspend 的情况下, 盖子的状态又可以被正确报告), 于是 systemd 过了一会意识到不对, 就又把笔记本丢回 suspend.

然后开始找解决方案.

果然, 厂商不遵守 ACPI 规范也不是一天两天了, 内核有一个 quirk 可以在唤醒后自动把盖子状态设置为打开, 只要在 kernel command-line parameters 里加上 `button.lid_init_state=open` 即可.

## hibernate 后 wifi 打不开

`rfkill` 提示 wlan hard blocked, 经过一番信息检索, 这一般是被物理开关禁用了 wlan. 无界16Pro上还真有一个, 是`Fn+F1`, 但我试着按了几次, 并不能解除 block(在hibernate 前这个按键是正常工作的), 多半又是个固件bug.

一番搜索无果, 不过在各种尝试后发现了一个可行的 workaround: 在 hibernate 前卸载 `iwlmvm` 和`iwlwifi`, 启动后再加载回来, 此时就不会出现 wifi 被锁死的问题.

如果你是 Archlinux 用户, 可以用 AUR 的 systemd-suspend-modules, 然后在 `/etc/suspend-modules.conf` 里添加要卸载的模块就行了.

## 总结

如果不是开源固件的机器, 用户修复固件bug的可能几乎为零, 而大部分厂商的态度都是: 能跑 Windows 就行, Linux 的问题不予处理. 到了机械革命这种小厂, 可能 Windows 的问题还没有处理完, 哪里会理会 Linux 用户呢?

好在笔者遇到的问题, 花亿点点时间, 或多或少都有应对方案, ~~也不是不能用嘛~~.

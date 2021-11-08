---
title: 玩转android
date: 2021-08-02
tags:
    - Android
    - Magisk
categories: Android
---

## 1、Android 几个模式

**正常模式**。也就是正常启动Android操作系统的样子。启动操作系统后，可以在开发者选项中打开USB调试。打开后可以用`adb`命令对连线的手机执行一些操作：

```sh
#查看当前连接的android设备
adb devices
#启动shell执行交互式命令
adb shell
#在设备上执行"ps -A"命令查看所有进程
adb shell ps -A
```

> 更多adb命令可以参考这个[sheet](https://gist.github.com/Pulimet/5013acf2cd5b28e55036c82c91bd56d8)

**fastboot**类似于pc的bios，也被人叫做“刷机模式”，可以看到一些设备信息。

```sh
#使用adb重启bootloader进入fastboot模式
adb reboot bootloader
```
> 大部分手机关机后，同时按住音量“+”或“-”，再长按开机键也可以进入fastboot模式

可以使用`fastboot`命令，查看fastboot模式下的设备

```sh
fastboot devices                 # 查看fastboot连接设备
fastboot reboot                  # 正常重启设备
```

![fastboot](https://ae03.alicdn.com/kf/Uceccd6e0a0fb4ee4af1c2b320cdf9c55A.jpg)

**recovery**模式类似于Windows PC的U盘启动器或者MacOS的还原模式。Android的recovery模式和MacOS的还原模式类似，将存储盘的一部分划成recovery分区当作U盘启动器用，里面有一个微型还原系统，可以对手机进行设备还原，重新写入操作系统。

```sh
#使用adb命令以recovery模式重启手机
adb reboot recovery
```

![recovery mode](https://ae04.alicdn.com/kf/U019bce29a35b46a2a875fe48f881aea8J.jpg)

> 除了`/recovery`分区，android还有`/boot`、`/system`、`/data`、`/cache`、`/misc`等分区，可以参考[有没有哪位能详细讲解一下安卓手机的分区? - 知乎 (zhihu.com)](https://www.zhihu.com/question/39030945)和[Android系统分区与升级 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/364003927)看看各分区的作用，关于Android A/B system分区可以参考[A/B（无缝）系统更新](https://source.android.com/devices/tech/ota/ab)。
>
> 使用下面命令可以看到安卓预设的各种分区。
>
> ```sh
> adb shell ls -al /dev/block/platform/soc/1da4000.ufshc/by-name
> ```

## 2、解锁bootloader

PC想要重装操作系统，还是挺简单的。ghost备份的方式安装或者U盘装机。但是安卓刷机就有点麻烦了。首先一个原因就是几乎所有的安卓机都有OEM锁。

![android boot process](https://ae02.alicdn.com/kf/U53865680079444718e8a40ffebdb9c6dg.jpg)

OEM全称Original Equipment Manufacturer，也就是原始设备制造商为了防止用户刷成其他系统设置的锁。bootloader是fastboot模式下的加载程序，OEM锁就类似于PC上BIOS的密码。

![BIOS password](https://www.top-password.com/images/bios-supervisor-password.png)

> PC操作系统基本被Windows占领了，Windows是闭源的，微软一家发售，所以在PC上顶多就只能对BIOS设置个密码。但Android是谷歌开源的操作系统，华为、小米、VIVO等手机制造商都可以对Android进行定制化。这些定制化系统有很多预装软件，预装软件是很有商业价值的，比如说小米视频等各大预装软件广告如云，小米金融、小米之家等软件带来用户转化。所以手机制造商给设备上锁不让用户装其他系统，就是为了避免给别人徒作嫁衣。如果用户硬要解锁呢，那手机厂商不给保修。

每家手机制造商的解锁方式都不一样，我这里找了华为、小米、一加的解锁方案：

[华为]: https://club.huawei.com/thread-3321291-1-1.html
[小米]: https://www.miui.com/unlock/
[OnePlus]: https://support.oneplus.com/app/answers/detail/a_id/588/~/how-to-unlock-bootloader-for-oneplus-smart-phone

这里以一加为例：

1、在系统设置的开发者选项里找到“OEM解锁”，把它置成允许。

2、正常模式连接`adb`后可以执行`adb reboot bootloader`进入fastboot模式（或者关机后同时按下音量`+/-`后，再长按开机键开机，即可进入fastboot模式）。

3、fastboot模式下执行`fastboot oem device-info`可以看到boot-loader目前解锁状态，执行`fastboot oem unlock`后出现一个界面让你选择，选择“unlock the bootloader”即可解锁。

```sh
$ fastboot oem unlock
...
OKAY [  0.037s]
finished. total time 0.038s
$ fastboot oem device-info
...
(bootloader) Verity mode: true
(bootloader) Device unlocked: true
(bootloader) Device critical unlocked: false
(bootloader) Charger screen enabled: true
(bootloader) enable_dm_verity: true
(bootloader) have_console: false
(bootloader) selinux_type: SELINUX_TYPE_INVALID
(bootloader) boot_mode: NORMAL_MODE
(bootloader) kmemleak_detect: false
(bootloader) force_training: 0
(bootloader) mount_tempfs: 0
OKAY [  0.020s]
finished. total time: 0.020s
```

## 3、刷recovery分区使用第三方recovery系统

解锁bootloader后，我们可以继续使用原来的操作系统，也可以自由刷机。刷机有几个部分可以刷，一个是用来恢复的`/recovery`分区，还有一个是正常使用的android操作系统也就是`/system`分区，还有一个就是`/boot`内核启动分区。

首先来说刷Recovery系统。

第三方的[Recovery系统有很多]( https://techsphinx.com/smartphones/best-custom-recovery-for-android-devices-2020)，其中最有名的就是[TWRP(Team Win Recovery Project)](https://twrp.me/about/)项目。

他们的项目在github上也开源了：https://github.com/TeamWin

TWRP支持的设备很多：https://twrp.me/Devices/，大部分机型都可以在这里面找到。

以oneplus5为例：

```sh
#将下载的twrp镜像刷入recovery分区
fastboot flash recovery twrp.img
#重启fastboot
fastboot reboot
```

请注意，许多设备会在启动期间自动替换自定义recovery。所以需要在刷完分区后第一次进入，按住键组合并启动到 TWRP。启动 TWRP 后，TWRP 将修补库存 ROM，以防止原有ROM 替换 TWRP。如果不遵循此步骤，将不得不重复安装。

## 4、使用magisk刷boot分区获取root权限

目前都是使用[magisk](https://github.com/topjohnwu/Magisk)来进行root权限管理，早期的supersu和kingroot已经被淘汰了，

1、从官方ROM包中，提取出`boot.img`，这个文件就是linux内核boot分区的镜像。

以oneplus5T为例，[OnePlus5T官方论坛](https://www.oneplusbbs.com/thread-4298129-1.html)提供了ROM包下载，解压后找到`boot.img`

```sh
total 6921336
drwxr-xr-x@  3 holmofy  staff          96  8  1 17:20 META-INF
drwxr-xr-x@  3 holmofy  staff          96  8  1 17:20 RADIO
-rw-r--r--@  1 holmofy  staff    23749928  1  1  2009 boot.img
-rw-r--r--@  1 holmofy  staff        8363  1  1  2009 compatibility.zip
drwxr-xr-x@ 16 holmofy  staff         512  8  1 17:20 firmware-update
-rw-r--r--@  1 holmofy  staff  2834780160  1  1  2009 system.new.dat
-rw-r--r--@  1 holmofy  staff           0  1  1  2009 system.patch.dat
-rw-r--r--@  1 holmofy  staff       13747  1  1  2009 system.transfer.list
-rw-r--r--@  1 holmofy  staff   677203968  1  1  2009 vendor.new.dat
-rw-r--r--@  1 holmofy  staff           0  1  1  2009 vendor.patch.dat
-rw-r--r--@  1 holmofy  staff        3375  1  1  2009 vendor.transfer.list
```

2、将`boot.img`拷贝到手机上，用`magisk.apk`将boot.img重写一遍，magisk会插入一段代码到`boot.img`中，生成新的`magisk_patched-23000_msSoR.img`（文件名后面是一个随机字符串）



3、将magisk生成的img文件拷贝到电脑上

```sh
adb reboot-bootloader  # 进入fastboot模式
fastboot flash boot magisk_patched-23000_msSoR.img  # 将magisk生成的镜像写入boot分区
fastboot reboot    # 重启手机
```

4、重启手机后，进入magisk应用，会发现已经root成功了。

5、好用的Magisk模块

https://github.com/Magisk-Modules-Repo

Refs: 

[Installation | Magisk (topjohnwu.github.io)](https://topjohnwu.github.io/Magisk/install.html#patching-images)

[目前ROOT成功率最高的软件是什么？ - 知乎 (zhihu.com)](https://www.zhihu.com/question/55892085/answer/1974101365)

## 5、第三方Android ROM操作系统

刷了`recovery`分区和`boot`分区，我们还可以刷`system`分区自定义的ROM，第三方的ROM多如牛毛(毕竟大多数就只是做做界面)：

Android Open Source Project: https://github.com/aosp-mirror

直接基于AOSP的扩展：https://github.com/AospExtended

谷歌自己的Pixel，自然就带了很多谷歌的产品：https://github.com/PixelExperience

基于Pixel魔改的：https://github.com/PixelExtended

小米的MIUI，带了很多小米的产品：https://github.com/MiCode

国内的一个开源ROM：https://github.com/MoKee

最小化的miniOS：https://github.com/StatiXOS

还有[lineageos](https://github.com/lineageos)、[GrapheneOS](https://github.com/GrapheneOS)、[RevengeOS](https://github.com/RevengeOS)、[arrowos](https://github.com/arrowos)、[aokp](https://github.com/aokp)、[Corvus-ROM](https://github.com/Corvus-ROM)、[BlissRoms](https://github.com/BlissRoms)、
[ProjectSakura)](https://github.com/ProjectSakura)、[Project-Xtended](https://github.com/Project-Xtended)...好多好多项目。

ROM开发

[Android-Kitchen](https://github.com/dsixda/Android-Kitchen)



## 6、Terminal on Android：Termux

在Android上玩终端，甚至将Android改装成Linux服务器，这些都可以用下面的软件实现。

[Termux](https://github.com/termux/)：提供了android端的终端命令以及`pkg`包管理器，也支持chroot方式挂在Linux发行版到指定目录，具体可以参考[PRoot - Termux Wiki](https://wiki.termux.com/wiki/PRoot#Installing_Linux_distributions)。

[meefik/linuxdeploy](https://github.com/meefik/linuxdeploy)：使用chroot将Linux发行版挂载到android的某个目录，也支持docker容器的方式



Refs:

^ https://android.gadgethacks.com/how-to/best-phones-for-rooting-modding-2020-0175988/

^ https://nexus5.gadgethacks.com/how-to/elementalx-only-custom-kernel-you-need-your-nexus-5-0157196/

^ https://android.gadgethacks.com/how-to/ultimate-guide-using-twrp-only-custom-recovery-youll-ever-need-0156006/

^ https://oneplus.gadgethacks.com/how-to/unroot-revert-your-oneplus-5-5t-100-stock-0182460/

^ https://forums.oneplus.com/threads/oneplus-5t-unlock-bootloader-flash-twrp-root-nandroid-efs.685403/

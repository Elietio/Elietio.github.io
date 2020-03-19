---
title: 谷歌SLOT A/B机制下安卓刷入TWRP和Magisk以及OTA的一些方法
tags:
  - 安卓
  - ROOT
categories:
  - 技术
mathjax: true
copyright: true
comment: true
photo: /img/magisk/magisk.png
abbrlink: d1df635e
date: 2019-10-12 22:14:49
---
A/B 系统分区是 Google 在 Android 7.0 时代引入的新机制，采用这个机制的设备拥有 A、B 两套系统分区，用户数据则能够在这两套系统分区之间共用。
<!-- more -->
这种分区机制带来的最大好处是无缝系统更新（seemless updates），当我们在 A 系统中进行 OTA 更新时，而实际更新的是另个一并未启用的 B 系统。手机重启后，系统分区从 A 切换到 B新系统，介于此机制，我们可以实现OTA升级后仍然保留Magisk的Root权限。由于使用了 A/B 分区，因此没有独立的 Recovery 分区；Recovery 现在是 Boot 的一部分。所以要通过 fastboot boot 实现临时从指定镜像启动，从而进入 TWRP，并通过刷入 twrp-installer 实现对 TWRP 的持久化。

## 解锁Bootloader 

1. 在「设置 - 关于手机」中点击 5 次「版本号」启用开发者选项,前往「设置 - 系统 - 开发者选项」，分别启用「OEM 解锁」和「高级重启」 
2. 长按电源键，从电源菜单中选择「引导器模式」，手机将会自动重启进入 Bootloader 模式。
3. 连接电脑，在cmd窗口内输入adb命令：
   ```cmd
   fastboot devices # 检查设备是否连接
   fastboot oem unlock
   ```
   回车，手机即出现解锁确认界面，音量键进行选择-「UNLOCK THE BOOTLOADER」，按电源键确认，手机重启开始解锁
   

## 刷入 TWRP

需要如下：
- 下载 [TWRP](https://twrp.me/Devices/)
- 需要 twrp.img 和 twrp-installer.zip 两个文件

执行以下操作：
1. 执行 `adb reboot bootloader` 进入 Bootloader 界面
2. 执行 `fastboot boot twrp.img` 进入临时 TWRP
3. 在TWRP中刷入twrp-installer.zip使TWRP持久化，安装包会在 AB 两个分区中都安装一次 TWRP，这样无论手机从哪个分区启动都可以进入 TWRP。

## 刷入Magisk获取Root

### 安装 Magisk（使用固化TWRP）

需要如下：
- 下载[Magisk-vX.X.zip 刷机包及 Magisk Manager.apk](https://forum.xda-developers.com/apps/magisk/official-magisk-v7-universal-systemless-t3473445)

执行以下操作：
1. 重启进去TERP
2. 在 TWRP 中刷入你下载的 Magisk 安装包，成功后重新启动手机
3. 安装 Magisk Manager 的 apk，确认 Magisk 已成功激活

### 安装 Magisk（使用临时 TWRP）

需要如下：
- 下载[Magisk-vX.X.zip 刷机包及 Magisk Manager.apk](https://forum.xda-developers.com/apps/magisk/official-magisk-v7-universal-systemless-t3473445)
- 下载 [TWRP](https://twrp.me/Devices/)

执行以下操作：
1. 执行 `adb reboot bootloader` 进入 Bootloader 界面
2. 执行 `fastboot boot TWRP.img` 进入临时 TWRP
3. 在 TWRP 中刷入你下载的 Magisk 安装包
   可选择Sideload 模式，TWRP 中输入正确密码解密分区，并选择 Advanced -> ADB Sideload。
   执行 `adb sideload Magisk-vX.X.zip`
4. `fastboot reboot`重新启动手机
5. 安装 Magisk Manager 的 apk，确认 Magisk 已成功激活

### 安装 Magisk（免 TWRP）

需要如下：
- 下载或提取ROM 所对应的 boot.img
- 下载[Magisk Manager.apk](https://forum.xda-developers.com/apps/magisk/official-magisk-v7-universal-systemless-t3473445)

执行以下操作：
1. 安装 Magisk Manager 的 apk。
2. 用 Magisk Manager 给 boot.img 手动补丁（` install —— 修补 boot 镜像文件`），将获得的 `magisk_patched.img` 传回电脑。
3. 再次进入 Bootloader，输入
   ```cmd
   fastboot boot magisk_patched.img
   fastboot flash boot magisk_patched.img #如果上面命令不生效可以使用这个，Android 10
   ```
   来加载生成后的 boot 分区文件获取**临时 root** 
   - 可使用`fastboot flash recovery magisk_patched.img`修复recovery image
4. 重启开机后，Magisk Manager 暂时有了 Root 权限，此时可以在 Magisk Manager 中正式安装 Magisk（`安装（install）——install——Direct Install（直接安装）`）。

## OTA时的操作

### 保留 TWRP
在 Magisk Manager 中下载并安装插件 TWRP A/B Retenion Script

### 保留 Root
- 能检测到 OTA 更新后，点击进入 Magisk Manager 应用，找到位于主界面的「卸载 Magisk」选项，然后点击「还原原厂镜像」，不要重启直接OTA(或许不需要还原这一步？但是magisk[官方文档](https://topjohnwu.github.io/Magisk/tutorials.html#ota-installation)有提及)
1. 使用 OTA 更新并安装（自动下载全量包），安装完成后不要重启
2. 在 Magisk Manager 中「安装 Magisk」
3. 选择「安装到未使用的槽位 (OTA 后)」(`Install to Inactive Slot (After OTA)`)
3. 写入完成后重启即可
# 安装
如果你已经安装了 Magisk,  **强烈建议直接通过 Magisk Manager 升级**。
以下教程适用于初次使用的用户。华为用户请查看具体小节，获取更多信息。

## 前提
- 如果准备安装自定义内核, 请在安装 Magisk**之后**刷入内核安装包。
- 确保已经删除任何 “boot image mods” ，比如其它 Root 方案（SuperSU）。最简单的方法是恢复原厂 Boot 镜像，或重刷**不自带 Root**的自定义 ROM。

## 自定义 Recovery
如果设备有自定义 Recovery 支持, 最简单的方法是通过它进行安装，比如 TWRP.

- 下载 Magisk 安装包
- 重启到自定义 Recovery
- 刷入安装包并重启
- 检查 Magisk Manager 是否安装，如果因为某些问题未能自动安装，请手动安装 Magisk Manager 应用

## 修补 Boot 镜像
这是安装 Magisk 的一个很酷的方式。 不管是因为你的设备没有一个良好的自定义 Recovery，或是正在使用 A/B 分区方案并且不希望 Recovery 与 Boot 镜像混在一起，还是有其它问题（比如[安装 OTA](tutorials.md#安装-ota)）, 均可以使用此安装方法。

要使用这个方法，需要有原厂 Boot 镜像, 可以通过解压 OEM 提供的工厂镜像，或是从 OTA 包内提取。如果无法独立完成这一步，请从互联网寻求帮助。 获得 Boot 镜像后，按照以下说明，完成安装：

- 复制 Boot 镜像文件到设备存储空间
- 下载并安装最新版 Magisk Manager
- 如果你计划通过 ODIN（Samsung 专用）刷入修补后的 Boot 镜像，需要转到 **设置 > 更新设定 > 已修补的 Boot 镜像输出格式**，将其调整为 *.img.tar*。
- 点按 **安装 > 安装 > 修补 Boot 镜像文件**，在弹出的文档应用内，找到并选中原始 Boot 镜像文件。
- Magisk Manager 会将 Magisk 安装进 Boot 镜像，并另存为`[内部存储空间]/Download/patched_boot.img[.tar]`
- 从手机复制修补后的文件到电脑上。如果你不能通过 MTP 找到它，可以通过 adb 拉取文件到电脑:`adb pull /sdcard/Download/patched_boot.img[.tar]`
- 刷入修补后文件并重启。 对于多数设备，使用 fastboot 命令：`fastboot flash boot patched_boot.img`

## 华为
使用麒麟处理器的华为设备采用了与常规设备不同的分区方法。 Magisk 通常安装在设备的`boot`分区，但华为设备没有这个分区。根据设备运行的 EMUI 版本，说明略有不同。即使设备已切换到自定义 ROM，仍需知道在切换之前，使用的是哪个版本的 EMUI。

### 获取原厂镜像
华为官方不提供原厂镜像下载，但多数设备的镜像都可以在[华为固件数据库](http://pro-teammt.ru/firmware-database/)中找到。要从压缩包内的 `UPDATE.APP` 提取镜像，需要使用[Huawei Update Extractor](https://forum.xda-developers.com/showthread.php?t=2433454)（仅 Windows）。

### EMUI 8
对于 EMUI 8 设备，它有一个名为`ramdisk`的分区，这将是 Magisk 的安装位置。

- 如果准备使用自定义 Recovery，只需按照前文的 Recovery 说明操作。比如 TWRP，首先下载 TWRP 镜像，然后使用`fastboot flash recovery_ramdisk /path/to/twrp.img`刷入 TWRP。
- 如果不使用自定义 Recovery，则必须从原厂镜像中提取`RAMDISK.img`。按照前文的 Boot 镜像修补说明进行操作，但使用`RAMDISK.img`文件而不是 Boot 镜像文件。要将修补后的镜像刷入设备，执行以下 fastBoot 命令：`fastboot flash ramdisk /path/to/patched_boot.img`。注意，需要刷入到`ramdisk`，而不是`boot`！

### EMUI 9
对于 EMUI 9 设备，`ramdisk`分区不再存在。因此，Magisk 将安装到`recovery_ramdisk`分区。**这意味着每次重启时，都必须启动到 Recovery 模式。这也意味着无法同时拥有 Magisk 和自定义 Recovery！** 要启动到 Recovery 模式，请在开机时按住**电源键和音量加**。

- 如果准备使用自定义 Recovery 安装 Magisk，只需按照前文的 Recovery 说明操作。比如 TWRP，首先下载 TWRP 镜像，然后使用`fastboot flash recovery_ramdisk /path/to/twrp.img`刷入 TWRP。**Magisk 安装后会覆盖自定义 Recovery！**
- 如果不使用自定义 Recovery，则必须从原厂镜像中提取`RECOVERY_RAMDIS.img`。按照前文的 Boot 镜像修补说明进行操作，但使用`RECOVERY_RAMDIS.img`文件而不是 Boot 镜像文件。要将修补后的镜像刷入设备，执行以下 fastboot 命令：`fastboot flash recovery_ramdisk /path/to/patched_boot.img`。注意，需要刷入到 `recovery_ramdisk`，不是 `boot` 也不是 `ramdisk`！

# Magisk 工具
Magisk 为开发人员提供了大量实用工具。本文档涵盖了3个二进制文件和它们包含的部件。二进制文件和其部件如下所示：

```
magiskboot                 /* binary */
magiskinit                 /* binary */
magiskpolicy -> magiskinit
supolicy -> magiskinit
magisk                     /* binary */
magiskhide -> magisk
resetprop -> magisk
su -> magisk
imgtool -> magisk
```

备注：Magisk 安装包内只有 `magiskboot` 和 `magiskinit`。`magisk` 已被嵌入 `magiskinit`。 推送 `magiskinit` 到设备并执行 `./magiskinit -x magisk <path>` 即可提取 `magisk` 二进制文件到指定位置。

### magiskboot
一个能够解包、打包 boot 镜像；解析、修补、提取 cpio；修补 dtb；hex 修补二进制文件和使用多种算法压缩、解压文件的工具。

`magiskboot` 本身支持 (即不依赖外部工具) 常见压缩格式，包括 `gzip`、`lz4`、`lz4_legacy` ([仅 LG 使用](https://events.static.linuxfound.org/sites/events/files/lcjpcojp13_klee.pdf))、`lzma`、`xz` 和 `bzip2`。

`magiskboot` 的目的是让 Boot 镜像修改更简单。 解包时，它会解析镜像标头和所有部分（kernel, ramdisk, second, dtb, extra），检测每个部分使用的压缩格式，并在提取时解压。每个提取出的部分都是原始数据，可以直接修改。打包时，需要原始 Boot 镜像，使用它的标头，仅修改必要条目 如大小及校验和，最后用原始格式压缩每个部分。

此工具也支持各种 CPIO 操作，可以直接修改 CPIO 档案而无需提取或打包。

```
Usage: ./magiskboot <action> [args...]

Supported actions:
  --unpack <bootimg>
    Unpack <bootimg> to kernel, ramdisk.cpio, and if available, second, dtb,
    and extra into the current directory. Return values:
    0:valid    1:error    2:chromeos    3:ELF32    4:ELF64

  --repack <origbootimg> [outbootimg]
    Repack kernel, ramdisk.cpio[.ext], second, dtb... from current directory
    to [outbootimg], or new-boot.img if not specified.
    It will compress ramdisk.cpio with the same method used in <origbootimg>,
    or attempt to find ramdisk.cpio.[ext], and repack directly with the
    compressed ramdisk file

  --hexpatch <file> <hexpattern1> <hexpattern2>
    Search <hexpattern1> in <file>, and replace with <hexpattern2>

  --cpio <incpio> [commands...]
    Do cpio commands to <incpio> (modifications are done directly)
    Each command is a single argument, use quotes if necessary
    Supported commands:
      rm [-r] ENTRY
        Remove ENTRY, specify [-r] to remove recursively
      mkdir MODE ENTRY
        Create directory ENTRY in permissions MODE
      ln TARGET ENTRY
        Create a symlink to TARGET with the name ENTRY
      mv SOURCE DEST
        Move SOURCE to DEST
      add MODE ENTRY INFILE
        Add INFILE as ENTRY in permissions MODE; replaces ENTRY if exists
      extract [ENTRY OUT]
        Extract ENTRY to OUT, or extract all entries to current directory
      test
        Test the current cpio's patch status
        Return values:
        0:stock    1:Magisk    2:unsupported (phh, SuperSU, Xposed)
      patch KEEPVERITY KEEPFORCEENCRYPT
        Ramdisk patches. KEEP**** are boolean values
      backup ORIG [SHA1]
        Create ramdisk backups from ORIG
        SHA1 of stock boot image is optional
      restore
        Restore ramdisk from ramdisk backup stored within incpio
      magisk ORIG KEEPVERITY KEEPFORCEENCRYPT [SHA1]
        Do Magisk patches and backups all in one step
        Create ramdisk backups from ORIG
        KEEP**** are boolean values
        SHA1 of stock boot image is optional
      sha1
        Print stock boot SHA1 if previously stored

  --dtb-<cmd> <dtb>
    Do dtb related cmds to <dtb> (modifications are done directly)
    Supported commands:
      dump
        Dump all contents from dtb for debugging
      test
        Check if fstab has verity/avb flags
        Return values:
        0:no flags    1:flag exists
      patch
        Search for fstab and remove verity/avb

  --compress[=method] <infile> [outfile]
    Compress <infile> with [method] (default: gzip), optionally to [outfile]
    <infile>/[outfile] can be '-' to be STDIN/STDOUT
    Supported methods: gzip xz lzma bzip2 lz4 lz4_legacy

  --decompress <infile> [outfile]
    Detect method and decompress <infile>, optionally to [outfile]
    <infile>/[outfile] can be '-' to be STDIN/STDOUT
    Supported methods: gzip xz lzma bzip2 lz4 lz4_legacy

  --sha1 <file>
    Print the SHA1 checksum for <file>

  --cleanup
    Cleanup the current working directory
```

### magiskinit
这个文件会替换 Boot 镜像 ramdisk 里的 `init` 文件，这对于支持 system-as-root 设备（所有 A/B 设备及一些奇怪的变种如华为 EMUI 9）来说是必须的。但它也支持传统设备，因此能在所有设备上使用相同的安装方法。更多信息请参考[Magisk 启动流程](details.md#magisk-booting-process)的**Pre-Init**小节。

### magiskpolicy
（此工具有一个 `supolicy` 的别名，用于兼容 SuperSU 的 sepolicy 工具）

`magiskinit` 的部件，可用于高级开发人员修改 SELinux 策略。在管理 Linux 服务器时，一般是修改 SELinux 策略源 (`*.te`) 然后重新编译 `sepolicy` 二进制文件，但在 Android 上我们直接修改二进制文件（或使用运行时策略）。

所有 Magisk 进程，包括 root shell 及其子进程，都运行在 `u:r:magisk:s0` 环境。所有安装了 Magisk 的系统所使用的规则都可以被视为原始 `sepolicy` 加以下修补： `magiskpolicy --magisk 'allow magisk * * *'`。

```
Usage: magiskpolicy [--options...] [policy statements...]

Options:
   --live            directly apply sepolicy live
   --magisk          inject built-in rules for a minimal
                     Magisk selinux environment
   --load FILE       load policies from FILE
   --compile-split   compile and load split cil policies
                     from system and vendor just like init
   --save FILE       save policies to FILE

If neither --load or --compile-split is specified, it will load
from current live policies (/sys/fs/selinux/policy)

One policy statement should be treated as one parameter;
this means a full policy statement should be enclosed in quotes;
multiple policy statements can be provided in a single command

The statements has a format of "<action> [args...]"
Use '*' in args to represent every possible match.
Collections wrapped in curly brackets can also be used as args.

Supported policy statements:

Type 1:
"<action> source-class target-class permission-class permission"
Action: allow, deny, auditallow, auditdeny

Type 2:
"<action> source-class target-class permission-class ioctl range"
Action: allowxperm, auditallowxperm, dontauditxperm

Type 3:
"<action> class"
Action: create, permissive, enforcing

Type 4:
"attradd class attribute"

Type 5:
"typetrans source-class target-class permission-class default-class (optional: object-name)"

Notes:
- typetrans does not support the all match '*' syntax
- permission-class cannot be collections
- source-class and target-class can also be attributes

Example: allow { source1 source2 } { target1 target2 } permission-class *
Will be expanded to:

allow source1 target1 permission-class { all-permissions }
allow source1 target2 permission-class { all-permissions }
allow source2 target1 permission-class { all-permissions }
allow source2 target2 permission-class { all-permissions }
```


### magisk
当此文件作为 `magisk` 这个名字使用时, 是一个有许多辅助功能的实用程序，及 `init` 启动 Magisk 服务的入口点。

```
Usage: magisk [applet [arguments]...]
   or: magisk [options]...

Options:
   -c                        print current binary version
   -v                        print running daemon version
   -V                        print running daemon version code
   --list                    list all available applets
   --install [SOURCE] DIR    symlink all applets to DIR. SOURCE is optional
   --daemon                  manually start magisk daemon
   --[init trigger]          start service for init trigger
   --unlock-blocks           set BLKROSET flag to OFF for all block devices
   --restorecon              fix selinux context on Magisk files and folders
   --clone-attr SRC DEST     clone permission, owner, and selinux context

Supported init triggers:
   startup, post-fs-data, service

Supported applets:
    magisk, su, resetprop, magiskhide, imgtool
```

### su
`magisk` 的部件，MagiskSU 的入口点。 Good old `su` command.

```
Usage: su [options] [-] [user [argument...]]

Options:
  -c, --command COMMAND         pass COMMAND to the invoked shell
  -h, --help                    display this help message and exit
  -, -l, --login                pretend the shell to be a login shell
  -m, -p,
  --preserve-environment        preserve the entire environment
  -s, --shell SHELL             use SHELL instead of the default /system/bin/sh
  -v, --version                 display version number and exit
  -V                            display version code and exit
  -mm, -M,
  --mount-master                force run in the global mount namespace
```

备注：尽管没有列出 `-Z, --context` 选项，它依然存在，以便兼容为 SuperSU 开发的应用。 但该选项会被忽略，没有任何效果。

### resetprop
`magisk` 的部件，用于操作系统属性。更多信息请查阅 [Resetprop](details.md#resetprop)。

```
Usage: resetprop [flags] [options...]

Options:
   -h, --help        show this message
   (no arguments)    print all properties
   NAME              get property
   NAME VALUE        set property entry NAME with VALUE
   --file FILE       load props from FILE
   --delete NAME     delete property

Flags:
   -v      print verbose output to stderr
   -n      set properties without init triggers
           only affects setprop
   -p      access actual persist storage
           only affects getprop and deleteprop
```

### magiskhide
`magisk` 的部件，用于控制 MagiskHide。它会连接到守护进程，修改 MagiskHide 设置。

```
Usage: magiskhide [--options [arguments...] ]

Options:
  --enable          Start magiskhide
  --disable         Stop magiskhide
  --add PROCESS     Add PROCESS to the hide list
  --rm PROCESS      Remove PROCESS from the hide list
  --ls              Print out the current hide list
```

### imgtool
`magisk` 的部件，用于创建和管理 `ext4` 镜像。

```
Usage: imgtool <action> [args...]

Actions:
   create IMG SIZE      create ext4 image. SIZE is interpreted in MB
   resize IMG SIZE      resize ext4 image. SIZE is interpreted in MB
   mount  IMG PATH      mount IMG to PATH and prints the loop device
   umount PATH LOOP     unmount PATH and delete LOOP device
   merge  SRC TGT       merge SRC to TGT
   trim   IMG           trim IMG to save space
```

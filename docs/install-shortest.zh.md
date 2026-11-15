# 最短刷入路径：小米 MIX 2S（polaris）

这一页只做一件事：把 [install.md](install.md)、[STORAGE.md](../devices/polaris/STORAGE.md)、
[building.md](getting-started/building.md) 里散着的步骤压成一条能照着敲的路径，并且说清楚哪一步绕不过去。

## 0. 一句话

| 你的 MIX 2S | 能不能装 | 去哪一节 |
|---|---|---|
| 已经有 128 GiB 的 `darwin` 分区（移植时改装过的机器） | 能，官方路径 | 第 3 节 |
| 原厂分区表 | 官方路径走不通 | 第 4 节 |
| 只想看它跑起来，不碰手机 | 能 | 第 4.1 节（QEMU） |

先确认属于哪一种。手机进 recovery（OrangeFox / TWRP）后：

```sh
adb shell 'readlink -f /dev/block/by-name/darwin; blockdev --getsize64 /dev/block/by-name/darwin'
```

输出的两行必须正好是 `/dev/block/sda22` 和 `137438932992`。是别的值，或者第一行报错，说明这是原厂分区表，
安装器会拒绝写入（`unexpected darwin device` 或 `partition capacity differs from the layout`），请看第 4 节。

## 1. 刷入写的是两个地方，不是一个

| 分区 | 大小 | 写进去的东西 | 怎么写的 |
|---|---|---|---|
| `boot` | 64 MiB | 内核、UEFI loader、`flashboot.apfs`（md0）、`root.apfs`（md1） | `fastboot flash boot` |
| `darwin` | 128 GiB | System 10 / OS 4 / App 1 GiB 三个固定槽，Data 约 113 GiB | recovery 里的 adb + `tools/polaris-storage.py` |
| `userdata` | — | 不动 | — |

**只刷 `boot` 是起不来的**：boot 里只有内核、引导器和两个小的 ramdisk，Darwin 的系统盘在 `darwin` 里。
所以「刷入」= 写系统 + 刷 boot，两步都要做，而且要用同一份源码构建出来的镜像。

## 2. 构建一次，两条命令刷机

已经有 `darwin`、也已经有旧系统时，日常更新就是：

```sh
make boot                                                   # 出 out/boot/polaris.img
# recovery 里：snapshot → pack → apply-system（见 3.5）
fastboot flash boot out/boot/polaris.img
```

第一次装则要多写一次系统盘和 Data，见 3.3。

## 3. 有 darwin 分区的机器：完整路径

### 3.1 一次性的准备

| 需要 | 具体要求 | 怎么确认 |
|---|---|---|
| 解锁的 bootloader | `fastboot devices` 有输出 | 小米解锁工具 |
| Recovery | OrangeFox 或 TWRP，带 root adb；能读 `/dev/block/by-name` | `adb devices` 显示 recovery |
| 主机 | Ubuntu 24.04（UEFI 的 `build_setup.sh` 按它写）或 macOS | `uname -a` |
| 磁盘 | 12 GB 以上空闲（解 IPSW 要 8 GB） | `df -h .` |
| 7-Zip | **23 或更新**（旧版读不了 APFS） | `7z \| head -2` |
| theos | 带 Linux 工具链和 `iPhoneOS16.5.sdk` | `ls ~/theos/toolchain/linux/iphone/bin/ld` |
| IPSW | `iPhone10,3,iPhone10,6_16.5_20F66_Restore.ipsw`，**6,165,698,198 字节** | `wc -c` |

那个 IPSW 是 iPhone X（iPhone10,3 / iPhone10,6）的 iOS 16.5 固件，项目不分发也不替你下载，得你自己有。
字节数不对 `sysimage/unpack.sh` 会直接拒绝。

### 3.2 构建（一次性，约一小时，机器在干活）

```sh
git clone https://github.com/6ewjso6-gif/hakios-xnu-kernel.git hakios
cd hakios
git submodule update --init mu_aloha_platforms
make doctor                       # 缺什么它会说

# 1) UEFI。MIX 2S 只能用 mu_aloha 的 UEFI：
#    HakiOS 的 U-Boot 只支持 POCO X5 Pro 和 QEMU，不支持 polaris。
uefi/apply.sh                     # 取 mu_aloha_platforms 并打补丁
( cd mu_aloha_platforms && ./build_setup.sh && pip install --upgrade -r pip-requirements.txt )
uefi/apply.sh build               # 10–30 分钟，产出 SDM845_EFI.fd

# 2) 把你自己的 IPSW 摊成 Darwin 用户空间（约 8 GB 临时占用，之后 5.3 GB）
make unpack IPSW=$HOME/Downloads/iPhone10,3,iPhone10,6_16.5_20F66_Restore.ipsw

# 3) 内核 + loader + 三个 APFS 镜像 + 打成 boot.img
make boot
```

产物：

| 文件 | 用途 |
|---|---|
| `out/boot/polaris.img` | 刷进 `boot`，上限 64 MiB |
| `userland/system.apfs` | 写进 `darwin` 的 System 槽 |
| `userland/flashboot.apfs` | boot 里的 md0，带 `/sbin/launchd` |
| `userland/root.apfs` | boot 里的 md1，挂到 `/polaris` |

### 3.3 第一次写系统（recovery 里，中间不要重启）

```sh
adb reboot recovery
adb devices                       # 必须能看到 recovery

# 先把原机 boot 备份出来，回退全靠它
adb shell 'dd if=/dev/block/by-name/boot of=/tmp/stock-boot.img'
adb pull /tmp/stock-boot.img stock-boot.img

# 可选但建议：连分区表、vbmeta、persist、modemst* 一起备份（recovery 里要有 su）
python3 tools/backup-partitions.py --yes

python3 tools/polaris-storage.py plan
python3 tools/polaris-storage.py snapshot out/storage/before-01
python3 tools/polaris-storage.py pack \
  --system userland/system.apfs \
  --os out/storage/before-01/os.apfs \
  --app out/storage/before-01/app.apfs \
  -o out/storage/system-update.json
# 空分区/旧布局的首次写入加 --first-install；已经有固定布局的机器不要加
python3 tools/polaris-storage.py apply-system out/storage/system-update.json --first-install
```

`--first-install` 只在 `darwin` 还不是固定布局时用（空分区、或老的连续 APFS 布局）。写完 System 再建 Data，
**只建一次**；`apply-data` 遇到已经有 HFS+/HFSX 卷会拒绝，更新时永远不要跑它：

```sh
# macOS 用 Apple 的 newfs_hfs；Linux 去掉最后一个参数，用 PATH 里的 mkfs.hfsplus
python3 tools/mkdatavol.py --layout configs/polaris-storage.json \
  out/storage/data-initial.simg /sbin/newfs_hfs
python3 tools/polaris-storage.py apply-data out/storage/data-initial.simg --initialize-data
```

### 3.4 刷 boot

```sh
adb reboot bootloader
fastboot devices
fastboot flash boot out/boot/polaris.img
fastboot reboot
```

开机后会直接进 Darwin；`screen_log=0` 在默认配置里，所以屏幕上看不到日志，日志仍然写到 `logdump` 分区。

### 3.5 以后更新

系统或内核任何一个变了，都要把两边一起更新，且用同一次构建的产物：

```sh
make boot
adb reboot recovery
python3 tools/polaris-storage.py snapshot out/storage/before-update-02   # 每次换新目录
python3 tools/polaris-storage.py pack \
  --system userland/system.apfs \
  --os out/storage/before-update-02/os.apfs \
  --app out/storage/before-update-02/app.apfs \
  -o out/storage/system-update.json
python3 tools/polaris-storage.py apply-system out/storage/system-update.json
adb reboot bootloader && fastboot flash boot out/boot/polaris.img && fastboot reboot
```

`apply-system` 先校验整份清单，再逐段写入并读回比对 SHA-256；已经一致的范围会跳过。写失败就留在 recovery，
不要带着半套系统开机。

### 3.6 回退到 Android

`darwin` 和 Android 的 `userdata` 是两个分区，刷回原厂 boot 就回到 Android，数据还在：

```sh
adb reboot bootloader
fastboot flash boot stock-boot.img
fastboot reboot
```

刷回去之后 `darwin` 里的东西还在，只是没人读它；分区表和 Android 的 `userdata` 全程没有被改过。

## 4. 原厂分区表的机器

### 4.1 不写机：QEMU 里先跑通

最快、也最安全的一步。用的是同一套 IPSW、userland、loader 产物，等于在碰手机之前先把构建链验证一遍：

```sh
make doctor                                  # 检查 qemu-system-aarch64、ramfb、UEFI 固件
make unpack IPSW=$HOME/Downloads/iPhone10,3,iPhone10,6_16.5_20F66_Restore.ipsw
REST=1 userland/build.sh                     # userland/root.apfs 与 userland/system.apfs
make loader                                  # prebuilt/loader.efi 没有 Apple 身份，必须自己编
qemu/run-darwin.sh 1200 -display gtk > logs/qemu.log
```

CPU 是模拟的：先黑屏约一分钟，约一分钟出 Apple 标志，约五分钟进主屏。只看内核、不要 Apple 文件的话是 `make qemu-open`。

### 4.2 要真装机：得先有 `darwin`

原厂 MIUI 的分区表里没有这个分区，它是移植时从 `userdata` 末尾切出来的（作者那台是 128 GiB）。
现成的工具只有 `tools/userdata-carve.py`，但上游文档明确写了**不要对 polaris 用它**（`STORAGE.md` 与
`install-safe.md` 都写了），原因是布局容量必须精确等于 `137438932992` 字节，切错一点安装器就整段拒绝，
而且这一步会清空 Android 数据。

所以只有在你清楚代价、并且手机是 256 GB 版本时，才有得谈：

1. 备份 GPT：`python3 tools/backup-partitions.py --yes`（需要 root），把 `backups/` 留在电脑上；
2. 把它写出的 `gpt-sda-head.bin` / `gpt-sda-tail.bin` 复制成 carve 要的名字：
   `cp backups/gpt-sda-head.bin backups/gpt-head.bin && cp backups/gpt-sda-tail.bin backups/gpt-tail.bin`；
3. `python3 tools/userdata-carve.py --wipe backups 128 darwin`，按它打印的 `dd` 命令写回两块 GPT；
4. 重启进 recovery，确认 `blockdev --getsize64 /dev/block/by-name/darwin` **正好**是 `137438932992`，
   不是就停，用备份的 GPT 还原；
5. `fastboot -w`，然后按 3.3 写系统、3.4 刷 boot。

这一步不在官方支持范围内，出问题需要自己用分区备份恢复。128 GB 机型切不出 128 GiB，这条路直接不成立。

## 5. 为什么没有「一键刷机包」

1. **boot 里没有系统。** 系统盘是 `userland/system.apfs`，写在 `darwin` 里，只能走 recovery 的 adb 写块设备。
2. **Apple 的文件不在仓库里，也不能替你下载。** `system.apfs` 只能从你自己的 IPSW 生成（6.16 GB）。
   所以没有「下载一个 zip 刷进去」这种办法，任何号称这样的包都不该信。
3. **MIX 2S 没有 U-Boot 支持。** 想从内存启动也得先编 EDK2 的 UEFI（10–30 分钟）。
4. **`darwin` 不是原厂分区。** 安装器按 `/dev/block/sda22` + 精确容量校验，普通 MIX 2S 过不了这一关。

## 6. 出错对照

| 报错 | 原因 | 怎么办 |
|---|---|---|
| `unexpected darwin device` | 分区不是 `sda22`，或根本没有 `darwin` | 原厂表，回第 4 节 |
| `partition capacity differs from the layout` | 容量不是 `137438932992` | 同上，别硬写 |
| `darwin is mounted; unmount it before accessing the layout` | recovery 自动挂上了 | `adb shell umount /dev/block/by-name/darwin` |
| `mkxnuimg.py: the UEFI of ... is not built` | 没编 UEFI | `uefi/apply.sh build` |
| `required input missing or empty: userland/root.apfs` | 没跑 `make unpack` 或 userland | 按 3.2 的顺序重来 |
| `image is ... configured boot partition limit is 67108864` | boot 镜像超过 64 MiB | 用默认 `polaris-boot.conf`，别加 `REST=1` |
| `adbd cannot run as root` / 写块设备被拒 | recovery 没给 root adb | 换 OrangeFox/TWRP，或在其设置里开 adb root |

# FRDM-IMX8MP UUU 烧录 eMMC 指南

> 适用：BSP i.MX Linux 6.6.52_2.2.0 (scarthgap) + meta-imx-frdm 层，目标板 **NXP FRDM-IMX8MP**（LPDDR4，机器 `imx8mpfrdm`），镜像 imx-image-multimedia，工具 NXP UUU。
> 原理：板子进入 Serial Download 模式后，UUU 先通过 USB 把 SPL+U-Boot 下载到板子 RAM 运行，再借 U-Boot 的 fastboot 把 bootloader 和完整 wic 镜像（分区表 + boot 分区 + rootfs）写入 eMMC。
> 注意：**本文档按 FRDM-IMX8MP 编写**（拨码开关为 SW5，串口为 J19），与 i.MX8M Plus EVK 的 SW2/J901 不同。

---

## 一、准备

### 1.1 需要的文件

在 `build/tmp/deploy/images/imx8mpfrdm/` 下（全部由 Yocto 产出，直接用，无需手工转换）：

| 文件 | 用途 |
|------|------|
| `imx-boot-imx8mpfrdm-sd.bin-flash_evk` | SPL+ATF+U-Boot+firmware 合并镜像（含 FRDM DDR/板级支持），写到 eMMC boot 区 |
| `imx-image-multimedia-imx8mpfrdm.rootfs.wic.zst` | 完整 wic 镜像（GPT 分区表 + boot 分区 + rootfs），UUU 自动解压 |

> 时间戳按实际构建为准；也可用不带时间戳的符号链接。

### 1.2 UUU 工具

下载 NXP 官方 UUU（免安装单文件）：

```
https://github.com/nxp-imx/mfgtools/releases
```

- Windows：下载 `uuu.exe`。需要 **1.4.x 及以上**（支持 `-b emmc_all` 和 `.zst` 自动解压）。
- Linux / WSL：下载 Linux 版 `uuu`，`chmod +x uuu`。
- Windows 10/11：ROM 的 HID 设备免驱；U-Boot fastboot 设备（0525:a4a5）需用 Zadig 装 WinUSB 驱动（见"常见问题"）。

### 1.3 硬件

- 5V Type-C 电源（FRDM 板供电口）
- USB OTG 线：接板子 **J3（USB OTG，Type-C）** 口
- 调试串口：接 **J19（Debug UART，Type-C）**，板载 CH342F 虚拟串口，115200 8N1，无需额外串口线

---

## 二、烧录步骤

### 第 1 步：板子进入 Serial Download 模式

拨码开关 **SW5**（4 位）设置为 **0001**（即 SW5-1/2/3 = OFF，SW5-4 = ON）：

| SW5[4:1] | 模式 |
|----------|------|
| 0010 | **eMMC 启动**（默认） |
| 0011 | SD 卡启动 |
| **0001** | **Serial Download（UUU 烧录模式）** |
| 0000 | 内部 fuse |

> 改拨码必须断电重启才生效。

### 第 2 步：连接并确认 USB 设备

1. USB OTG 线接 **J3**，另一端接 PC；J19 接 PC 看串口 log。
2. 上电。
3. 确认 PC 识别到设备（VID:PID = `1fc9:0146`）：

```cmd
:: Windows，在工作目录下
uuu.exe -lsusb
:: 应输出类似： Known devices: 0000:1FC9:0146
```

看不到设备时：换数据线（不是充电线）、换 USB 口（避开 hub）、拔插重试、检查 SW5 是否为 0001。

### 第 3 步：烧录文件

烧录文件已复制到 Windows `E:\imx8mp-flash\`（也可用仓库里的 `copy_flash_files.sh` 重新复制）：

```
E:\imx8mp-flash\
├── uuu.exe
├── imx-boot-imx8mpfrdm-sd.bin-flash_evk          (~2.1 MB)
└── imx-image-multimedia-imx8mpfrdm.rootfs.wic.zst (~958 MB)
```

### 第 4 步：烧录

```cmd
cd /d E:\imx8mp-flash
uuu.exe -v -b emmc_all imx-boot-imx8mpfrdm-sd.bin-flash_evk imx-image-multimedia-imx8mpfrdm.rootfs.wic.zst
```

一条命令完成全部烧录：

1. 通过 SDP 协议下载 SPL+U-Boot 到板子 RAM 并运行；
2. 把 imx-boot 写入 eMMC boot 区（含 `mmc partconf` 使能 boot 分区）；
3. 把整个 wic（分区表 + boot 分区 + rootfs）写入 eMMC 用户区。

> - `-v` 显示详细 log。整个流程约 5~15 分钟（rootfs ~958 MB，取决于 USB 速度）。
> - **第一次烧录时**，U-Boot 起来后会以 fastboot 设备（VID 0525:PID a4a5）重新枚举，Windows 若无驱动，UUU 会一直停在 "Wait for Known USB Device Appear..."——此时用 Zadig 给 "USB download gadget" 装 WinUSB 驱动（不要动 1fc9:0146 的 HID 设备），UUU 会立刻继续。
> - 串口（J19）应能看到 SPL/U-Boot log，U-Boot 会显示 "Model: ... FRDM ..."。

### 第 5 步：切回 eMMC 启动

1. UUU 显示 `Done` 后断电。
2. **SW5 拨回 0010（eMMC 启动）**。
3. 重新上电，J19 串口（115200）应能看到 U-Boot 和 Linux 启动 log。

---

## 三、验证

### 3.1 串口 log

正常应看到（节选）：

```
U-Boot SPL ...
DDRINFO: start DRAM init
DDRINFO: ddrphy calibration done
...
U-Boot 2024.04-...
Model: NXP i.MX8MPlus FRDM board     <- FRDM 设备树生效的标志
...
Starting kernel ...
```

> 之前用 EVK 镜像时 U-Boot 会显示 "Model: NXP i.MX8MPlus LPDDR4 EVK board" 并在 `fastboot 0` 处报 `USB init failed: -22`（TCPC 找不到）——那是设备树错配，本镜像已修正。

### 3.2 登录检查

串口登录（root，无密码）或 `ssh root@<board-ip>`：

```bash
lsblk         # 确认 rootfs 从 eMMC 启动（FRDM 上 eMMC 通常是 mmcblk2）
uname -a      # 6.6.52
ip addr       # DHCP 自动获取（以太网 PHY 为 YT8521，FRDM 补丁已带支持）
ps aux | grep weston
```

> NXP 镜像默认通过 extlinux 启动，bootargs 由 U-Boot 自动拼好，不要手动覆盖 bootcmd。

---

## 四、U-Boot 调试速查

上电时在串口按任意键进 U-Boot 命令行：

```bash
mmc dev 2                       # eMMC（以 printenv / lsblk 实际编号为准）
mmc part                        # 查看分区表
printenv                        # 当前环境
env default -a; saveenv         # 恢复默认环境（启动异常时先试这个）
fastboot 0                      # 进 fastboot 模式（重新烧录）
run bootcmd                     # 重跑启动
```

---

## 五、常见问题排查

| 症状 | 排查 |
|------|------|
| `uuu: USB device not found` | SW5 是否 0001；线是否插 J3 OTG 口；换线/换口；重新上电 |
| SDP 阶段 `LIBUSB_ERROR_PIPE (-9)` | 先启动 uuu 再给板子上电；直插 PC 的 USB 2.0 口；换线 |
| SDPS Okay 后长时间无反应 | **等 10 秒左右**，看设备管理器是否出现 "USB download gadget"；用 Zadig 装 WinUSB 驱动；看 J19 串口 U-Boot 是否起来 |
| U-Boot 显示 EVK board / `USB init failed: -22` | 烧的是 EVK 版 imx-boot，重新烧 FRDM 版（imx-boot-imx8mpfrdm-*） |
| 烧完重启无反应 | SW5 是否拨回 0010；重新烧一遍（boot 区可能损坏） |
| `VFS: Unable to mount root fs` | `lsblk` 确认 eMMC 设备号；U-Boot 里 `mmc dev 2; mmc part` 确认分区 |
| UUU 中途报错失败 | 重新上电（回到 SDP 模式）重烧即可，可重复烧录无副作用 |

---

## 六、重新构建（imx8mpfrdm 机器）

镜像由标准 BSP 6.6.52_2.2.0 + `meta-imx-frdm` 层（tag `imx-frdm-4.0`）构建。层已加入 `build/conf/bblayers.conf`，重构建：

```bash
cd ~/imx-yocto-bsp
source sources/poky/oe-init-build-env build
MACHINE=imx8mpfrdm bitbake imx-image-multimedia
```

> 对 meta-imx-frdm 做过的本地适配（因该层基于 6.6.36，部分补丁在 6.6.52 中已包含/冲突）：
> - `linux-imx_6.6.bbappend`：删除了 0001/0020/0021/0022/0036 五个补丁（已在 6.6.52 内核中或目标 commit 不存在）
> - `0042-...iw612-otbr-dts.patch`：Makefile hunk 已按 6.6.52 现状修正（保留 ap1302 条目）
> - `firmware-nxp-wifi_%.bbappend`：删除重复的 `PACKAGES += nxpiw610-sdio`（meta-imx 6.6.52 已添加）
> - `conf/layer.conf`：IMAGE_BOOT_FILES 加 `mcore-demos/` 前缀（6.6.52 部署路径变化）
> - `build/conf/local.conf`：追加 `ERROR_QA:remove:pn-u-boot-imx = "patch-fuzz"`
> - 原 `imx8mpevk` 机器的构建不受影响，可随时用 `MACHINE=imx8mpevk` 重编

---

## 七、音频说明（SOF）

- FRDM 层**未提供 SOF 设备树**（`imx8mp-evk-sof-*.dtb` 不适用于 FRDM 板）。
- FRDM 默认音频走传统 ALSA 驱动；`imx8mp-frdm-8mic.dtb` 对应 8MIC 音频扩展板。
- 若需要在 FRDM 上跑 SOF（HiFi4 DSP），需基于 `imx8mp-frdm.dts` 自行移植 DSP 节点 + `sof-imx8mp.ri` 固件（固件已在镜像里，`/lib/firmware/imx/sof/`）。

---

## 八、附录

### 8.1 调试串口（J19）

FRDM-IMX8MP 的调试口是 J19 USB Type-C（CH342F 芯片），插上 PC 后出现两个虚拟 COM 口，**第一个**是 A 核调试口。设置 115200 8N1 无流控。

工具：Windows 用 PuTTY/MobaXterm，Linux 用 `screen /dev/ttyUSB0 115200` 或 minicom/picocom。

### 8.2 拨码开关（SW5）

| SW5[4:1] | 模式 |
|----------|------|
| 0010 | eMMC 启动 |
| 0011 | SD 卡启动 |
| 0001 | USB Serial Download |
| 0000 | 内部 fuse |

### 8.3 UUU 调试选项

| 选项 | 作用 |
|------|------|
| `-v` | 详细 log |
| `-d` | 调试 log |
| `-dry` | 只显示将执行的命令，不实际烧录 |
| `-lsusb` | 列出检测到的 USB 设备 |
| `-t N` | 超时毫秒数 |

### 8.4 烧到 SD 卡（备用方案）

```bash
zstd -d imx-image-multimedia-imx8mpfrdm.rootfs.wic.zst -o rootfs.wic
sudo dd if=rootfs.wic of=/dev/sdX bs=1M conv=fsync
# SW5 拨 0011（SD 启动）
```

---

**最后更新**：2026-08-29
**对应 BSP**：i.MX Linux 6.6.52_2.2.0 (scarthgap) + meta-imx-frdm (imx-frdm-4.0)
**对应 UUU**：1.4.x 及以上
**适用板子**：NXP FRDM-IMX8MP (imx8mpfrdm)

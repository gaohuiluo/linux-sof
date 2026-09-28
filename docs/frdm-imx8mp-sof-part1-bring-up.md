# 上板：让 SOF 在 FRDM-i.MX8MP 上跑起来

> 系列第一篇。总览见 [frdm-imx8mp-sof-porting-guide.md](frdm-imx8mp-sof-porting-guide.md)。

这一篇是"能开机跑 SOF"的最短路径。整篇只依赖 bitbake，跟主机的 Python/cmake 工具链完全无关，改 DSP 源码才需要那些（第二篇）。

好消息是，FRDM 上跑 SOF 的四样东西里，三样 BSP 已经帮你备好，真正要动手的只有一件：**移植 FRDM 的 SOF 设备树**。

## 先盘一下 FRDM 的现状

启动 SOF 需要四个组件。在 FRDM 这块板子上：

| 组件 | FRDM 现状 | 结论 |
|---|---|---|
| 内核 SOF 驱动 | defconfig 里 `CONFIG_SND_SOC_SOF_OF=m`、`CONFIG_SND_SOC_SOF_IMX8M=m`、`CONFIG_SND_SIMPLE_CARD=y`、`CONFIG_SND_SOC_WM8962=m` 都开着 | 已就绪，内核不用改 |
| SOF 固件 `.ri` | FRDM 构建默认没装（`firmware-sof-imx` 依赖的 `use-nxp-bsp` override 在 FRDM 镜像未生效）。好在也没装旧包，不会有文件冲突 | 手动加 `sof-zephyr` |
| Topology `.tplg` | 随 `sof-zephyr` 一起安装 | 手动加时一并解决 |
| SOF 设备树 DTB | 内核树里只有 EVK 的 sof 变体，FRDM 一个都没有 | **唯一缺口，需自己移植** |

所以这一篇的重心，就是从 EVK 模板出发，改出一份 FRDM 能用的 SOF 设备树，然后接进 Yocto、编译、烧录、上板验证。

`.config` 里的驱动配置可自行核对：`build/tmp/work/imx8mpfrdm-poky-linux/linux-imx/6.6.52+git/build/.config`。

## 第一步：建一个自定义 layer

把 FRDM 的定制内容都放进一个独立 layer，不动 NXP 原始层。

```bash
cd ~/imx-yocto-bsp/sources
mkdir -p meta-myboard/conf
mkdir -p meta-myboard/recipes-kernel/linux/linux-imx

cat > meta-myboard/conf/layer.conf <<'EOF'
BBPATH .= ":${LAYERDIR}"
BBFILE_COLLECTIONS += "meta-myboard"
BBFILE_PATTERN_meta-myboard = "^${LAYERDIR}/"
# BBFILES 要能匹配三级目录 recipes-kernel/linux/linux-imx/*.bbappend
BBFILES += "${LAYERDIR}/recipes-*/*/*.bb ${LAYERDIR}/recipes-*/*/*.bbappend \
            ${LAYERDIR}/recipes-*/*/*/*.bb ${LAYERDIR}/recipes-*/*/*/*.bbappend"
LAYERSERIES_COMPAT_meta-myboard = "scarthgap"
LAYERVERSION_meta-myboard = "1"
EOF

cd ~/imx-yocto-bsp
source setup-environment build
bitbake-layers add-layer ../sources/meta-myboard
```

`BBFILES` 里那两行的意思是：既匹配两级目录的 recipe，也匹配三级目录（内核 bbappend 就放在 `recipes-kernel/linux/linux-imx/` 这样的三级路径下），漏掉三级那行的话 bbappend 不会被识别。

## 第二步：移植 FRDM 的 SOF 设备树

这是整篇唯一真正的开发工作。模板选 `imx8mp-evk-revb4-sof-wm8962.dts`（因为它也是 WM8962，跟 FRDM 同 codec），基板换成 `imx8mp-frdm.dts`。

先用 devtool 把内核源码拉进 workspace：

```bash
cd ~/imx-yocto-bsp/build
devtool modify linux-imx
cd workspace/sources/linux-imx/arch/arm64/boot/dts/freescale
cp imx8mp-evk-revb4-sof-wm8962.dts imx8mp-frdm-sof-wm8962.dts
```

改造后的完整内容如下：

```dts
// SPDX-License-Identifier: GPL-2.0+
// FRDM-IMX8MP SOF + WM8962，移植自 imx8mp-evk-revb4-sof-wm8962.dts

#include "imx8mp-frdm.dts"

/ {
	/* 删掉基板的传统 ALSA 声卡节点（SOF 接管 SAI3） */
	/delete-node/ sound-wm8962;

	reserved-memory {
		/* FRDM 基板没有 DSP 保留内存，这里是"新增"而非"删除" */
		dsp_reserved: dsp@92400000 {
			reg = <0 0x92400000 0 0x2000000>;
		};
	};

	sof-sound-wm8962 {
		compatible = "simple-audio-card";
		label = "wm8962-audio";
		simple-audio-card,bitclock-master = <&sndcodec>;
		simple-audio-card,frame-master = <&sndcodec>;
		/* FRDM 耳机检测在 IO 扩展芯片 pcal6416_1 的 pin 2（EVK 是 gpio4 28） */
		simple-audio-card,hp-det-gpio = <&pcal6416_1 2 GPIO_ACTIVE_HIGH>;
		simple-audio-card,widgets =
			"Headphone", "Headphones",
			"Microphone", "Headset Mic",
			"Speaker", "Speaker";
		simple-audio-card,routing =
			"Headphones", "HPOUTL",
			"Headphones", "HPOUTR",
			"Speaker", "SPKOUTL",
			"Speaker", "SPKOUTR",
			"Headset Mic", "MICBIAS",
			"IN1R", "Headset Mic",
			"IN1L", "Headset Mic";
		simple-audio-card,dai-link {
			format = "i2s";
			cpu {
				sound-dai = <&dsp 1>;
			};
			sndcodec: codec {
				sound-dai = <&wm8962>;
			};
		};
	};
};

/* WM8962 节点基板已定义（I2C 地址、时钟、供电都全），只需补 sound-dai-cells */
&wm8962 {
	#sound-dai-cells = <0>;
	status = "okay";
};

/* DSP 节点：与 EVK revb4 SOF 版完全一致 */
&dsp {
	#sound-dai-cells = <1>;
	compatible = "fsl,imx8mp-dsp";
	reg = <0x0 0x3B6E8000 0x0 0x88000>;

	pinctrl-names = "default";
	pinctrl-0 = <&pinctrl_sai3>;

	power-domains = <&audiomix_pd>;

	assigned-clocks = <&clk IMX8MP_CLK_SAI3>;
	assigned-clock-parents = <&clk IMX8MP_AUDIO_PLL1_OUT>;
	assigned-clock-rates = <12288000>;
	clocks = <&audio_blk_ctrl IMX8MP_CLK_AUDIOMIX_OCRAMA_IPG>,
		<&audio_blk_ctrl IMX8MP_CLK_AUDIOMIX_DSP_ROOT>,
		<&audio_blk_ctrl IMX8MP_CLK_AUDIOMIX_DSPDBG_ROOT>,
		<&audio_blk_ctrl IMX8MP_CLK_AUDIOMIX_SAI3_IPG>, <&clk IMX8MP_CLK_DUMMY>,
		<&audio_blk_ctrl IMX8MP_CLK_AUDIOMIX_SAI3_MCLK1>, <&clk IMX8MP_CLK_DUMMY>,
		<&clk IMX8MP_CLK_DUMMY>,
		<&audio_blk_ctrl IMX8MP_CLK_AUDIOMIX_SDMA3_ROOT>;
	clock-names = "ipg", "ocram", "core",
		"sai3_bus", "sai3_mclk0", "sai3_mclk1", "sai3_mclk2", "sai3_mclk3",
		"sdma3_root";

	mbox-names = "txdb0", "txdb1", "rxdb0", "rxdb1";
	mboxes = <&mu2 2 0>, <&mu2 2 1>,
		 <&mu2 3 0>, <&mu2 3 1>;
	memory-region = <&dsp_reserved>;
	/delete-property/ firmware-name;

	tplg-name = "sof-imx8mp-wm8962.tplg";
	machine-drv-name = "asoc-simple-card";
	status = "okay";
};

/* SAI3/SDMA3 交给 DSP 驱动，传统驱动停用 */
&sai3 {
	status = "disabled";
};
&sdma3 {
	status = "disabled";
};
```

跟 EVK 模板逐项对照，改动点就这五处：

| EVK 模板里的写法 | FRDM 上怎么处理 | 原因 |
|---|---|---|
| `#include "imx8mp-evk.dts"` | 换成 `imx8mp-frdm.dts` | 基板不同 |
| `/delete-node/ &codec`（wm8960）+ 在 i2c3 里重定义 wm8962 | 整段删掉 | FRDM 基板本就定义好了 `wm8962@1a`，直接复用 |
| `hp-det-gpio = <&gpio4 28 0>` | `<&pcal6416_1 2 GPIO_ACTIVE_HIGH>` | FRDM 耳机检测走 IO 扩展芯片 |
| `/delete-node/ dsp_reserved_heap` 等 | 删掉这些行，改成**新增** `dsp_reserved` | FRDM 基板没这些节点，delete 会编译报错 |
| routing 含 `"DMICDAT", "Digital Mic"` | 删掉 | 数字麦是 EVK revb4 的配置，FRDM 板载没有 |

如果第一次编 dts 报 `label not found`，基本都是保留了模板里针对 EVK 节点的 `/delete-node/`，按上表清理即可。

## 第三步：把新 dts 接进构建

开发阶段直接用 devtool 改 workspace 验证；稳定后 `devtool finish linux-imx ../sources/meta-myboard` 把 patch 收编到 layer 里。内核 bbappend（`recipes-kernel/linux/linux-imx/linux-imx_%.bbappend`）负责带上这份 patch，machine conf 用 `KERNEL_DEVICETREE:append` 把 DTB 加进构建（参照 `imx8mp-lpddr4-frdm.conf` 的写法）。

这里有个**必须知道的坑**，否则会浪费一整轮编译烧录：只把 `KERNEL_DEVICETREE:append` 写在内核 bbappend 里是不够的。

内核 bbappend 是 recipe 作用域，它能让 DTB 编出来、部署到 deploy 目录。但 wic 镜像的 boot 分区装哪些文件，是由**镜像作用域**的 `IMAGE_BOOT_FILES = "... ${@make_dtb_boot_files(d)}"` 决定的，它读的是镜像作用域的 `KERNEL_DEVICETREE`，看不到 recipe 作用域里 append 的那份。结果就是：DTB 有了、也在 deploy 里，但**没进 boot 分区**，U-Boot 启动时报 `Failed to load 'imx8mp-frdm-sof-wm8962.dtb'`。

所以要在 `local.conf`（或 machine conf）里再补一份镜像作用域的：

```bash
# build/conf/local.conf —— 这一条是 boot 分区能带上 DTB 的关键
KERNEL_DEVICETREE:append:imx8mpfrdm = " freescale/imx8mp-frdm-sof-wm8962.dtb"
```

验证有没有生效：`bitbake -e imx-image-multimedia | grep ^IMAGE_BOOT_FILES=`，输出里应该能看到 `imx8mp-frdm-sof-wm8962.dtb`。也可以查 `tmp/sysroots/imx8mpfrdm/imgdata/imx-image-multimedia.env`（wic 构建 boot 分区的输入）。

顺便确认一下 MACHINE 没搞混：`bitbake -e | grep ^MACHINE=` 应该是 `imx8mpfrdm`。如果 `local.conf` 里 `MACHINE ??=` 还写着 `imx8mpevk`，改过来——build 目录下 EVK 和 FRDM 两个 workdir 常常并存，很容易拿错。

## 第四步：加上 SOF 固件包

FRDM 构建默认没装任何 SOF 固件包，直接加 NXP 新版 Zephyr 固件即可。这个 `sof-zephyr-2.11.0` tarball 里有 `sof-zephyr-gcc/sof-imx8m.ri`、`sof-zephyr-xcc/sof-imx8m.ri` 和 `sof-tplg/sof-imx8mp-wm8962.tplg`（默认 `sof` 软链接指向 xcc 版）：

```bash
# build/conf/local.conf
MACHINE_FIRMWARE:append = " sof-zephyr"   # 装 /lib/firmware/imx/sof-zephyr-gcc/ + sof-tplg/ + sof 软链接
IMAGE_INSTALL:append = " sof-tools"       # sof-logger / sof-ctl 等调试工具
```

## 第五步：编译、烧录、上板验证

先编内核（含新 DTB）和镜像：

```bash
bitbake linux-imx
bitbake imx-image-multimedia
```

产物有两个，都在 `~/imx-yocto-bsp/build/tmp/deploy/images/imx8mpfrdm`：

```bash
ls -la imx-boot-imx8mpfrdm-sd.bin-flash_evk                      # U-Boot
ls -la imx-image-multimedia-imx8mpfrdm.rootfs-*.wic.zst          # 整卡镜像
```

wic 的分区布局是 p0=imx-boot（32KiB 偏移 rawcopy）| p1=/boot vfat 256MiB | p2=rootfs ext4，所以 uuu 只需要上面这两个文件。

### 用 uuu 烧到 eMMC

uuu 在 Windows 上跑最省事（WSL 下要配 usbipd 转发 USB，不推荐）。完整细节见 `docs/uuu-emmc-flash-guide.md`，这里是速查版。

1. 把两个文件拷到 Windows：`imx-boot-imx8mpfrdm-sd.bin-flash_evk` 和 `imx-image-multimedia-imx8mpfrdm.rootfs-<时间戳>.wic.zst`（uuu ≥1.4 直接吃 .zst，无需解压）。

2. 板子进串行下载模式：拔电 → 拨码开关 **SW5 设为 0001** → USB OTG 线接板子 **J3** 连电脑 → 上电。

3. 在两个文件所在目录，Windows cmd 里烧录：

   ```bat
   uuu -lsusb
   REM 确认列出 1FC9:0146 设备再继续
   uuu -v -b emmc_all imx-boot-imx8mpfrdm-sd.bin-flash_evk imx-image-multimedia-imx8mpfrdm.rootfs-<时间戳>.wic.zst
   ```

   它会先经 USB 把 SPL+U-Boot 下到 RAM，再把 imx-boot 写进 eMMC boot 区、整个 wic 写进用户区，约 5~15 分钟，看到 `Done` 即成功。如果 uuu 卡在 "Wait for Known USB Device Appear..."，用 Zadig 给 "USB download gadget" 装 WinUSB 驱动。

4. **SW5 拨回 0010（eMMC 启动）**，重新上电，进 U-Boot 选 DTB：

   ```
   setenv fdtfile imx8mp-frdm-sof-wm8962.dtb
   saveenv
   boot
   ```

   两点提醒：烧录会覆盖 eMMC 里保存的环境变量，所以每次烧完都要重设一次 `fdtfile`；输入时务必用**英文输入法/半角字符**，中文输入法的全角字母在终端里看着一样，但 U-Boot 会报 `Unknown command`（也可以用等价的 `env set` / `env save`）。

> 备用：烧到 SD 卡的话，`zstd -d xxx.wic.zst -o xxx.wic && sudo dd if=xxx.wic of=/dev/sdX bs=1M conv=fsync`，SW5 拨 0011（SD 启动）。只有 dd 这条路才要手工解压。

### 板上验证

```bash
dmesg | grep -i sof        # 看到 firmware info / topology ABI 即成功
cat /proc/asound/cards     # 应出现 wm8962-audio，由 sof 驱动
aplay -l && arecord -l
aplay /usr/share/sounds/alsa/Front_Center.wav
```

一个互斥关系要记住：**不要**同时用 `imx8mp-frdm-rpmsg.dtb`（M 核 rpmsg 音频）和 SOF DTB，两者都占着 `&dsp`，只能二选一。如果声卡出现了但功能混乱，先 `cat /proc/device-tree/model` 和确认当前 `fdtfile`，多半是启到了 rpmsg 那份。

## 让录音真正出声：WM8962 的 mixer 设置

播放通了不代表录音就能用。FRDM 的 3.5mm 口是 **CTIA 标准的四段（TRRS）耳麦复合插孔**（HP 和 MIC 共用一个孔，见 NXP UM12340），麦克风接到 WM8962 的 `IN1R`/`IN1L`。而录音链路 `IN1 → INPGA → MIXIN → ADC` 各级**出厂默认全关**，第一次录音前必须手动打开。

一个命名上的注意点：ASoC 的 DAPM 控件注册时会自动去掉 `Switch`/`Volume` 后缀，比如 `INPGAR IN1R Switch` 在 amixer 里叫 `INPGAR IN1R`。以 `amixer -c 0 scontrols` 的实际输出为准。

```bash
# ① 打开麦克风通路（IN1 → PGA → MIXIN → ADC）
amixer -c 0 sset 'INPGAR IN1R' on
amixer -c 0 sset 'INPGAL IN1L' on
amixer -c 0 sset 'MIXINR PGA' 45%
amixer -c 0 sset 'MIXINL PGA' 45%
amixer -c 0 sset 'Capture' 70%
amixer -c 0 sset 'Digital' 60%

# ② 关掉所有未接线的输入引脚（IN2/IN3/IN4 悬空会灌入底噪）
amixer -c 0 sset 'MIXINR IN3R' 0;  amixer -c 0 sset 'MIXINL IN3L' 0
amixer -c 0 sset 'MIXINR IN2R' 0;  amixer -c 0 sset 'MIXINL IN2L' 0
amixer -c 0 sset 'INPGAR IN2R' off; amixer -c 0 sset 'INPGAR IN3R' off; amixer -c 0 sset 'INPGAR IN4R' off
amixer -c 0 sset 'INPGAL IN2L' off; amixer -c 0 sset 'INPGAL IN3L' off; amixer -c 0 sset 'INPGAL IN4L' off

# ③ 芯片级降噪
amixer -c 0 sset 'Capture HPF' on          # 高通滤波，去低频嗡嗡声
amixer -c 0 sset 'ADC High Performance' on # ADC 高性能模式，降低本底噪声
```

调参思路：底噪大先降 `MIXINx PGA`（模拟前级增益对噪声贡献最大），说话音量偏小再升 `Capture`/`Digital` 补偿。要记住每一级增益都会等比放大底噪。

如果录出来是**纯雪花/白噪声**，基本就是输入悬空。先用 debugfs 读芯片寄存器排除软件原因：`grep ^0019: /sys/kernel/debug/regmap/2-001a/registers`（bit1 是 MICBIAS，要在录音运行中读，录音一结束 DAPM 就断电了）。寄存器都对却还是无声或噪声，那就是耳麦的问题——三段耳机没有麦克风引脚，OMTP 老国标插头的麦克风与地反接也录不出来。

验证录音（`-vv` 会显示实时电平条，对着耳麦说话应看到电平起伏）：

```bash
arecord -D plughw:0,0 -f S16_LE -r 48000 -c 2 -vv /tmp/test.wav
aplay -D plughw:0,0 /tmp/test.wav
```

## 到这里你已经有什么

板上 `dmesg | grep sof` 有输出、`wm8962-audio` 声卡在、能播放能录音——SOF 已经在 FRDM 上跑起来了，用的是 NXP 预编译的固件和 topology。

想改 DSP 里的算法，下一步就是自己从源码把 `.ri` 和 `.tplg` 编出来。这需要一套主机工具链，见第二篇：[进阶：从源码编译固件与 topology](frdm-imx8mp-sof-part2-build-from-source.md)。

## 附：本篇相关的常见问题

- **`firmware file not found`**：固件要放在 NXP 路径 `/lib/firmware/imx/sof/`，文件名是 `sof-imx8m.ri`（内核 imx8m.c 硬编码，是 imx8m 不是 imx8mp）。别照上游文档放 `/lib/firmware/intel/sof/`。
- **DTB 编译报 `label not found`**：保留了 EVK 模板里的 `/delete-node/ dsp_reserved_heap` 等行，FRDM 基板没这些节点，删掉。
- **U-Boot `Failed to load '...dtb'`**：DTB 编出来了但没进 wic boot 分区。原因和修法见第三步——`local.conf` 里要再 append 一份镜像作用域的 `KERNEL_DEVICETREE`。
- **烧录后 U-Boot 加载了默认 dtb**：烧录覆盖了保存的环境变量，`fdtfile` 回到默认。每次烧完重设 `fdtfile` 并 `saveenv`。
- **U-Boot 报 `Unknown command 'setenv'`**：字符问题，中文输入法的全角字母在终端里看着一样但 U-Boot 不认。切英文输入法逐字输入，或用 `env set` / `env save`。
- **声卡在但功能混乱**：boot 用了 `imx8mp-frdm-rpmsg.dtb`，rpmsg 音频占用了 DSP。确认 `fdtfile` 指向 SOF DTB。
- **codec 认成 WM8960**：照 WM8960 的 EVK 例子抄 routing 会没声音，本篇的 dts 已按 WM8962 改好。
- **MACHINE 混用**：`build/tmp/work` 下 `imx8mpevk-*` 和 `imx8mpfrdm-*` 两个 workdir 并存，改完 `local.conf` 确认 `bitbake -e | grep ^MACHINE=` 是 `imx8mpfrdm`。
- **`dsp@92400000` 32MB 保留内存冲突**：若与 rootfs/显存规划（如 framebuffer CMA）撞车报 ioremap 失败，先调 reserved-memory 布局。
- **播放正常但录音静音**：按上文 mixer 一节打开录音通路；且必须是四段 TRRS 耳麦。
- **WSL 性能**：所有编译都在 WSL 文件系统内（`~/...`）做，不要在 `/mnt/c/` 下做，跨 UNC 路径的 grep/find 极慢。Windows 侧只当编辑器和串口终端。

# 在 FRDM-i.MX8MP 上玩转 SOF：从上板到自研算法（系列总览）

这是一个记录：如何在一块 **FRDM-IMX8MP** 开发板上，把 SOF（Sound Open Firmware）从零跑起来，再进一步把自己的音频算法写进 DSP 固件里。

- **板子**：FRDM-IMX8MP（NXP 官方 LPDDR4 版，`MACHINE=imx8mpfrdm`），不是常见的 EVK。
- **BSP**：i.MX Linux 6.6.52_2.2.0（scarthgap）。
- **主机环境**：WSL2 + Ubuntu 20.04。
- **目标**：先在板上把 SOF 跑通（能播放、能录音），再把自研算法部署到 HiFi4 DSP 上验证。

文中所有结论都在本机的 BSP 源码树和 `~/sof-dev` 工作区里逐一编译验证过（recipe、内核 defconfig、dts、firmware、topology 都实际跑过），并给出了文件路径，可自行复查。设备树部分是基于 EVK 模板加 FRDM 基板差异推导的，首次编译如遇 label 报错按文中说明微调即可。

---

## 先说清楚两件最容易搞混的事

在动手之前，有两个问题几乎每个人都会卡一下。先把它们讲透，后面就顺了。

### SOF 源码在 Yocto 里吗？要额外下载吗？

取决于你要"用"还是"改"：

| 你要做什么 | 要额外下载源码吗 | 原因 |
|---|---|---|
| 只跑 SOF（播放/录音） | 不用 | 内核驱动在 `linux-imx` 里，固件是 NXP 预编译包 |
| 改内核里的 SOF 驱动 | 不用 | 驱动源码随 `linux-imx` 一起在 Yocto 内，用 devtool 改 |
| 改 DSP 端固件（写算法） | **要** | BSP 里的 `sof-zephyr` recipe 拉的是 NXP 预编译产物，树里没有能编出 `.ri` 的 recipe |

一个省事的小技巧：`bitbake sof-tools -c unpack` 会把完整的 `thesofproject/sof` 仓库（`imx-stable-v2.11` 分支）解到 `build/tmp/work/imx8mpfrdm-poky-linux/sof-tools/2.11.0-r0/git/`，可以直接从这里起步建 west 工作区，省一次 clone。

### 编 HiFi4 固件需要付费 License 吗？

**不需要，只要你不走 Cadence XCC 那条路。** 三条工具链路线：

| 路线 | 工具链 | License | 个人开发者可用性 |
|---|---|---|---|
| 用 BSP 预编译固件 | 无需编译 | 随 NXP BSP EULA | 立即可用 |
| GCC 开源路线（推荐） | Zephyr SDK 捆绑版，或 crosstool-NG 自建 | 无 | 完全可用 |
| Cadence XCC | Xtensa XCC | 商业授权 + NDA | 基本拿不到 |

GCC 和 XCC 的唯一差别在性能优化库：XCC 带 Cadence 闭源的 HiFi4 优化库，GCC 没有，但完全能编出功能正确的固件。学习和部署自研算法走 GCC 就够了，追求极限性能再考虑 XCC。SOF 官方文档也明确 i.MX 平台支持这两条工具链，GCC 路线标注为 "no license"。

---

## 全链路长什么样

整个系统要动三个**互相独立**的构建域，最后三个产物落到板上拼在一起：

```
┌──────────────────────────────────────────────────────────────┐
│ 域 1  Yocto (bitbake)      产物：Image + DTB + rootfs          │
│   内核 SOF 驱动（已就绪）+ 你移植的 FRDM SOF 设备树 + 固件包    │
├──────────────────────────────────────────────────────────────┤
│ 域 2  SOF/Zephyr (west)    产物：sof-imx8m.ri（DSP 固件）       │
│   你的算法编译进这里 ← 后期的主战场                            │
├──────────────────────────────────────────────────────────────┤
│ 域 3  Topology (alsatplg)  产物：sof-imx8mp-wm8962.tplg         │
│   描述 pipeline，你的算法挂在哪条音频链上                      │
└──────────────────────────────────────────────────────────────┘
        ↓ 三个产物都落到板上
/lib/firmware/imx/sof/sof-imx8m.ri              ← 固件（注意文件名是 imx8m 不是 imx8mp）
/lib/firmware/imx/sof-tplg/sof-imx8mp-wm8962.tplg
```

三个域之间只有一个约束：ABI 版本要对齐（固件、topology、内核驱动的 IPC ABI 必须同代，本系列全程在 2.11 分支内自洽）。域 1 只依赖 bitbake，跟主机的 Python/cmake 无关；域 2、域 3 依赖一套主机工具链。

FRDM 与 EVK 的硬件差异，决定了这块板子唯一真正要开发的东西是设备树：

- 板载 codec 是 **WM8962**（挂在 SAI3，I2C 地址 0x1a），不是 EVK 老版的 WM8960。
- 耳机检测接在 IO 扩展芯片 `pcal6416_1` 的 pin 2，EVK 是 `gpio4 28`。
- FRDM 基础 dts 已有传统 ALSA 声卡节点 `sound-wm8962`，SOF 要把它删掉、由 DSP 接管 SAI3。
- FRDM 基础 dts **没有** DSP 专用保留内存，SOF dts 要新增 `dsp_reserved`（EVK 是删已有节点，方向相反）。

---

## 这个系列怎么读

按"先跑起来 → 再自己编 → 最后写算法"的顺序，拆成三篇。只想先听个响的，读完第一篇就够了。

1. **[上板：让 SOF 在 FRDM 上跑起来](frdm-imx8mp-sof-part1-bring-up.md)**
   最短路径。只用 bitbake，不碰主机工具链。核心就一件开发工作：移植 FRDM 的 SOF 设备树。做完能在板上 `dmesg | grep sof` 看到输出，能播放、能录音。

2. **[进阶：从源码编译固件与 topology](frdm-imx8mp-sof-part2-build-from-source.md)**
   搭好主机工具链（Python 3.11、west、Zephyr SDK、alsatplg），从 SOF 源码编出自己的 `.ri` 和 `.tplg`，替换掉第一篇用的 NXP 预编译件，跑通同样的测试。这是写算法的前提。

3. **[实战：把自研算法写进 DSP 固件](frdm-imx8mp-sof-part3-custom-dsp-algorithm.md)**
   在 SOF 里 processing module 是什么、怎么写、怎么注册、怎么用 topology 挂进音频链，以及最快的开发迭代循环和实时约束。

---

## 参考资料

- SOF 官方构建指南（含 crosstool-NG 方案与 "no license" 说明）：<https://thesofproject.github.io/latest/getting_started/build-guide/build-from-scratch.html>
- SOF 官方 NXP i.MX8 用户指南：<https://thesofproject.github.io/latest/platforms/nxp/nxp_imx8.html>
- SOF 仓库（用 `imx-stable-v2.11` 分支，与 BSP 对齐）：<https://github.com/thesofproject/sof>
- NXP i.MX8M Plus + SOF 社区 wiki：<https://github.com/thesofproject/sof/wiki/NXP-i.MX8M-Plus>
- Zephyr SDK releases：<https://github.com/zephyrproject-rtos/sdk-ng/releases>
- ALSA 源码（alsa-lib / alsa-utils，提供 alsatplg）：<https://www.alsa-project.org/files/pub/>
- 本项目其他文档：eMMC 烧录 `docs/uuu-emmc-flash-guide.md`；Yocto 侧修改方法 `docs/yocto-sof-dev-workflow.md`

---

**最后更新**：2026-08-30
**对应 BSP**：i.MX Linux 6.6.52_2.2.0（scarthgap），MACHINE=imx8mpfrdm
**对应 SOF**：imx-stable-v2.11（Zephyr），固件路径 `/lib/firmware/imx/sof/`

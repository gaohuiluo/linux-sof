# 进阶：从源码编译 SOF 固件与 topology

> 系列第二篇。上一篇 [上板：让 SOF 跑起来](frdm-imx8mp-sof-part1-bring-up.md) 用的是 NXP 预编译固件。这一篇搭好主机工具链，从源码编出自己的 `.ri` 和 `.tplg`，替换掉预编译件跑通同样的测试。这也是第三篇写算法的前提。

第一篇只靠 bitbake。这一篇不同，它依赖一整套主机工具链，而 Ubuntu 20.04 自带的版本几乎全都太老。所以本篇分两半：**先一次性把环境搭好，再讲编译**。环境不齐就往下走，会在编译时反复卡安装问题。

本机的 `~/sof-dev` 就是按下面的步骤搭起来的，每节都给了验证命令，照着自检即可。

## 环境总览：四样东西

| 组件 | 为什么需要 | 装在哪 |
|---|---|---|
| Python ≥ 3.10 + cmake ≥ 3.20 + ninja | 新版 Zephyr/SOF 构建的硬性下限 | `~/sof-dev/python311` + `~/.local/bin` |
| west + sof/zephyr 工作区 | 多仓库管理，拉齐 SOF 依赖 | `~/sof-dev` |
| DSP 交叉工具链（Xtensa GCC） | 编 HiFi4 固件 `.ri` | `~/sof-dev/zephyr-sdk` 或自建 |
| alsatplg ≥ 1.2.5 | 编 topology `.tplg` | `~/sof-dev/alsa-local` |

## 主机构建工具：Python 3.11 + cmake + ninja

这版 Zephyr 要求 **Python ≥ 3.10**（`zephyr/cmake/modules/python.cmake` 里的 `PYTHON_MINIMUM_REQUIRED 3.10`），而且 `WEST_PYTHON` 优先——**west 本身必须跑在新 Python 上**，只给 cmake 传 `-DPython3_EXECUTABLE` 没用。Ubuntu 20.04 系统只有 3.8。免 sudo 的办法是用便携版 python-build-standalone，解压即用：

```bash
# 经 ghfast 加速下载，-C - 断点续传防超时
curl -L -C - -o /tmp/py311.tar.gz "https://ghfast.top/https://github.com/astral-sh/python-build-standalone/releases/download/20260825/cpython-3.11.16%2B20260825-x86_64-unknown-linux-gnu-install_only.tar.gz"
mkdir -p ~/sof-dev/python311 && tar -xzf /tmp/py311.tar.gz -C ~/sof-dev/python311 --strip-components=1
```

再装构建工具。Zephyr 要求 **cmake ≥ 3.20**（`zephyr/cmake/modules/zephyr_default.cmake` 的 `cmake_minimum_required(VERSION 3.20.0)`），Ubuntu 20.04 的 apt 只有 3.16，所以 **cmake 用 pip 装、ninja/gcc 用 apt 装**：

```bash
sudo apt install -y ninja-build build-essential
python3 -m pip install --user "cmake>=3.20,<4" -i https://pypi.tuna.tsinghua.edu.cn/simple
which cmake && cmake --version   # 应是 ~/.local/bin/cmake 且 ≥3.20；若还是 /usr/bin/cmake(3.16) 就 export PATH=$HOME/.local/bin:$PATH
```

**验证**：`~/sof-dev/python311/bin/python3 --version` 是 3.11.x；`cmake --version` ≥ 3.20；`ninja --version` 有输出。

## west 与 sof/zephyr 工作区

west 是 Zephyr 的多仓库管理工具，要装进上一步的 3.11 便携 Python，否则新版 Zephyr 会因 Python 太老报错：

```bash
P=~/sof-dev/python311/bin/python3
$P -m pip install -U pip west anytree -i https://pypi.tuna.tsinghua.edu.cn/simple
export PATH=$HOME/sof-dev/python311/bin:$HOME/.local/bin:$PATH
west --version   # 能打印版本号即可
```

（如果用系统 3.8 装 west 会报 `Could not find a version that satisfies the requirement ruamel.yaml`，这正是要把 west 装进 3.11 的原因。）

建工作区，用与 BSP 6.6.52 对齐的 NXP 维护分支：

```bash
mkdir -p ~/sof-dev && cd ~/sof-dev
git clone https://github.com/thesofproject/sof.git sof
cd sof && git checkout imx-stable-v2.11
cd .. && west init -l sof && west update
```

装 Zephyr 构建依赖，**只装 base 子集**：

```bash
P=~/sof-dev/python311/bin/python3
$P -m pip install -r ~/sof-dev/zephyr/scripts/requirements-base.txt -i https://pypi.tuna.tsinghua.edu.cn/simple
# 之后若再报 No module named 'xxx'，照方抓药：$P -m pip install xxx
```

为什么只装 base：完整的 `requirements.txt` 会串带 build-test/run-test/extras/compliance 四个文件（twister 测试、CI 用），其中 `pylint>=3` 不支持老 Python 会直接报 `from versions: none`；而且这个分支根本没有 `requirements-build.txt`。base 子集里已经包含 anytree/pyelftools/PyYAML/pykwalify，够用了。

`west update` 拉 GitHub 超时（`Operation too slow`）几乎是国内网络的必经之路。两个办法：有代理就 `git config --global http.proxy http://<代理IP>:7890`（代理软件记得开"允许局域网连接"，IP 用 `grep nameserver /etc/resolv.conf` 拿到的那个）；没代理就用镜像重写 `git config --global url."https://ghfast.top/https://github.com/".insteadOf "https://github.com/"`，克隆完再 `git config --global --unset` 撤销。修好后回到 `~/sof-dev` 直接重跑 `west update` 就行——`west init` 只需成功一次，换目录再 init 反而会报 `no west.yml found`。

**验证**：`ls ~/sof-dev/zephyr` 存在；`cd ~/sof-dev/sof && git rev-parse --abbrev-ref HEAD` 输出 `imx-stable-v2.11`。

## DSP 固件工具链：Xtensa GCC

编 HiFi4 固件 `.ri` 需要 Xtensa 交叉工具链，也就是系列总览里说的"License 问题"所在。按你的目标选路线：

| 目标 | 用哪条 | 对应编译时的环境变量 |
|---|---|---|
| 不改 DSP 源码（只做第一篇） | 不用装，用 BSP 预编译固件 | 无 |
| 要改 DSP 源码（本篇及以后） | **Zephyr SDK**（首选，解压即用） | `ZEPHYR_SDK_INSTALL_DIR` + `ZEPHYR_TOOLCHAIN_VARIANT=zephyr` |
| SDK 里没有 imx8m 工具链时 | crosstool-NG 自建 GCC | `XTENSA_TOOLCHAIN_PATH` + `ZEPHYR_TOOLCHAIN_VARIANT=xtensa` |
| （参考）商业 XCC | 需 Cadence/NXP 授权，通常拿不到 | — |

两条 GCC 路线编出的 `.ri` 功能等价，可随时互换重编。

### 首选：Zephyr SDK

Zephyr SDK 捆绑了 NXP i.MX8M 的 Xtensa 工具链（bundle 名里含 `xtensa-nxp_imx8m`）。到 <https://github.com/zephyrproject-rtos/sdk-ng/releases> 下载 `zephyr-sdk-*-linux-x86_64.tar.xz`，下载前先在 release 页面的 bundle 清单里确认它含 `xtensa-nxp_imx8m`，没有就退到 crosstool-NG。

```bash
cd ~/sof-dev
tar xf ~/Downloads/zephyr-sdk-*-linux-x86_64.tar.xz   # 解出 zephyr-sdk-<版本>/
ln -sfn zephyr-sdk-<版本> zephyr-sdk                  # 固定软链接名，编译时的变量指向它
```

本机实测 SDK 1.0.1 的 `gnu/` 下含 `xtensa-nxp_imx8m_adsp_zephyr-elf`，可用。但 SDK 1.0.1 配这个 BSP 对齐的 Zephyr 3.7.99 需要三个兼容补丁：Zephyr 请求 `find_package(Zephyr-sdk 0.16)`，而 SDK 1.0.1 的版本声明文件**故意**拒绝 <1.0 的请求（注释自述 "SDK 1.0 is not backward compatible"），并且把 `generic.cmake`/`target.cmake` 从 `cmake/zephyr/` 挪进了 `cmake/zephyr/gnu/`。三步修复（原文件先备份）：

```bash
cd ~/sof-dev/zephyr-sdk

# ① 解压 hosttools（dtc 等）+ 注册 cmake 包（需要 PATH 里有 cmake）
export PATH=$HOME/.local/bin:$PATH
./setup.sh -h -c

# ② 放开版本兼容限制（只改这一处，比整文件重写保守）
cp cmake/Zephyr-sdkConfigVersion.cmake cmake/Zephyr-sdkConfigVersion.cmake.orig
sed -i 's/set(ZEPHYR_SDK_MINIMUM_COMPATIBLE_VERSION 1.0)/set(ZEPHYR_SDK_MINIMUM_COMPATIBLE_VERSION 0.0)/' \
    cmake/Zephyr-sdkConfigVersion.cmake

# ③ 补两个旧路径 shim（内容就是 include 到新位置）
printf 'include(${CMAKE_CURRENT_LIST_DIR}/gnu/generic.cmake)\n' > cmake/zephyr/generic.cmake
printf 'include(${CMAKE_CURRENT_LIST_DIR}/gnu/target.cmake)\n'   > cmake/zephyr/target.cmake
```

补丁后 CMake 打印 `Found host-tools: zephyr 1.0.1` 即生效。

**验证**：`ls ~/sof-dev/zephyr-sdk/gnu/xtensa-nxp_imx8m_adsp_zephyr-elf/bin/*-gcc` 有输出。

### 备选：crosstool-NG 自建 GCC

SOF 官方 build guide 的开源方案，约 1 小时构建（需要先有 sof 仓库的 clone）：

```bash
cd ~/sof-dev
git clone https://github.com/thesofproject/xtensa-overlay
git clone https://github.com/thesofproject/crosstool-ng
git -C xtensa-overlay checkout sof-gcc10.2
git -C crosstool-ng    checkout sof-gcc10x

cd crosstool-ng
./bootstrap && ./configure --prefix=$(pwd) && make

# 用 imx8mp-audiomix 配置建工具链（配置文件在 sof 仓库 scripts/xtensa/ 下，以你 checkout 版本的实际文件为准）
cp ../sof/scripts/xtensa/imx8mp-audiomix/* .
./ct-ng menuconfig     # 可微调 CT_INSTALL_PREFIX
./ct-ng build          # 约 1 小时
```

产物是 `xtensa-imx8mp-audiomix-elf-gcc` 前缀的交叉工具链。

> 关于 XCC：BSP 里 `sof-zephyr-xcc` 目录下的固件就是 NXP 用 XCC 编好的产物——想要 XCC 版直接用 NXP 编好的即可，个人开发者不用自己去拿 XCC 授权。

## topology 工具链：源码编 alsatplg

编 `.tplg` 用 `alsatplg`（alsa-utils 里的拓扑编译器）。SOF 这个分支的 `tools/topology/CMakeLists.txt` **硬检查 `alsatplg ≥ 1.2.5`**（低版本会静默吞掉非法输入、编出坏 `.tplg`），而 Ubuntu 20.04 apt 的 alsa 只有 1.2.2。所以从源码编一份装到独立前缀 `~/sof-dev/alsa-local`，不污染系统：

```bash
cd ~/sof-dev
V=1.2.11   # 实测可用版本
# alsa-lib
curl -LO https://www.alsa-project.org/files/pub/lib/alsa-lib-$V.tar.bz2
tar xf alsa-lib-$V.tar.bz2 && cd alsa-lib-$V
./configure --prefix=$HOME/sof-dev/alsa-local --disable-python --disable-alisp
make -j$(nproc) && make install
cd ..
# alsa-utils（提供 alsatplg）
curl -LO https://www.alsa-project.org/files/pub/utils/alsa-utils-$V.tar.bz2
tar xf alsa-utils-$V.tar.bz2 && cd alsa-utils-$V
PKG_CONFIG_PATH=$HOME/sof-dev/alsa-local/lib/pkgconfig \
  ./configure --prefix=$HOME/sof-dev/alsa-local \
              --disable-alsamixer --disable-alsaconf --disable-bat --disable-nls
make -j$(nproc) && make install
cd ..
```

**验证**：`~/sof-dev/alsa-local/bin/alsatplg --version` 打印 1.2.11（≥1.2.5 即可）。

因为它装在非标准前缀，编 topology 时有**两处**要把它挂上，缺一就报 `alsatplg: not found`：cmake 配置阶段用 `-DCMAKE_PREFIX_PATH` 让 CMake 找到它，ninja 构建阶段把它的 `bin` 放进 `PATH`、`lib` 放进 `LD_LIBRARY_PATH`（运行时要加载自己的 libatopology）。具体命令在下面编 topology 那节。

## 环境就绪自检表

编固件/编 topology 前，逐条过一遍，全绿再往下：

```bash
export PATH=$HOME/sof-dev/python311/bin:$HOME/.local/bin:$PATH

# 主机工具
~/sof-dev/python311/bin/python3 --version              # 3.11.x
cmake --version | head -1                              # ≥3.20（路径应是 ~/.local/bin/cmake）
ninja --version                                        # 有输出

# west + 工作区
west --version                                         # 有版本号
git -C ~/sof-dev/sof rev-parse --abbrev-ref HEAD       # imx-stable-v2.11
ls ~/sof-dev/zephyr >/dev/null && echo zephyr-ok

# DSP 工具链（走 Zephyr SDK）
ls ~/sof-dev/zephyr-sdk/gnu/xtensa-nxp_imx8m_adsp_zephyr-elf/bin/*-gcc

# topology 工具链
~/sof-dev/alsa-local/bin/alsatplg --version            # ≥1.2.5
```

---

环境齐了，下面开始编。

## 编 firmware（.ri）

SOF 主仓库对 i.MX8MP 的 Zephyr 构建入口是 `scripts/xtensa-build-zephyr.py`：

```bash
cd ~/sof-dev/sof
# 三个环境变量缺一不可。PATH 顺序：python311 的 west 优先，让 Zephyr 拿到 3.11；
# ~/.local/bin 提供 pip 装的 cmake/ninja
export PATH=$HOME/sof-dev/python311/bin:$HOME/.local/bin:$PATH
export ZEPHYR_SDK_INSTALL_DIR=$HOME/sof-dev/zephyr-sdk
export ZEPHYR_TOOLCHAIN_VARIANT=zephyr

# 走 crosstool-NG 的话改成这两个（与上面互斥，别同时设，否则脚本找错工具链）：
#   export XTENSA_TOOLCHAIN_PATH=$HOME/sof-dev/x-tools
#   export ZEPHYR_TOOLCHAIN_VARIANT=xtensa

# 平台名是 imx8m，不是 imx8mp
./scripts/xtensa-build-zephyr.py imx8m
```

有两个地方特别容易踩：

- **平台名是 `imx8m` 不是 `imx8mp`**。脚本的 `platform_configs` 里注册名就是 `imx8m`（对应 Zephyr board `imx8mp_evk`），传 `imx8mp` 会报 `Unsupported platform`。
- **产物固件名也是 `sof-imx8m.ri`**。i.MX8MP 的内核驱动 `sound/soc/sof/imx/imx8m.c` 请求的就是 `sof-imx8m.ri`，NXP 官方 tarball 里也只有这个名字，没有 `sof-imx8mp.ri`。

产物在 `build-imx8m/zephyr/zephyr.ri`（本机实测约 275KB，314/314 全过、`BUILD_EXIT_CODE=0`），并会自动按 `sof-<平台名>.ri` 规则暂存一份到 `build-sof-staging/sof/imx/sof/community/sof-imx8m.ri`，名字天然就是内核要的，无需手工改名。

rimage 打印的 `error: 'alias_mask' not found` 是解析 `imx8m.toml` 的非致命提示，`.ri` 会正常写出，忽略即可。

写完自研模块后（第三篇），同一条命令重编就产出含你算法的固件。

## 编 topology（.tplg）

有个常见误解要先澄清：`sof-imx8mp-wm8962.tplg` **能从 SOF 源码直接编出**，既不需要"手抄 imx93 模板"，也不是"NXP 未开源"。它由 `tools/topology/topology1/CMakeLists.txt` 里一条生成规则产出，用通用模板 `sof-imx8-wm8960` 带参数生成：

```
# tools/topology/topology1/CMakeLists.txt 里的这一行
"sof-imx8-wm8960\;sof-imx8mp-wm8962\;-DCODEC=wm8962\;-DRATE=48000\;-DPPROC=volume\;-DSAI_INDEX=3\;-DDMA_DOMAIN"
#   模板            产物名(→.tplg)      codec        采样率        处理组件      SAI3     DMA域
```

也就是说 pipeline 是 `PCM0 <—— volume ——> SAI3 (WM8962)`，产物名天然就是 dts 里 `tplg-name` 指向的 `sof-imx8mp-wm8962.tplg`。

编译时的关键，就是前面说的 alsatplg 要在**配置和构建两处**都挂上：

```bash
# ① 一次性配置（若 build-topo 还没建）。CMAKE_PREFIX_PATH 让 CMake 找到 alsatplg
export PATH=$HOME/.local/bin:$PATH
cd ~/sof-dev
cmake -G Ninja -B build-topo -S sof/tools \
      -DCMAKE_PREFIX_PATH=$HOME/sof-dev/alsa-local

# ② 每次编译：alsatplg 进 PATH、它的 libatopology 进 LD_LIBRARY_PATH
export PATH=$HOME/sof-dev/alsa-local/bin:$HOME/.local/bin:$PATH
export LD_LIBRARY_PATH=$HOME/sof-dev/alsa-local/lib:$LD_LIBRARY_PATH
ninja -C ~/sof-dev/build-topo topology/topology1/production/sof-imx8mp-wm8962.tplg

# 产物（本机实测 7587 字节）：
#   ~/sof-dev/build-topo/topology/topology1/production/sof-imx8mp-wm8962.tplg
```

几个要点：

- **topology 迭代不需要重编 `.ri`**。调系数、换组件组合，只重发 `.tplg` 即可，这是后期开发最省时间的一点。
- **ABI 要同代**。`.tplg`、`.ri`、内核驱动三者的 IPC ABI 必须一致，全程待在 2.11 分支内就自洽，别混不同 SOF 版本。
- 想把自研算法挂进 pipeline，就复制 CMakeLists 那一行、把 `-DPPROC=volume` 换成你的组件，或改 `sof-imx8-wm8960.m4` 模板——这在第三篇细说。

BSP tarball 里 NXP 预编译的 `.tplg` 与你自编的字节可能不同（构建选项不同），但同为 2.11 ABI、功能等价，两者都能上板。

## 部署到板上

注意 NXP BSP 的固件路径与上游文档不同：NXP 内核从 **`/lib/firmware/imx/sof/`** 取固件（BSP 的 symlink 结构就是为此设计的），上游文档写的 `/lib/firmware/intel/sof/` 在这套 BSP 上无效。

```bash
scp build-imx8m/zephyr/zephyr.ri root@192.168.0.106:/lib/firmware/imx/sof/sof-imx8m.ri
scp build-topo/topology/topology1/production/sof-imx8mp-wm8962.tplg root@192.168.0.106:/lib/firmware/imx/sof-tplg/

# 板上重启驱动（模块名以 lsmod | grep sof 实际为准），或直接 reboot
rmmod snd_sof_imx8m snd_sof_of snd_sof_xtensa_dsp 2>/dev/null
modprobe snd_sof_of && modprobe snd_sof_imx8m
```

然后跑第一篇的验证命令（`dmesg | grep sof`、`aplay`、`arecord`），确认用**自己编的**固件和 topology 跑通同样的效果。

## 附：本篇相关的常见问题

- **编 topology 报 `alsatplg: not found`**：alsatplg 装在非标准前缀，cmake 配置阶段要 `-DCMAKE_PREFIX_PATH`，ninja 阶段要把 `alsa-local/bin` 加进 `PATH`、`alsa-local/lib` 加进 `LD_LIBRARY_PATH`，两处都挂才行。
- **编 topology 报版本过低 / 静默产出坏 tplg**：apt 的 alsatplg 是 1.2.2，CMakeLists 要求 ≥1.2.5，按上文源码编 1.2.11。
- **`xtensa-build-zephyr.py` 报 `Unsupported platform`**：平台名传成了 `imx8mp`，应为 `imx8m`。
- **`Topology ABI mismatch`**：`.tplg` 与 `.ri` 不同版本，用 `sof-logger`/dmesg 看 ABI 号，两端一起换。
- **`west update` 超时**：见上文 west 一节的代理/镜像两个办法，修好后回 `~/sof-dev` 重跑，不要重新 init。

---

固件能自己编了，接下来就是把你自己的算法塞进去。见第三篇：[实战：把自研算法写进 DSP 固件](frdm-imx8mp-sof-part3-custom-dsp-algorithm.md)。

# 实战：写一个实时降噪模块，消掉录音底噪

> 系列第三篇。前提是第二篇 [从源码编译固件与 topology](frdm-imx8mp-sof-part2-build-from-source.md) 的自检表全绿，能自己编出 `.ri` 和 `.tplg`。

这一篇的目标很具体：你用 `arecord` 录音，现在录出来的文件带着一层底噪（模拟前端和 ADC 的宽带嘶声）。我们要在 DSP 上写一个**实时降噪模块**挂进录音链路，让 `arecord` 直接拿到人声清晰、底噪干净的文件。同时解决两件工程上的事——**怎么开关它、怎么看它的 Log**。

先明确一点：DSP 降噪是锦上添花，不是用来补救接线问题的。开工前请确认第一篇 [上板篇的 WM8962 mixer 一节](frdm-imx8mp-sof-part1-bring-up.md#让录音真正出声wm8962-的-mixer-设置) 已经做好——关掉悬空输入、开高通、开 ADC 高性能模式。模拟侧每压低 1dB 底噪，都比在 DSP 上硬降要干净。DSP 降噪处理的是模拟侧压不掉的那层残余宽带噪声。

## 算法该挂在录音链的哪一段

`arecord` 是采集（capture），数据流是：

```
麦克风 → WM8962(ADC) → SAI3 → DSP → PCM0(capture) → arecord
                                 ↑
                          降噪模块挂这里
```

所以降噪模块必须挂在 **capture pipeline** 上，位于 DSP 收到 SAI3 送来的采集数据之后、送进 PCM0 给 arecord 之前。这跟播放（playback）链是两条独立的 pipeline，别挂错——挂到 playback 上只会处理放音，对录音文件毫无作用。

topology 里控制这个的就是第二篇提到的 `-DPPROC` 那条生成规则里的处理组件，以及 `sof-imx8-wm8960.m4` 模板中 capture pipeline 那一节，后面「挂进 capture pipeline」一节会具体改。

## 选哪种降噪算法

底噪的特征是**宽带、平稳**的低电平嘶声，人声则是在它之上起伏的高电平信号。针对这个特征，实时可行且效果直接的做法有两档：

- **入门档：自适应噪声门 + 下行扩展器（本篇实现的）。** 持续估计当前的噪声电平（floor），把低于阈值的信号按比例往下压、高于阈值的人声原样放过。它不需要 FFT，纯时域、定点、无动态内存，稳稳落在 HiFi4 的实时预算里。对"人声之间的间隙嘶声"和"整体本底"压制效果立竿见影，是消底噪最划算的第一步。
- **进阶档：谱减法 / Wiener 滤波。** 要在**人声持续期间**也把叠在语音上的嘶声抠掉，就得进频域：FFT → 每个频点估噪声谱 → 减去 → IFFT。效果更细腻但代价大（FFT 开销 + 帧延迟），需要用 HiFi4 的 DSP 库，且要仔细控制"音乐噪声"伪影。本篇先把入门档跑通，进阶档在最后给方向。

一句话：**先用扩展器把底噪压干净**，这对绝大多数"录音有嘶声"的场景已经够用；追求极致再上谱减。

## 模块骨架

以你 checkout 分支里的 `src/audio/volume.c` 为模板改（不同 SOF 版本 module API 签名有差异，照抄旧示例会编不过）。下面是降噪模块的结构，读写循环和格式/通道/环形缓冲的处理照抄 `volume.c` 的 `volume_s32_process()`：

```c
// src/audio/noise_supp.c —— 实时降噪：自适应本底估计 + 下行扩展器
#include <sof/audio/module_adapter/module/module/base.h>
#include <sof/audio/buffer.h>
#include <sof/audio/component_types.h>
#include <sof/trace/trace.h>
#include <sof/math/numbers.h>
#include <module/module/base.h>
#include <module/trace/trace.h>
#include <stdint.h>

/* trace 上下文，照抄 volume.c 的写法最稳 */
DECLARE_SOF_RT_UUID("noise_supp", noise_supp_uuid, 0xa1, 0xb2, /* 后续字节自定义 */ ...);
DECLARE_TR_CTX(nr_tr, SOF_UUID(noise_supp_uuid), LOG_LEVEL_INFO);

/* 运行时可调参数，通过 set_config 从 topology / sof-ctl 下发（见「开关与调参」） */
struct noise_supp_config {
    int32_t enable;        /* 0=旁路（原样输出），1=开启 */
    int32_t threshold_q31; /* 门限：包络低于它就判为噪声，按比例压制。默认约 -45 dBFS */
    int32_t ratio_q16;     /* 扩展比：低于门限时的衰减斜率，Q16。越大压得越狠 */
    int32_t attack_q31;    /* 包络跟踪的攻击/释放平滑系数（一阶 IIR） */
    int32_t release_q31;
};

/* 每通道的运行状态 */
struct noise_supp_chan {
    int32_t env_q31;   /* 当前包络（跟踪信号电平） */
    int32_t floor_q31; /* 自适应噪声本底估计 */
    int32_t gain_q31;  /* 上一帧应用的增益，用于平滑，避免抽气声 */
};

struct noise_supp_data {
    struct noise_supp_config cfg;
    struct noise_supp_chan   ch[PLATFORM_MAX_CHANNELS];
    uint32_t log_div;  /* Log 分频计数，见「查看 Log」，防止刷屏拖垮实时 */
};
```

`init` / `free` 负责私有数据的分配与释放，跟 volume.c 一模一样：

```c
static int noise_supp_init(struct processing_module *mod)
{
    struct module_data *md = module_get_private_data(mod);
    struct noise_supp_data *d =
        rzalloc(SOF_MEM_ZONE_RUNTIME, 0, SOF_MEM_CAPS_RAM, sizeof(*d));
    if (!d)
        return -ENOMEM;

    /* 默认值：开启、门限约 -45dBFS、扩展比 3:1、快攻击慢释放 */
    d->cfg.enable        = 1;
    d->cfg.threshold_q31 = 0x0016C310;   /* ≈ 10^(-45/20) 的 Q31 */
    d->cfg.ratio_q16     = 3 << 16;
    d->cfg.attack_q31    = 0x40000000;   /* 平滑系数，可后续调 */
    d->cfg.release_q31   = 0x08000000;

    md->private = d;
    tr_info(&nr_tr, "noise_supp init: enable=%d thr=0x%08x",
            d->cfg.enable, d->cfg.threshold_q31);
    return 0;
}

static void noise_supp_free(struct processing_module *mod)
{
    rfree(module_get_private_data(mod)->private);
}
```

核心是 `process`。**这里是硬实时区，绝对不能有 malloc / printf / 除法长循环**（Log 用下面的分频节流方式）：

```c
static int noise_supp_process(struct processing_module *mod,
                              struct input_stream_buffer *input_buffers, int num_in,
                              struct output_stream_buffer *output_buffers, int num_out)
{
    struct noise_supp_data *d = module_get_private_data(mod)->private;

    /* 旁路：enable=0 时原样拷贝（拷贝逻辑照抄 volume.c 的 passthrough 分支） */
    if (!d->cfg.enable) {
        /* copy input -> output，处理好帧数/通道/环形缓冲 wrap */
        return 0;
    }

    /* 遍历每帧、每通道（frames/channels 的取法照抄 volume.c）：
       对每个 s32 样本 x（Q1.31）：

       1) 整流取瞬时幅度： a = |x|
       2) 包络跟踪（一阶 IIR，攻击/释放不同系数）：
          env = (a > env) ? env + ((a-env)*attack>>31)
                          : env + ((a-env)*release>>31)
       3) 噪声本底估计：只在“判为噪声”时缓慢跟随，人声期间冻结，
          floor 朝 env 缓慢逼近（release 更慢），得到平稳的本底水平
       4) 计算增益：
          - env >= threshold（有人声）→ gain 朝 Q1.31 的 1.0 平滑回升
          - env <  threshold（是噪声）→ 按 ratio 下压：
              超出本底的量越小，增益越接近 0，把嘶声压向静音
       5) gain 做一阶平滑（用 ch.gain_q31）避免增益突变产生“抽气/呼吸”声
       6) y = q31_mult(x, gain)，写回输出

       所有乘法用定点 helper（如 sat_int32、q_multsr_sat_32x32），
       不要用浮点、不要在循环里除法 */

    return 0;
}
```

`set_config`（收 topology/sof-ctl 下发的参数）、`reset`（清 env/floor/gain 状态）、`prepare`，以及最后的 module_interface 注册，结构都照 volume.c：

```c
static int noise_supp_set_config(struct processing_module *mod, uint32_t config_id,
                                 enum module_cfg_fragment_position pos, size_t data_offset,
                                 const uint8_t *fragment, size_t fragment_size)
{
    struct noise_supp_data *d = module_get_private_data(mod)->private;

    /* 把下发的字节拷进 cfg。真实实现要按 SOF 的 config blob 格式解析
       （参照 eq_iir.c 的 set_config——它是“带参数下发”的最佳范本）。
       这里示意最关键的 enable 开关和门限： */
    if (fragment_size >= sizeof(struct noise_supp_config)) {
        const struct noise_supp_config *nc = (const void *)fragment;
        d->cfg = *nc;
        tr_info(&nr_tr, "set_config: enable=%d thr=0x%08x ratio=0x%08x",
                d->cfg.enable, d->cfg.threshold_q31, d->cfg.ratio_q16);
    }
    return 0;
}

static int noise_supp_reset(struct processing_module *mod)
{
    struct noise_supp_data *d = module_get_private_data(mod)->private;
    for (int i = 0; i < PLATFORM_MAX_CHANNELS; i++) {
        d->ch[i].env_q31   = 0;
        d->ch[i].floor_q31 = 0;
        d->ch[i].gain_q31  = ONE_Q31;   /* 复位为直通增益 1.0 */
    }
    return 0;
}

struct module_interface noise_supp_interface = {
    .init       = noise_supp_init,
    .prepare    = NULL,               /* 无预处理时可省 */
    .process    = noise_supp_process,
    .set_config = noise_supp_set_config,
    .reset      = noise_supp_reset,
    .free       = noise_supp_free,
};
DECLARE_MODULE(noise_supp_interface);
```

## 注册模块

把模块编进固件要四样东西，我照 `src/audio/dcblock/` 这个内建组件做了一套，落在 `src/audio/noise_supp/` 目录：

1. **UUID**：在仓库根的 `uuid-registry.txt` 里加一行 `a5f3c1d2-9e4b-4a76-b1c208d4e6f7a9b0 noise_supp`。构建时 `scripts/cmake/uuid-registry.cmake` 会把它生成到 `uuid-registry.h`；代码里用 `SOF_DEFINE_REG_UUID(noise_supp)` 声明，名字必须和这里对上。
2. **Kconfig**：`src/audio/noise_supp/Kconfig` 写一个 `config COMP_NOISE_SUPP`（`tristate`、`default y`），再在 `src/audio/Kconfig` 里 `rsource "noise_supp/Kconfig"`。
3. **构建接线**：模块目录自带一个 `CMakeLists.txt`（`add_local_sources(sof noise_supp.c noise_supp_generic.c)` + 按 IPC 版本选 `noise_supp_ipc3.c`/`ipc4.c`）。
4. **Topology**：在 capture pipeline 里用它（下一节）。

> **这里有个坑，卡了我一轮。** 光在 `src/audio/CMakeLists.txt` 里加 `add_subdirectory(noise_supp)` 是**不够的**——那条路径只在 llext（动态库）构建里走。imx8m 走的是内建构建，Zephyr 侧的组件源码是在 **`sof/zephyr/CMakeLists.txt`** 里用一串显式的 `zephyr_library_sources()` 列出来的（dcblock 那段在 650 行附近）。第一次编译我漏了这个文件，结果 `CONFIG_COMP_NOISE_SUPP=y`、`.config` 和 `autoconf.h` 里都有、cmake 也没报错，但 `build.ninja` 里一条 `noise_supp.c.obj` 的编译规则都没有，模块根本没进固件。加上这一段才对：
>
> ```cmake
> # sof/zephyr/CMakeLists.txt，紧跟 dcblock 那个 if 块之后
> if(CONFIG_COMP_NOISE_SUPP)
>     zephyr_library_sources(
>         ${SOF_AUDIO_PATH}/noise_supp/noise_supp_generic.c
>         ${SOF_AUDIO_PATH}/noise_supp/noise_supp.c
>         ${SOF_AUDIO_PATH}/noise_supp/noise_supp_${ipc_suffix}.c
>     )
> endif()
> ```
>
> `${ipc_suffix}` 由构建按 IPC 版本解析成 `ipc3`/`ipc4`，正好对应我的文件名。验证有没有真编进去：`grep -c noise_supp build-imx8m/zephyr/zephyr.map`，为 0 就是没进；我修好后是 154 个符号，`.ri` 也从 275KB 涨到 279KB。

## 挂进 capture pipeline

关键是挂对链路——必须是 capture，不是 playback。我确认过库存模板 `sof-imx8-wm8960.m4`：playback（第 40 行）用的是 `pipe-PPROC-playback.m4`，capture（第 47 行）**写死**了 `pipe-volume-capture.m4`。也就是说 `-DPPROC` 只影响放音，光换 PPROC 参数对录音毫无作用。所以**不能靠改 PPROC，得单独给 capture 链换 pipe 文件**。

我的做法是新起一套 topology 变体，不动库存文件：

**① m4 组件宏** `tools/topology/topology1/m4/noise_supp.m4`：照 `dcblock.m4` 抄，`W_NOISE_SUPP` / `N_NOISE_SUPP` 两个宏。UUID 用 `DECLARE_SOF_RT_UUID` 声明，字节序要和固件里的 `SOF_DEFINE_REG_UUID(noise_supp)` 对上。

> **坑二：进程类型（process type）别自创。** widget 里有个 `SOF_TKN_PROCESS_TYPE`。如果填一个新字符串（比如 `"NOISE_SUPP"`），**内核侧** SOF 拓扑解析器的 `process_map[]` 不认，`.tplg` 加载就会失败，逼你连内核一起重编重刷。规避办法：这里**复用内核已认识的 `"DCBLOCK"`**，但 widget 的 UUID 填自己的——IPC3 里固件是按 **UUID** 找模块的，内核根本不需要知道 `noise_supp` 这个名字。这样只动固件和 topology，内核不用碰。

**② 默认参数 blob** `tools/topology/topology1/m4/noise_supp_default.m4`：32 字节 SOF ABI 头 + 7 个小端 int32 参数（enable / thr_open / thr_close / 四个 shift），字节序和 `enum noise_supp_param` 一致。

**③ capture pipe** `tools/topology/topology1/sof/pipe-noise-supp-capture.m4`：照 `pipe-dcblock-capture.m4` 抄，把 `W_DCBLOCK` 换成 `W_NOISE_SUPP`，数据流是 `host PCM_C ← Noise Supp ← sink DAI`。

**④ 顶层 topology** `tools/topology/topology1/sof-imx8-nr-wm8960.m4`：拷 `sof-imx8-wm8960.m4`，只把 capture 那行改成走我的 pipe：

```m4
PIPELINE_PCM_ADD(sof/pipe-noise-supp-capture.m4,
	2, 0, 2, s32le,
	1000, 0, 0,
	`RATE', `RATE', `RATE')
```

**⑤ 注册产物** 在 `tools/topology/topology1/CMakeLists.txt` 的 wm8962 那行后面加一行，产物名 `sof-imx8mp-nr-wm8962`：

```
"sof-imx8-nr-wm8960\;sof-imx8mp-nr-wm8962\;-DCODEC=wm8962\;-DRATE=48000\;-DPPROC=volume\;-DSAI_INDEX=3\;-DDMA_DOMAIN"
```

> **坑三：m4 里多一个引号，报错报在文件末尾。** 我第一版 `noise_supp.m4` 的 `SOF_TKN_PROCESS_TYPE` 那行末尾多写了一个反引号配对的 `'`，渲染出来变成 `SOF_TKN_PROCESS_TYPE	"DCBLOCK"'`（尾巴多个单引号）。`m4` 不报错、大括号也全平衡，但 `alsatplg` 报 `Unexpected char` 且行号指到**文件末尾+1 行**（EOF），极具误导性。定位办法：`cat -A` 看渲染出的 conf，逐段核对；或写个脚本忽略字符串/注释后统计 `{}`、`[]` 深度。删掉那个多余的 `'` 就过了，`.tplg` 正常产出 7.5KB。

验证阶段产物用 `sof-imx8mp-nr-wm8962.tplg`，在板上另存到 `/lib/firmware/imx/sof-tplg/`，dts 的 `tplg-name` 临时指过去；确认无误再决定是否覆盖正式的 `sof-imx8mp-wm8962.tplg`。**改拓扑不需要重编 `.ri`**。

## 怎么开关它

你要的开关有三个层次，从"运行中即时切"到"编译期决定"：

### 1. 运行时开关（推荐，无需重启驱动）

在 topology 里给降噪模块挂一个 **bytes/enum kcontrol**，它就会在 `arecord` 用的那张声卡上暴露成一个 amixer 控件，`set_config` 会收到你写入的值。这样开关降噪只是一条 amixer 命令，`arecord` 不用停：

```bash
# 先看看控件叫什么（名字由 topology 里的 kcontrol 名决定）
amixer -c 0 scontrols | grep -i noise

# 开 / 关（若做成 enum/switch 控件）
amixer -c 0 sset 'Noise Suppression' on
amixer -c 0 sset 'Noise Suppression' off
```

如果做成 bytes control（把整个 `noise_supp_config` 下发，可同时调门限/比例），就用 `sof-ctl`（sof-tools 包自带）：

```bash
# 把一组参数（enable/threshold/ratio...）打成 blob 下发给模块
sof-ctl -Dhw:0 -n 'Noise Suppression' -s my_nr_params.bin
```

无论哪种，最终都会走到模块的 `set_config`，改的是 `d->cfg`。`process` 每帧读 `d->cfg.enable`：为 0 就走旁路原样输出，为 1 就降噪。**这就是为什么骨架里 enable 是运行时字段而不是编译宏**——能边录边听、边调门限，迭代最快。

### 2. topology 默认值

topology 里给 kcontrol 设初值，决定**开机默认**是否降噪、默认门限多少。不动 amixer 时就是这套值。

### 3. 编译期彻底移除

真要把模块从固件里拿掉（省 DSP 空间或排查问题），关掉 defconfig 里的 `CONFIG_COMP_NOISE_SUPP` 重编 `.ri`。日常开关不用走这步，用运行时开关就够。

## 怎么看它的 Log

DSP 端的 `tr_info()` / `tr_err()` 输出不走内核 dmesg，要用 SOF 自己的 trace 通道，板上 `sof-logger`（sof-tools 包）读取：

```bash
# 板上实时看 DSP 端日志（模块里 tr_info 打的都在这）
sof-logger -l /sys/kernel/debug/sof/trace

# 或落盘再看
sof-logger -l /sys/kernel/debug/sof/trace -o /tmp/dsp.log &
```

要让 `tr_info` 真的出来，firmware 编译时得开日志。在降噪模块对应的 defconfig 里开 `CONFIG_TRACE`，并把 `CONFIG_SOF_LOG_LEVEL` 调到 info 级。这样 `init`、`set_config` 里那几条 `tr_info` 就能看到——尤其能确认 `set_config` 真收到了你的开关值。

内核侧（驱动加载、固件启动、ABI 是否匹配）仍在 dmesg：

```bash
dmesg | grep -i sof
# 可选：调高内核侧 SOF 调试等级（路径以实际为准）
echo 4 > /sys/module/snd_sof/parameters/sof_debug
```

**实时纪律**：`process` 是每帧都跑的硬实时函数，**不能**在里面无脑 `tr_info`，否则日志 I/O 会拖垮实时、直接 xrun。骨架里那个 `log_div` 就是干这个的——每 N 帧（比如每秒一次）才打一条，用来观测当前 env/floor/gain：

```c
/* process 末尾，节流打点，不是每帧都打 */
if (++d->log_div >= 48000 / frames) {   /* 约每秒一次 */
    d->log_div = 0;
    tr_info(&nr_tr, "ch0 env=0x%08x floor=0x%08x gain=0x%08x",
            d->ch[0].env_q31, d->ch[0].floor_q31, d->ch[0].gain_q31);
}
```

这条 Log 也是调门限的依据：录一段只有底噪的静音，看 `floor` 稳定在什么量级，把 `threshold` 设在它上方一点点，人声就能过、底噪被压住。

## 开发迭代循环

板子开着 SSH，改代码 → 听效果一轮大约两分钟：

```bash
# ① 改 src/audio/noise_supp.c，重编固件
./scripts/xtensa-build-zephyr.py imx8m

# ② 部署 + 重启驱动
scp build-imx8m/zephyr/zephyr.ri root@192.168.0.106:/lib/firmware/imx/sof/sof-imx8m.ri
ssh root@192.168.0.106 'rmmod snd_sof_imx8m; modprobe snd_sof_of; modprobe snd_sof_imx8m'

# ③ 录一段听效果
ssh root@192.168.0.106 'arecord -D hw:0,0 -f S32_LE -r 48000 -c 2 -d 5 /tmp/t.wav'
```

只调门限/开关不用重编固件——用运行时的 amixer/sof-ctl 边录边调（见「怎么开关它」）。只改拓扑结构时也不用重编 `.ri`，重发 `.tplg` 即可。

## 验证效果

分三步确认降噪真的生效，而不是错觉：

```bash
# 1) 关降噪，录一段静音 + 一段说话，作为对照
amixer -c 0 sset 'Noise Suppression' off
arecord -D plughw:0,0 -f S32_LE -r 48000 -c 2 -d 8 /tmp/off.wav

# 2) 开降噪，同样录一段
amixer -c 0 sset 'Noise Suppression' on
arecord -D plughw:0,0 -f S32_LE -r 48000 -c 2 -d 8 /tmp/on.wav

# 3) 对比：静音段的底噪应明显变小，人声应基本不变
aplay /tmp/off.wav
aplay /tmp/on.wav
```

判断标准：静音段（不说话时）的嘶声在 `on.wav` 里应该明显压低甚至接近全静，而人声段的清晰度和音量不该掉。如果人声也发闷、字头被吃掉，说明门限设高了或释放太快——把 `threshold` 调低、`release` 放慢。如果底噪还在，就是门限太低或扩展比不够——调高 `threshold`、加大 `ratio`。这些都能用运行时控件即时试，配合上面那条 env/floor Log 看数值。

拉进音频软件（Audacity 等）看波形/频谱更直观：静音段的本底应该整体下沉。

## 写代码前必须知道的实时约束

DSP 上是硬实时环境，`process` 每帧都在赶 deadline，下面这些会直接决定算法能不能跑：

| 约束 | 对降噪模块意味着什么 |
|---|---|
| 实时预算 | HiFi4 @ 800MHz，帧约 5.3ms @ 48k/256。`process` 里**不许 malloc / printf / 除法长循环**；包络跟踪、增益计算全用定点乘加 |
| 定点化 | 样本是 s32（Q1.31）。门限、增益、系数都用 Q 格式，乘法走 `q_multsr_sat_32x32` 之类的饱和 helper，别用浮点 |
| 内存 | 私有数据 `rzalloc(SOF_MEM_ZONE_RUNTIME, ...)`，一次性在 init 分配；`process` 内零分配 |
| 环形缓冲 | 输入输出 buffer 是环形，遍历样本要处理 wrap，照抄 volume.c 的 copy 逻辑，别自己数指针 |
| Log 节流 | `tr_info` 只在 init/set_config 或 `process` 里按 `log_div` 分频打，绝不每帧打 |
| ABI | topology / 固件 / 内核三者 IPC ABI 必须同代，全程待在 2.11 分支 |

## 想进一步：谱减法 / Wiener 滤波

扩展器压的是"低于门限"的部分，人声持续说话时叠在语音上的那层嘶声它压不掉。要连这层也处理，就得进频域：

- 每帧做 FFT（HiFi4 有优化的 FFT 库，别自己写），得到幅度谱。
- 在"判为噪声"的帧上估计并更新**噪声谱**（每个频点一个估计）。
- 对每个频点做谱减或算 Wiener 增益（`gain = SNR / (SNR + 1)`），乘回幅度谱。
- IFFT 回时域，用 overlap-add 拼帧。

代价是明显的 FFT 计算开销和至少一帧的处理延迟，还要防"音乐噪声"（谱减残留的忽有忽无的窄带伪影，通常靠增益下限/时域平滑压制）。这套的骨架仍是同一个 processing module，只是 `process` 内部换成 FFT 流水线、私有数据里多存 FFT 工作区和噪声谱。先把本篇的扩展器跑顺、把 pipeline/开关/Log 这套工程链打通，再往这一步走，会稳很多。

## 一个建议的推进节奏

| 阶段 | 内容 | 验收 |
|---|---|---|
| 1 | 第二篇：搭主机工具链、过自检表 | 自检表全绿 |
| 2 | 第一篇：上板跑通 SOF，调好 WM8962 mixer | 能播放录音，模拟侧底噪已压到最低 |
| 3 | 第二篇编译部分：自己编 `.ri` + `.tplg` 替换预编译件 | 用自编产物跑通同样测试 |
| 4 | 本篇：写 noise_supp 扩展器，挂进 capture pipeline，接上运行时开关和 Log | `arecord` 录出的静音段底噪明显下降、人声清晰 |
| 5 | （可选）升级谱减/Wiener，处理语音期间的残余噪声 | 人声持续期的嘶声也被压制、无明显伪影 |

先把模拟侧（第一篇 mixer）做干净，再上 DSP 扩展器——这个顺序能让你少走很多"在 DSP 上硬补接线问题"的弯路。

# AD104 SM speed-select 解锁：可复刻工程流程

面向 **Ada AD104（RTX 4070 Ti 级）单卡、单机** 的完整复刻手册。方法源自
[JustAResearcher/gpu-unlock-ada](https://github.com/JustAResearcher/gpu-unlock-ada)（GPL-2.0）
的 Linux 内核模块思路，本手册描述的是把它搬到 **Windows + UEFI preboot** 的做法：
在 NVIDIA 驱动加载之前，用一个自签的 UEFI 应用完成 firmware 事务并把
selector 覆盖（SS0/SS1）写进去，然后交还给原版 Windows 启动。

> 本文所有与本机绑定的值都用占位符表示（见「敏感信息处置」一节）。
> 复刻者必须用自己的捕获数据填满每一个占位符 —— 不要照抄别人的值。

---

## 性能对比证据（实测，先看结论）

单机单卡：NVIDIA GeForce RTX 4070 Ti（AD104，`10de:2782`），驱动 617.42，
VBIOS `95.04.3c.80.3e`，功耗上限 285 W，显存 12,282 MiB。**解锁前后 VBIOS、驱动、
功耗上限均未变化**，差异只来自运行期 selector 覆盖。

测量方法：cuBLAS `cublasGemmEx`，N = 4096（4096×4096×4096），每个进程 15 次采样取中位数，
每个状态跑 3 个独立进程，取三者中位数。全部数值同时做过 256 项逐元素精确比对。

| 累加模式（N=4096） | 未解锁 TFLOP/s | 中位数 | 解锁后 TFLOP/s | 中位数 | 变化 |
|---|---:|---:|---:|---:|---:|
| FP32 pedantic | 25.15 / 25.13 / 25.28 | 25.15 | 27.31 / 27.54 / 28.04 | 27.54 | +9.5%（†） |
| FP16 → FP32 累加 | 66.20 / 65.92 / 65.91 | **65.92** | 134.76 / 133.71 / 133.95 | **133.95** | **+103.2%** |
| BF16 → FP32 累加 | 64.81 / 66.72 / 65.40 | **65.40** | 134.20 / 133.95 / 134.27 | **134.20** | **+105.2%** |

单次 GEMM 墙钟：FP16→FP32 2.085 ms → 1.026 ms；BF16→FP32 2.101 ms → 1.024 ms。

（†）FP32 pedantic 走的是 FP32 乘法器，本来就不受 speed-select 影响；+9.5% 属于时钟/环境波动，
**不作为解锁收益**，列出来只是为了让读者知道噪声量级是 10% 以内。

旁证：未解锁状态下单独对比 **FP16→FP16 累加** 得 118.11 TFLOP/s，而同机 FP16→FP32 只有 67.45 TFLOP/s
—— 说明限制只卡在「FP32 累加」这条通路上。解锁后 FP32 累加达到 ~134 TFLOP/s，已经越过该参考值。

数值正确性（这是最关键的一条）：三种模式在解锁前后

* `full_256_exact = true`，`full_256_max_abs_error = 0`；
* 三种模式输出的 `output_sha256` 完全相同，且解锁前后同一个值 ——
  即吞吐翻倍的同时**逐元素结果没有任何变化**，不是降精度换速度。

## 从日志判断「真的解锁了」

一次成功的运行在 USB 日志里必然出现这四组行（行内数值因卡而异，格式固定）：

```text
AD104 preflight: status=0 graph=152 code_profiles=4 steps=5
AD104 driver detach return: step=1..5                     (每个 step 都 status=0)
AD102 payload result: ... marker=<non-zero> marker_ok=1 ...   (固件跑完并回 marker)
selector result: SS0=0x88888888 expected=0x88888888 matched=1
selector result: SS1=0x8 expected=0x8 matched=1
PCIe FLR returned: result=0
post-FLR ... SS0=88888888 SS1=00000008                    (覆盖在 FLR 之后仍在)
Transaction result=0 safe=1 retained=0 started=1 quiesced=1 flrs=1 DMA_live=0
TRANSACTION OK: hardware readbacks passed, DMA released ...
```

判定要点：`matched=1` 是写入后回读，`post-FLR` 那行是复位后再读，两者都为
`SS0=0x88888888 / SS1=0x8` 才算成立；`retained=0` 表示没有留下未验证的 DMA 映射。
（原始日志含真实 handle 地址，已从本仓库移除；此处只保留格式与语义。）

性能与日志两组证据是**互相独立**的两条线：日志证明事务做成功了，benchmark 证明它带来了收益，
两者同时成立才叫「解锁成功」。

---

## 0. 原理（一段话）

Ada 的 GSP/FWSEC 固件在初始化后会把 SM 的 speed-select 档位寄存器锁在某个值上，
于是 **FP16/BF16 张量指令配 FP32 累加** 被限速（大约是满速的一半）。
解锁 = 在固件跑完、驱动接手之前：

1. 读物理 VBIOS，解析 ROM 链，定位 FWSEC 镜像与其几何（image/imem/dmem/pkc/interface/mapper）；
2. 用 DMA 把 FWSEC 与 payload 灌进 falcon，让它执行并放开 **permission 位**
   （WPR、FEAT、WPR_CFG 等），拿到 boot marker；
3. 写 **SS0/SS1 selector**（`0x88888888` / `0x8`）并回读确认；
4. 触发一次 **PCIe FLR**，验证覆盖在 FLR 之后仍然存在、且 GSP 已被 halt；
5. 释放 GPU 所有权，交还原版 `bootmgfw.efi` 启动 Windows。

整个操作只碰运行期寄存器与固件可见状态，**不写 VBIOS flash**，断电即回原状。

---

## 1. 前置条件与边界

| 项 | 要求 |
|---|---|
| 硬件 | 你自己拥有的、单张 AD104 卡；已知可用的 profile 参考机（同 GPU 型号） |
| 固件 | **Secure Boot 关闭**（应用会返回 `EFI_SECURITY_VIOLATION`）；UEFI 模式，CSM 关闭 |
| 系统 | Windows x64；能进固件启动菜单 |
| U 盘 | FAT32，≥ 128 MB（资产里最大的 `gsp_ga10x.bin` 约 84 MB） |
| 心态 | 这是**一次性物理尝试**的工程流程：一次跑不通就断电重启，不要连续重试 |
| 法律/许可 | 上游为 GPL-2.0-only；不随附任何 NVIDIA 固件、ROM 或驱动二进制 |

**硬纪律**（上游 port 的设计意图，复刻时不要为了"能跑"而放宽）：

- 身份校验是多重 pin（PCI ID + subsystem + PCI 配置 256 字节 + 物理 ROM SHA256 + 冷态寄存器
  + 已加载驱动代码 SHA256 + device path），**任何一项不符都必须拒绝**，不能只放宽到 PCI ID；
- 失败时**主动 halt**（显示已释放，宁可停机也不把半改状态交给 Windows）；
- 未经验证的固件可见 DMA 映射**保留到断电**，不做"猜测性释放"；
- 只允许 USB 热插拔在既定所有权/路径规则内增删关系，GPU 与虚拟控制台的关键关系必须保持。

---

## 2. 工具清单

### 2.1 必需

| 工具 | 用途 | 备注 |
|---|---|---|
| **git** | 取上游源码 | 任意版本 |
| **GCC（MinGW-w64 或 Cygwin）**，含 `ar`、`nm` | 编译 freestanding x64 目标文件 + 静态库 | 实测 GCC 11.4（Cygwin）与 15.2（MinGW-w64）均可；**不在 PATH 时要显式传路径** |
| **MSVC Build Tools（VS 2022）**，含 `link.exe` | 链接 EFI 镜像（`/subsystem:efi_application`） | 由 `vswhere` 自动发现，或用 `-VcVars64` 指定 `vcvars64.bat` |
| **Python 3**（3.11+） | 资产提取、manifest、EFI 校验脚本 | 无需第三方包即可跑提取/校验脚本 |
| **PowerShell 5.1/7** | 构建与测试脚本 | 首次需 `Set-ExecutionPolicy -Scope Process Bypass -Force` |
| **U 盘格式化工具** | FAT32 | Windows 自带即可 |

### 2.2 验证阶段用（Windows 侧）

| 工具 | 用途 |
|---|---|
| **PyTorch（CUDA wheel）** | 提供 `cublas64_*.dll`；基准脚本用 ctypes 直接调 cuBLAS |
| **numpy** | 结果校验 |
| **nvidia-smi** | 观察 VBIOS/驱动/功耗/温度（NVML 偶发 `Failed to initialize NVML`，不影响基准） |

### 2.3 排错时可能用到

| 工具 | 用途 |
|---|---|
| `dumpbin /disasm`（MSVC） | 反汇编捕获到的已加载驱动代码段（核对 `Stop()` 之类回调的行为）；裸二进制要先包一个最小 COFF 才能反汇编 |
| `objdump`（LLVM/binutils） | 同上 |
| 十六进制编辑器 | 核对 ROM 链、FWSEC 描述符 |

### 2.4 不需要

- 不需要 WSL / Linux（资产生成在上游是 `make assets`，若你已有可跑的 Linux 环境更快；纯 Windows 也能用上游 Python 脚本重算）；
- 不需要刷 VBIOS 的工具（本方法不写 flash）；
- 不需要第三方的 UEFI shell。

---

## 3. 阶段一：只读捕获（不改任何东西）

1. 用仓库里的 **diagnostic / probe** 子工程（`windows/ad104-msi/diagnostic/`）构建只读探针：

   ```powershell
   cd <WORKDIR>\gpu-unlock-ada\windows\ad104-msi\diagnostic
   .\Build.ps1 -Gcc 'C:\cygwin64\bin\gcc.exe'
   ```

   探针会先跑 `tests/host_tests.c` 的数千条 mock 断言，再产出 `build/AD104-PROBE.EFI`。

2. 部署到 U 盘：

   ```
   <USB>:\EFI\BOOT\BOOTX64.EFI      <- AD104-PROBE.EFI
   ```

3. UEFI 启动该 U 盘（Secure Boot 关）。探针**只读**：不写寄存器、不改 boot 变量、不碰 ESP。
   结束后 U 盘上会多出：

   ```
   <USB>:\AD104-PROBE\CAPxxxx.JSON     捕获文件
   <USB>:\AD104-PROBE\<PROM_DUMP>.bin  物理 VBIOS 全量 dump（1 MiB）
   ```

4. 捕获里必须能拿到这些字段（缺一项就别往下走，说明探针在本机不完整）：

   - `config256`：PCI 配置空间前 256 字节（BAR、subsystem、revision、capability 链）；
   - `register_snapshot`：冷态寄存器（WPR/WPR_CFG/FEAT/PLM/SS0/SS1/FBPA/…）；
   - `physical_prom`：物理 PROM 的 size + SHA256；
   - `gops` / `display_devices` / `console_out_handle_path` / `conout_variable`：显示与虚拟控制台句柄；
   - `platform.gpu_agents`：每个已加载显示驱动镜像的 `image_handle`、`version`、`image_bytes`、
     `code_type`、`stop_rva`、`code.rva/length/loaded_sha256`、`relocations_in_code`、`file_path`；
   - `uefi_graph` 等价的 open-protocol 边表（`handle`/`agent`/`controller`/`protocol`/`attributes`/`count`）。

> **不要发布捕获文件**：里面有你的 ROM、你的机器 GUID、你的句柄布局。

---

## 4. 阶段二：从捕获推导 pin

### 4.1 先判断"固件几何能不能复用"

把捕获里的物理 PROM 交给上游的 vbios 解析逻辑（`vendor/vbios_610.c` 里那套 ROM 链解析
在主机侧有等价脚本），确认三件事：

1. ROM 链：首个 `PCIR`、扩展 `PCIR`、`NPDS` 段，得到 `base` / `ext` / `adjust`；
2. FWSEC 描述符：`image_off`、`image_size`、`imem_size`、`dmem_size`、`pkc_off`、
   `interface_off`、`ucode_id`、`engine_mask`、`sig_off`、`sig_bytes`、`mapper`；
3. **FWSEC 镜像的 SHA256**。

若第 3 项与上游 profile 里记录的 `GU_AD104_FWSEC_IMAGE_SHA256` 一致 → 固件偏移、资产表、
payload 全部可原样复用，你只需要重钉"身份类"参数（下面 4.2）。若不一致 → 必须按你的
capture 重新推一遍几何，并重新评估 payload 的 permission 位是否仍适用；这一步没有捷径。

### 4.2 身份类 pin（复刻时必须逐项替换）

| 概念 | 来源字段 | 落到代码 |
|---|---|---|
| GPU device/subsystem ID | `config256[0]`, `config256[11]` | `port.h`: `GU_BOARD_ID`, `GU_BOARD_SUBSYSTEM` |
| BAR0 基址 | `config256[4]` | `port.h`: `GU_BAR0_BASE`；`profile.c` 的 BAR 解码检查 |
| BAR1/BAR3/BAR5 编码 | `config256[5..9]` | `profile.c`: `c[5..9]` 期望值 |
| PROM SHA256 | `physical_prom.sha256` | `profile.c`: `GuAd104PromMatches()` |
| 冷态 SS0 / SS1 / PLM / LinkSta | `register_snapshot` | `port.h`: `GU_STOCK_PROFILE_SS0`；`profile.c` 冷值表；`vendor/unlock_610.c` 的 selector 前置判断 |
| Windows ESP 几何 | 捕获里的 `HARDDRIVE_DEVICE_PATH` | `boot_io.c` + `local/boot_profile.h`：`PartitionNumber/Start/Size/GUID` |
| `bootmgfw.efi` 大小 + SHA256 | 直接在你的 ESP 上算 | `boot_io.c`: `GuBootLoadWindows()` |
| USB 宿主 device-path 前缀 | 捕获里 USB host 句柄的 path（去掉末尾 USB 节点） | `msi_release.c`: `UsbPointerPath()` 的 `host[]` |

### 4.3 句柄图 pin

从捕获的边表里读出本机的：

- `pci`（GPU 句柄）、`gop`（显示句柄）、**`gop2`**（若有第二个显示子句柄）、
  `out_agent` / `gfx_agent` / `nv_agent` / `aux_agent`（4 个驱动镜像各自的 binding 句柄）；
- `console` / `input` / `error`（虚拟控制台三件套）；
- `usb_host` + `usb_agent`（USB host controller 及其 `UsbHc` 占用者）；
- `pointer_agent` / `key_agent`（指针/键盘 splitter 的 BY_DRIVER 占用者）。

这些值只进**测试夹具**和运行期发现逻辑，不要把实际句柄硬编码进产品路径 ——
运行期应像 `msi_release.c` 那样从 open-protocol 边表里**推导**出 Plan，只在预检里做校验。

### 4.4 生成测试夹具

用提取脚本把捕获翻译成 C 头文件与二进制：

```powershell
python tools/extract_private_capture.py <CAPTURE_JSON> --out build/capture_assets \
    --header-dir tests
```

产出：

```
build/capture_assets/physical_prom.bin          物理 ROM（校验用，不随源码发布）
build/capture_assets/driver-<id>.bin            每个已加载显示驱动镜像的 .text 片段
tests/captured_config.h                         PCI 配置 256 字节夹具
tests/cap4_graph.h                              open-protocol 边表夹具（本机 152 条）
tests/cap4_usb_paths.h                          USB 分支 device path 夹具
```

脚本里有一张 `CODE` 映射表：**已加载代码的 SHA256 → (agent id, 长度)**。
你的捕获换了卡就得更新这张表，否则提取会因哈希不符而拒绝（这是特性，不是障碍）。

### 4.5 准备 5 个固件资产

```
<WORKDIR>/ad104-assets/
  booter_load_ad102.bin                 57,672 B
  v67-ad102-610.43.03-four-plm.bin      64,000 B
  gsp_ga10x.bin                     84,277,400 B
  gsp_boot_ad102.bin                    36,864 B
  gsp_boot_desc_ad102.bin                   84 B
```

用你自己 ROM 里的版本 + 上游 pinned 的生成流程（`make assets`）产出，然后让
`profile.c` 里的名称/大小/哈希表指向它们。**名字带版本号意味着换驱动版本就要重算**：
资产是驱动版本相关的，不是卡相关的。

---

## 5. 阶段三：移植代码（改动面）

只有 5 类文件需要动，别扩散：

| 文件 | 改什么 |
|---|---|
| `port.h` | 板级 ID、BAR0 基址、冷态 SS0 常量 |
| `profile.c` | PROM 哈希、BAR 解码期望、冷态寄存器表、selector 前置条件、资产表 |
| `vendor/unlock_610.c` | 冷入场的 selector 初值判断（例如 SS1 允许 1 或 3）、错误路径里的 halt 语义 |
| `boot_io.c` | ESP 几何 + `bootmgfw.efi` 大小/哈希；U 盘资产目录；日志目录 |
| `msi_release.c` | 句柄图推导、第二个显示句柄与其辅助占用者的 detach 步、USB 增删规则、预检/收尾校验 |

移植 `msi_release.c` 时最容易踩的三个坑（本机实测都踩到了）：

1. **第二个显示子句柄**：GPU 句柄下可能有两个显示子句柄（`gop` 和 `gop2`），
   每个都要**单独** `DisconnectController`；一次调用只能释放一个（`Stop()` 只关自己那个 controller）。
2. **计数别重复**：遍历边表时"识别出子句柄"和"计一次子关系"是两件事，
   写成 `if(...)child++; else if(...)gop2=...; else return REFUSE; child++;` 会把同一条边数两次，预检永远失败。
3. **detach 步数要与 `GU_MSI_STEPS` 对齐**：把两次 aux detach 合并成 `steps>=3` 会让收尾校验
   在正确的图上误判；应分别绑到 step 3 / step 4，NV 那条绑到 `steps>=GU_MSI_STEPS`。

---

## 6. 阶段四：主机侧测试（不碰 GPU）

```powershell
Set-ExecutionPolicy -Scope Process Bypass -Force
.\Test-Experimental.ps1 -Gcc 'C:\cygwin64\bin\gcc.exe' \
  -Prom 'captures\<PROM_DUMP>.bin' \
  -Assets 'build\assets' \
  -DriverCode 'build\capture_assets' \
  -WindowsImage 'C:\Windows\Boot\EFI\bootmgfw.efi' \
  -IncludeCore
```

要绿的五套（缺一不可）：

| 套件 | 它证明什么 |
|---|---|
| admission cases | 资产被篡改、身份不符、冷态不符、所有权不符、重复尝试 → 全部**拒绝且零写入** |
| core + compat + UEFI port scenarios | 真资产的完整事务（PCI/MMIO/DMA/firmware 全模拟）：单次 selector 写、恰好一次 FLR、FLR 后满速回读 |
| app flow | 阶段推进与失败 halt 语义 |
| **msi_release** | 用**真捕获的 152 条边表**跑预检/释放/收尾：句柄图漂移、USB 增删、晚到指针、链路状态漂移 |
| boot I/O | 用**真实文件字节**验证 ESP 识别、`bootmgfw.efi` 大小/哈希、资产装载、日志创建 |

---

## 7. 阶段五：构建 EFI

```powershell
.\Build-Experimental.ps1 -Gcc 'C:\cygwin64\bin\gcc.exe' \
  -Python 'C:\software\Python 3_11\python.exe' \
  -VcVars64 'C:\software\Visual Studio\VC\Auxiliary\Build\vcvars64.bat'
```

流程 = GCC freestanding 编 `-ffreestanding -fno-builtin -fno-stack-protector -mno-red-zone -mno-stack-arg-probe -fshort-wchar`
→ `ar` 打包 → `link /subsystem:efi_application /entry:UefiMain /machine:x64 /nodefaultlib`
→ `tools/verify_experimental.py` 校验（x64 EFI 入口、重定位、无 import/TLS/CLR、逐函数栈帧）。

必须先编过的一条纪律：**能编过不等于能用** —— `verify_experimental.py` 报 `hardware_tested: false` 是正常的，
真实硬件验证是阶段七的事。

---

## 8. 阶段六：部署到 U 盘

```
<USB>:\EFI\BOOT\BOOTX64.EFI          解锁应用（本次 93,696 B）
<USB>:\AD104-ASSETS\*.bin            5 个资产，程序会逐个校验 **大小 + SHA256**
<USB>:\AD104-UNLOCK\                 运行日志目录（程序自建，写 RUNxxxx.TXT）
<USB>:\AD104-PROBE\                  只读探针 + 捕获（回退用）
<USB>:\README.TXT                    操作说明
```

部署后逐个比对哈希（源码目录 vs U 盘），一个字节都不能差：
`Get-FileHash -Algorithm SHA256`。资产路径/文件名是**写死**在 `boot_io.c` 的 `GuBootLoadAssets()` 里的，
改名就会被拒。

---

## 9. 阶段七：执行与判读

1. 固件里 Secure Boot 关；UEFI 启动 U 盘。
2. 屏上提示（原文为法语，含义）：

   | 按键 | 行为 |
   |---|---|
   | **U** | 进入一次物理尝试，结束后自动启动 Windows |
   | **S** / **Esc** | 跳过，直接正常 Windows |
   | 30 秒无操作 | 同跳过 |

3. 释放 GPU 期间画面可能**看起来卡死** —— 不要动键鼠，等它自己起 Windows。
4. 结束后 U 盘 `AD104-UNLOCK\RUNxxxx.TXT` 是唯一权威日志。判读顺序：

   ```
   USB baseline ...              ← USB 宿主与 bus agent 已 pin
   FWSEC geometry ...            ← 几何与 profile 一致
   permission ... -> expected    ← WPR/WPR_CFG/FEAT 已放开
   AD102 payload result: marker=... marker_ok=1 halted=1   ← payload 正常结束并 halt
   selector write: SS0=0x88888888 / SS1=0x8, matched=1     ← **核心：覆盖写入并回读一致**
   PCIe FLR returned: result=0                              ← FLR 成功
   post-FLR ... SS0=88888888 SS1=00000008                   ← **覆盖在 FLR 后仍存在**
   Transaction result=0 safe=1 retained=0 ... DMA_live=0    ← 干净收尾
   TRANSACTION OK: ... benchmark still required
   ```

   任何一行出现 `STOP: stage=... status=...` 就是**中止**：机器会主动停机（屏幕显示 STOPPED），
   断电 10 秒后从 Windows Boot Manager 正常启动即可，显卡没有被改。

5. 失败/无响应的处置：**彻底断电** → 等 10 秒 → 从固件启动菜单选 Windows Boot Manager。
   不存在"刷坏了"这种状态，因为整个过程不写 flash。

---

## 10. 阶段八：验证解锁是否真的生效

**不要**只看"能进系统"。用 cuBLAS 做前后对比（`benchmark.py` 就是干这个的）：

```powershell
python benchmark.py --out bench_results\stock-run-1.json    --label stock      # 解锁前
python benchmark.py --out bench_results\unlocked-run-1.json --label unlocked   # 解锁后
```

判定标准（三者同时成立才算成功）：

1. **FP16→FP32 与 BF16→FP32 吞吐大约翻倍**（本机 65.9 → 134.0 TFLOP/s，+103%）；
2. **数值完全一致**：每阶段 `full_256_exact=true`、`max_abs_error=0`、`output_sha256` 与解锁前**同一个值**；
3. **FLR 之后仍然有效**（日志里 `post-FLR SS0/SS1` 回读等于写入值），说明不是靠尚未初始化的假象。

顺带记录 `nvidia-smi` 的 VBIOS/驱动/功耗上限：VBIOS 版本**不应变化**（没刷 flash），功耗上限也不应变化。

---

## 11. 常见坑（本机实测）

| 症状 | 原因 | 处理 |
|---|---|---|
| `file has not been read` / 覆盖文件被拒 | 文件系统策略要求先读后写 | 先 `read`，再 `write`/`edit` |
| PowerShell 拒绝执行 `.ps1` | 执行策略 | `Set-ExecutionPolicy -Scope Process Bypass -Force` |
| `-Werror=misleading-indentation` 编译失败 | GCC 11 对紧凑格式化代码的缩进告警 | 加 `-Wno-misleading-indentation`（只压这一条，别整体降级） |
| JSON 解析报 `Unexpected token \ufeff` | PowerShell `Set-Content -Encoding utf8` 带 BOM | 读时先剥 BOM，或写时用 `-Encoding utf8NoBOM` |
| `Failed to initialize NVML: Unknown Error` | 某些驱动/VM 下 NVML 不可用 | 基准不依赖 telemetry；照常看 TFLOP/s |
| `cublas64_12.dll` 不存在 | 新版 torch 用 cu13 | 脚本按 `cublas64_*.dll` 通配回退 |
| 出现 `%SystemDrive%` 这类怪目录 | 脚本里环境变量未展开 | 删掉，检查脚本里的路径拼接 |
| EFI 里 `hardware_tested: false` | 正常，构建阶段不碰硬件 | 别把它当失败 |
| 预检一直 `EFI_ACCESS_DENIED` | 句柄图计数重复 / 第二显示子句柄未建模 / detach 步数不对 | 回看第 5 节的三条坑 |

---

## 12. 敏感信息处置（发布这份文档时的约定）

本文与配套材料中，**以下内容一律用占位符或删除**：

| 原值 | 处置 |
|---|---|
| 物理 PROM 的 SHA256、ROM 内容 | `<PROM_SHA256>`；二进制不发布 |
| Windows ESP 的磁盘 GUID、分区起始/大小 | `<ESP_GUID>` / `<ESP_PART_START>` / `<ESP_PART_SIZE>` |
| `bootmgfw.efi` 的大小与 SHA256 | `<BOOTMGFW_SIZE>` / `<BOOTMGFW_SHA256>` |
| BAR0 地址、subsystem ID | `<BAR0>` / `<SSID>` |
| 运行期句柄值（形如 `0x6b2a1c18`） | 删除，或写成 `<HANDLE>` 这类占位符 |
| 捕获文件 `CAP*.JSON`、PROM dump、提取出的 `driver-*.bin` | **不发布**（含你的 ROM 与机器布局） |
| 5 个固件资产 | **不发布**（NVIDIA 版权） |
| 本机绝对路径 | `<WORKDIR>` |

可以保留的：GPU 型号与 PCI device ID（公开信息）、驱动版本号、`0x88888888` / `0x8` 这类
selector 常量、寄存器偏移、判读用的日志格式。

---

## 13. 一页速查

```
probe(build) -> boot USB -> CAP*.JSON + PROM dump
   -> 解析 ROM 链 / FWSEC 几何 / 镜像哈希 ──► 与上游一致? ──否──► 重推几何（慢路）
                                              │是
                                              ▼
   -> 重钉 6 类身份 pin（ID/BAR/PROM/冷态/ESP/USB 前缀）
   -> extract_private_capture.py -> tests/*.h + capture_assets
   -> 备 5 个资产（大小+哈希进 profile）
   -> Test-Experimental.ps1（5 套全绿）
   -> Build-Experimental.ps1 -> verify_experimental.py
   -> 部署 U 盘（逐文件哈希比对）
   -> UEFI 启动，按 U，等 Windows 自己回来
   -> 读 AD104-UNLOCK/RUNxxxx.TXT：selector matched=1 + post-FLR 保持 + safe=1
   -> benchmark.py：FP16/BF16→FP32 约翻倍 且 output_sha256 不变
```

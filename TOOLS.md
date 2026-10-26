# 工具清单

分三类：**必装**（不装就无法构建/测试）、**验证用**（确认解锁真的生效）、**排错用**（可选）。
`tools/CHECK-ENV.ps1` 会把第一、二类全部检查一遍并打印版本。

## 一、必装（构建 + 主机测试）

| 工具 | 实测版本 | 用途 | 获取 |
|---|---|---|---|
| Windows 10/11 x64 | — | 全部流程 | — |
| PowerShell | 5.1 / 7.x | 构建与测试脚本 | 系统自带 |
| Git for Windows | 2.4x | 取上游源码、版本管理 | https://git-scm.com/download/win |
| GCC（Cygwin 或 MinGW-w64），**必须带 `ar` 与 `nm`** | Cygwin GCC 11.4 实测通过；上游用 MinGW-w64 GCC 15.2 | 编译 UEFI 应用与全部测试 | https://cygwin.com/ （勾选 gcc-core、binutils、make）或 https://www.mingw-w64.org/ |
| MSVC x64 链接器 `link.exe` | VS 2022 Build Tools | 生成 `.efi`（EFI 子系统入口、`/fixed:no`） | https://visualstudio.microsoft.com/downloads/ → Build Tools → “使用 C++ 的桌面开发” |
| Python | 3.11+ | 资产提取、PE 校验、基准测试 | https://www.python.org/downloads/windows/ |
| FAT32 U 盘 | ≥ 128 MB（资产约 84 MB） | 引导介质 | — |

`vswhere.exe` 随 VS 安装器提供；不传 `-VcVars64` 时脚本会自动定位它。

## 二、验证用（确认解锁生效）

| 工具 | 实测版本 | 用途 | 获取 |
|---|---|---|---|
| PyTorch（CUDA wheel） | 2.13.0.dev + cu132 | 提供 `cublas64_*.dll`、CUDA runtime 与计时 | https://pytorch.org/get-started/locally/ |
| numpy | 任意近期版本 | 256 项逐元素比对、输出哈希 | `pip install numpy` |
| NVIDIA 驱动 | 617.42 实测 | 提供 `nvidia-smi` 遥测 | https://www.nvidia.com/Download/index.aspx |

`benchmark.py` 通过 `ctypes` 直接调用 `cublas64_<ver>.dll`，按版本号自动发现该 DLL，
所以 cu12 / cu13 的 wheel 都能用。

## 三、排错用（可选）

| 工具 | 用途 |
|---|---|
| `dumpbin /disasm`（随 MSVC） | 反汇编捕获到的驱动 `Stop()`，确认 detach 语义 |
| `objdump -D -b binary -m i386:x86-64`（binutils） | 同上，无 MSVC 时使用 |
| 十六进制编辑器（HxD / 010 Editor） | 检查 ROM 链与 FWSEC 头部 |
| UEFI Shell（可选） | 手工 `load` 应用、查看 `dh` 句柄树 |

**不需要**：WSL、Linux、VBIOS 刷写工具、代码签名证书（应用走自签 + 关闭 Secure Boot）。

## 四、本包自带的工具

| 文件 | 作用 |
|---|---|
| `tools/CHECK-ENV.ps1` | 环境体检（本清单的可执行版本） |
| `tools/extract_private_capture.py` | 从捕获 JSON 生成 5 个固件资产（`--out`）与 3 个测试 fixture 头（`--headers`），逐项按 pin 校验 |
| `tools/verify_experimental.py` | 校验生成的 EFI：x64 入口、重定位、无 import/TLS/CLR、逐函数栈帧 |
| `tools/verify_archive.py` | 校验事务归档格式 |
| `tools/pe_format.py` | PE 头部解析辅助 |
| `tools/fwsec_inspect.c` | 独立检查 ROM 内的 FWSEC 几何 |
| `benchmark.py` | cuBLAS GEMM 基准：fp32/fp16/bf16，N 可配，输出 JSON + 精确性比对 |
| `compare.py` | 两份 benchmark JSON 对比（吞吐、误差、输出哈希） |
| `Build.ps1` / `Build-Experimental.ps1` | 构建核心库 / 构建 `.efi` |
| `Test-Experimental.ps1` | 全部主机测试套件（17 准入 + 90 核心 + 15 app flow + 49 msi-release + 8 boot I/O） |
| `diagnostic/Build.ps1` | 构建只读探针 `AD104-PROBE.EFI` |

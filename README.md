# AD104 Windows 解锁端口 — 可复刻发布包 v0.5.4

在 NVIDIA 驱动加载**之前**（UEFI preboot）完成 AD104 firmware 事务并写入 SM speed-select
覆盖，随后交还原版 `bootmgfw.efi` 正常启动 Windows。**不写 VBIOS flash**，断电即恢复原状。

方法源自 [JustAResearcher/gpu-unlock-ada](https://github.com/JustAResearcher/gpu-unlock-ada)（GPL-2.0）
的 Linux 内核模块思路，本包是它的 **Windows + UEFI** 实现与复刻套件。

## 实测收益（RTX 4070 Ti / AD104，N=4096 cuBLAS GEMM）

| 累加模式 | 未解锁 | 解锁后 | 变化 |
|---|---:|---:|---:|
| FP32 pedantic | 25.15 TFLOP/s | 27.54 TFLOP/s | +9.5%（时钟噪声，不算收益） |
| FP16 → FP32 | 65.92 TFLOP/s | **133.95 TFLOP/s** | **+103.2%** |
| BF16 → FP32 | 65.40 TFLOP/s | **134.20 TFLOP/s** | **+105.2%** |

三种模式解锁前后 `output_sha256` 完全一致，`max_abs_error = 0`：
**吞吐翻倍、逐元素结果不变**。完整证据与方法见 `evidence/`，判读方法见
`REPLICATION-GUIDE.md` 开头两节。

## 包内结构

```
README.md                 本文件
TOOLS.md                  工具清单（必装 / 验证用 / 排错用）与本包自带工具
REPLICATION-GUIDE.md      完整复刻手册（性能证据、pin 推导、五个测试套件、故障排查）
evidence/                 性能对比数据 + 一次成功运行的（已脱敏）日志 + 复现命令
tools/CHECK-ENV.ps1       一键体检：本机是否具备构建/测试/基准测试所需的工具
local.example/            需要你自己填的两份本地配置模板（不进 Git）
  boot_profile.h          → 复制到 local/boot_profile.h 并填入你的 ESP 分区 GUID
  pins.json               → 复制到 local/pins.json 并填入你的 PROM 哈希与驱动代码哈希
<源码>                    app.c / boot_io.c / msi_release.c / profile.c / port.c …
diagnostic/               只读探针（产生捕获 JSON，永不写 GPU）
tests/                    五个主机测试套件（全部模拟，不接触真实硬件）
vendor/                   从上游移植的 firmware 事务核心（GPL-2.0）
```

本包**不含**、也永远不该包含：真实 ROM 或 NVIDIA 固件资产、`captures/` 原始捕获、
`tests/captured_config.h` `tests/cap4_graph.h` `tests/cap4_usb_paths.h`（必须由你的捕获生成）、
`local/`（本机 ESP GUID）、任何 `build/` 产物。

## 快速开始（6 步）

```powershell
# 0) 体检环境（缺什么会直接告诉你）
pwsh -File tools/CHECK-ENV.ps1

# 1) 只读捕获：得到一个 JSON（这一步不写 GPU）
#    按 REPLICATION-GUIDE.md 第 3 节构建 diagnostic/Build.ps1，放到 U 盘 EFI\BOOT\BOOTX64.EFI，
#    UEFI 启动一次，拿回 CAPxxxx.JSON

# 2) 从捕获生成 5 个固件资产 + 3 个测试 fixture 头（一步完成，全部按捕获校验）
python tools/extract_private_capture.py CAPxxxx.JSON --out build/capture_assets --headers tests

# 3) 填本地配置
#    local/boot_profile.h  ← 你的 ESP 分区 GUID（依据捕获里的 HARDDRIVE_DEVICE_PATH）
#    local/pins.json       ← 你的 PROM SHA256 + 每个 agent 的 code.loaded_sha256
#    并按 REPLICATION-GUIDE.md 第 4 节的表格，把 port.h / profile.c / boot_io.c / msi_release.c
#    里的占位值改成你自己的（本包已把 ESP 几何、bootmgfw 大小/哈希、PROM 哈希置零）

# 4) 构建 + 全套主机测试（纯模拟，安全）
.\Build-Experimental.ps1 -Gcc 'C:\cygwin64\bin\gcc.exe' -Python python `
  -VcVars64 'C:\Program Files\Microsoft Visual Studio\2022\BuildTools\VC\Auxiliary\Build\vcvars64.bat'
.\Test-Experimental.ps1 -Gcc 'C:\cygwin64\bin\gcc.exe' -Prom 'captures\your_prom.bin' `
  -Assets 'build\assets' -DriverCode 'build\capture_assets' `
  -WindowsImage 'C:\Windows\Boot\EFI\bootmgfw.efi' -IncludeCore

# 5) 部署到 FAT32 U 盘并启动，屏上按 U 触发一次尝试
#    <USB>:\EFI\BOOT\BOOTX64.EFI   +   <USB>:\AD104-ASSETS\*.bin（5 个）
```

## 你必须自己填的值（照抄别人的 = 卡在预检，不会损坏任何东西）

| 值 | 来源 | 写到哪里 |
|---|---|---|
| PROM SHA256（1 MiB 全片） | 捕获 `physical_prom` | `profile.c`（`GuAd104PromMatches`）+ `local/pins.json` |
| Windows ESP：分区号 / 起始 LBA / 扇区数 / GUID | 捕获里的 `MEDIA_HARDDRIVE_DP` | `boot_io.c`、`tests/test_boot_io.c`、`local/boot_profile.h` |
| `bootmgfw.efi` 大小 + SHA256 | 你自己的 ESP | `boot_io.c`、`tests/test_boot_io.c` |
| 板卡 SSID、BAR0、冷态 SS0 | 捕获 `config256` / `register_snapshot` | `port.h`、`profile.c` |
| 4 个 GPU agent 的 `code.loaded_sha256` | 捕获 `platform.gpu_agents` | `local/pins.json` |
| USB 宿主路径前缀 | 捕获里 USB 分支父句柄的设备路径 | `msi_release.c`（`UsbPointerPath` 的 `host[]`） |

任何一项对不上，应用会在**写 GPU 之前**拒绝运行并打印 `STOP: stage=…`，
这是设计目标，不是 bug。**不要为了让它跑起来而放宽守卫。**

## 安全边界

* 只在运行期寄存器层面生效，**不写 flash**；完整断电即回到 stock。
* 每一次成功都要靠日志（事务 OK）和基准测试（吞吐翻倍）**两条独立证据**同时成立。
* 失败路径：应用会主动 halt（此时显示已释放，不能让 Windows 带着半完成状态启动）。
  处理办法只有一个——**彻底断电**、等 10 秒、从固件菜单选 Windows Boot Manager。
* 它是为「一台机器、一张卡、一次人工尝试」设计的，不是通用安装器。

## 许可与出处

* 本端口：GPL-2.0-only（见 `LICENSE`）。
* 上游方法与 `vendor/` 事务核心：JustAResearcher/gpu-unlock-ada，GPL-2.0。
* 第三方声明：`THIRD_PARTY_NOTICES.md`、`UPSTREAM-WINDOWS-NOTICES.md`、`LICENSES/`、`diagnostic/THIRD_PARTY.md`。
* 不含任何 NVIDIA 二进制：固件资产由复刻者用上游 hash-pinned 生成流程从**自己的** ROM 产生。

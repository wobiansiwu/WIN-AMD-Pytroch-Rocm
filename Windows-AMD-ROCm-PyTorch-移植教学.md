# Windows + AMD 显卡零编译跑 PyTorch：RX 6750 GRE（gfx1031）ROCm 移植实战

> **适用读者**：手持 AMD Radeon 消费级显卡（RDNA / RDNA2 / RDNA3）、想在 Windows 11 上用 GPU 跑 PyTorch 炼丹，但被「AMD 不支持 CUDA」卡住的开发者。
>
> **本文定位**：**不编译源码**、**一条 pip 命令**完成移植。文中所有命令与输出均来自一次真实成功的移植过程（Windows 11 + RX 6750 GRE 12GB），非纸面推演。
>
> **文档版本**：基于 ROCm 10.0.0（stable 通道）/ PyTorch 2.13.0 实测。

---

## 目录

- [一、背景与目标](#一背景与目标)
- [二、前置准备](#二前置准备)
- [三、详细操作步骤](#三详细操作步骤)
- [四、常见报错与解决（踩坑记录）](#四常见报错与解决踩坑记录)
- [五、测试与验证](#五测试与验证)
- [六、总结与展望](#六总结与展望)
- [附录：参考资料](#附录参考资料)

---

## 一、背景与目标

### 1.1 为什么要做这次移植

在 Windows + AMD 消费级显卡上跑 PyTorch，长期是「地狱难度」，原因有三层：

| 层级 | 历史上的问题 |
|---|---|
| **驱动层** | ROCm 长期只支持 Linux，Windows 支持直到 ROCm v7 才起步 |
| **运行时层** | AMD 官方 ROCm 的兼容性矩阵里，RDNA2 消费卡长期**只列 `gfx1030`**，`gfx1031` / `gfx1032` 等一律不在名单 |
| **框架层** | PyTorch **没有 Windows + ROCm 的官方发行版**，只能自己编译 |

2026 年 2 月，有博主（[曦远Code](https://www.cnblogs.com/deali/)）连续写了两篇文章记录在 Windows 上给 RX 6650 XT（`gfx1032`）编译 PyTorch 的过程：**第一篇直接宣告失败**，第二篇才编译成功。那条路的代价是：装 Visual Studio 2022、拉几十 GB 源码、编译几小时、还要打源码补丁。

**但这条路现在已经没必要走了。**

AMD 的 [TheRock](https://github.com/ROCm/TheRock) 项目（ROCm 的主开发仓库，AMD 官方维护）已经转向 **多架构（multi-arch）发布模式**，开始直接发布 **Windows 平台的预编译 PyTorch 轮子**，并按显卡架构拆分成 `device-*` 扩展包。

与此同时：

- TheRock issue [#1002「[Windows] How could I build for gfx1031?」](https://github.com/ROCm/TheRock/issues/1002) 已于 **2026-02-12 标记完成并关闭**（由 PR #1629 解决）
- [SUPPORTED_GPUS.md](https://github.com/ROCm/TheRock/blob/main/SUPPORTED_GPUS.md) 中，Windows 平台的 `gfx1031` 状态为 **Build Passing ✅ / Sanity Tested ✅ / Release Ready ✅** —— 三项全绿
- AMD 官方文档在「GPU 不在支持列表」时，明确指引用户转向 TheRock：

  > If your GPU is not listed, it might be community-enabled through TheRock nightly builds. For more information, see TheRock supported GPUs. For installation guidance, see TheRock releases.

**所以本次移植的核心目标就是：验证「零编译」路线在真实的 gfx1031 显卡上是否可行。**

### 1.2 最终达到的效果

**结论：完全可行。全程未编译任何源码，未安装 Visual Studio，未拉取 PyTorch 源码。**

实机验收数据（真实输出，非预期值）：

```
python     : 3.12.14
torch      : 2.13.0+rocm10.0.0
hip/rocm   : 7.15.26333
available  : True                          ← GPU 可用
device cnt : 1
device 0   : AMD Radeon RX 6750 GRE 12GB
gcnArch    : gfx1031                       ← 架构识别正确
vram       : 12.0 GB

matmul 4096³ x20 : 0.0129 s/iter  ->  10.6 TFLOPS
conv2d           : (8,3,640,640) -> (8,32,640,640)  OK
train            : loss 2.4124 -> 2.3525            ← 反向传播 + 优化器 OK
autocast fp16    : (32,10) OK                       ← 混合精度 OK
vram             : free 11.15 / total 11.98 GB
```

### 1.3 三条最重要的结论（时间紧可只看这里）

1. **不要编译**。官方预编译轮子已经存在，一条 `pip install` 搞定。
2. **不要用 Python 3.10**。用 **Python 3.12 / 3.13 / 3.14**，可以避开 `numpy` 缺轮子的坑（详见 [2.4](#24-关键决策为什么必须避开-python-310)）。
3. **不要装 HIP SDK**。pip 方案自带所需运行时，装了 HIP SDK 反而会冲突。

---

## 二、前置准备

### 2.1 硬件环境

| 项目 | 本次实测配置 |
|---|---|
| GPU | AMD Radeon RX 6750 GRE 12GB（Navi 22，`gfx1031`） |
| 显存 | 11.98 GB |
| 核心频率 | 2439 MHz |
| 显存位宽 | 192 bit |
| CPU / 内存 | [此处需补充：本次未记录 CPU 与内存规格] |

> **其他显卡怎么办？**
> 本文流程是通用的，只需把命令里的 `device-gfx1031` 换成你自己卡的架构代号。查表见 [ROCm TheRock GFX target 对照表](https://github.com/ROCm/TheRock/blob/main/RELEASES.md#supported-python-device--install-extras)，或 [AMD 官方 GPU 架构规格表](https://rocm.docs.amd.com/en/latest/reference/gpu-arch-specs.html)。
> 常见对照：RX 7900 XTX/XT → `gfx1100`；RX 7800 XT/7700 XT → `gfx1101`；RX 7600 → `gfx1102`；RX 6900/6800 XT → `gfx1030`；**RX 6750 XT/6700 XT/6750 GRE → `gfx1031`**；RX 6600 XT/6600 → `gfx1032`；RX 6500 XT → `gfx1034`。

### 2.2 软件环境

| 项目 | 本次实测值 | 要求 / 说明 |
|---|---|---|
| 操作系统 | Windows 11 build 26200（**25H2**） | 官方兼容性矩阵要求 Windows 11 **25H2** ✅ |
| 显卡驱动 | **AMD Software 26.8.1**（驱动号实测 `32.0.21045.5002`，会随驱动更新变动） | 官方矩阵要求 Adrenalin **26.8.1** ✅ **恰好符合，无需升级** |
| 包管理器 | conda 26.7.1（conda-forge 走中科大镜像） | 用 miniconda / anaconda / 纯 venv 均可 |
| Python | **3.12.14** | **必须 ≥ 3.10，强烈建议 3.12+** |
| 磁盘可用空间 | C: 29.9 GB / D: 167.9 GB | 环境实际占用 **3.65 GB**，加上 pip 缓存需预留 **10 GB** 以上 |

### 2.3 必须事先确认的三件事

#### ① 确认你的 GPU 架构代号（gfx target）

这是整个流程最关键的一个参数，**填错就会装到别的卡的 GPU 内核包上**。

```powershell
# 方法一：查 AMD 官方架构对照表（推荐）
# https://rocm.docs.amd.com/en/latest/reference/gpu-arch-specs.html

# 方法二：查 TheRock 的 GFX target 对照表
# https://github.com/ROCm/TheRock/blob/main/RELEASES.md

# 方法三：用 GPU-Z 看显卡架构（如 Navi 22），
#         再去 Mesa 源码里查对应代号
# https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/src/amd/common/amd_family.c
```

> **【注意事项】**
> - 架构代号（`Navi 22`）和 LLVM target（`gfx1031`）**不是一回事**，不要混用。
> - 同一张卡在 AMD 官网的「产品名」和 ROCm 文档里的「产品名」可能不同。例如本文的 **RX 6750 GRE 12GB**，官方对照表里写作 **RX 6750 XT / 6700 XT**（同属 Navi 22 核心，同为 `gfx1031`）。**判断依据请以 gfx target 为准，不要以商品名为准。**
> - **不要凭猜测填 `gfx1030`**。虽然同在 RDNA2，但 `gfx1030` 和 `gfx1031` 的 GPU 内核包不通用。

#### ② 确认驱动版本

```powershell
# 查看已安装的 AMD Software 版本（通过注册表卸载项）
Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*",
                 "HKLM:\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*" `
                 -ErrorAction SilentlyContinue |
  Where-Object { $_.DisplayName -match 'AMD Software' } |
  Select-Object DisplayName, DisplayVersion | Format-Table -AutoSize

# 查看显卡驱动号
Get-ItemProperty 'HKLM:\SYSTEM\CurrentControlSet\Control\Class\{4d36e968-e325-11ce-bfc1-08002be10318}\*' `
                 -ErrorAction SilentlyContinue |
  Select-Object DriverDesc, DriverVersion | Format-List
```

> **【注意事项】**
> - Windows 上的 ROCm **跑在 AMD 显卡驱动之上**。驱动太旧会直接导致 `torch.cuda.is_available()` 返回 `False`。
> - 请安装 **「AMD Software: Adrenalin Edition」完整版**，不要只装纯 Driver。
> - 官方兼容性矩阵（ROCm 10.0.0）对 Radeon 的要求是 **Windows 11 25H2 + Adrenalin 26.8.1**。
> - 装/升级驱动后**必须重启**。

#### ③ 清理 HIP SDK / 旧 ROCm 残留

```powershell
# 检查是否有 ROCm / HIP 残留目录
foreach ($p in @("C:\Program Files\AMD\ROCm",
                 "C:\Program Files (x86)\AMD\ROCm")) {
  if (Test-Path $p) { Write-Host "⚠️ 存在残留: $p" -ForegroundColor Yellow }
  else              { Write-Host "✅ 不存在: $p" -ForegroundColor Green }
}

# 检查 PATH 里有没有 ROCm / HIP 条目
$env:PATH -split ';' | Where-Object { $_ -match 'rocm|hip' }

# 检查 HIP / ROCm 相关环境变量
Get-ChildItem env: | Where-Object { $_.Name -match 'HIP|ROCM|HSA' }
```

> **【注意事项】**
> - TheRock 官方文档明确说明：**系统里如果装过 HIP SDK / 旧版 ROCm，会导致运行和构建出错，建议先卸载**。
> - ⚠️ **卸载 ROCm 后可能残留空目录 `C:\Program Files\AMD\ROCm`**，务必手工删除。TheRock issue #1002 中就有开发者被这个空目录坑到——构建脚本扫到该目录后解析版本号失败，抛出 `ValueError: max() arg is an empty sequence`。
> - 只装「AMD Software 显卡驱动」不会产生这些残留，本文实测环境中上述目录均不存在。

### 2.4 关键决策：为什么必须避开 Python 3.10

这是一个**很容易踩、且报错信息极具误导性**的坑，值得单独说明。

**背景**：AMD 的包索引里，`numpy` 只镜像了 4 个 Windows 轮子：

```
numpy-2.5.2-cp312-cp312-win_amd64.whl
numpy-2.5.2-cp313-cp313-win_amd64.whl
numpy-2.5.2-cp314-cp314-win_amd64.whl
numpy-2.5.2-cp314-cp314t-win_amd64.whl     ← 注意：没有 cp310！
```

**影响**：`torchvision` 的元数据里 **`numpy` 是硬依赖**（已核实）：

```
torchvision 0.28.0 → requires: numpy, torch==2.13.0, pillow!=8.3.*,>=5.3.0
```

而 `torch` 本身**不依赖 numpy**——它的依赖全是纯 Python 包（依据 PyPI 上 `torch 2.13.0` 的 `requires_dist` 元数据；AMD 的 `+rocm10.0.0` 变体由同一份源码构建，依赖列表应当一致，但**未逐字核对 AMD wheel 的元数据**）：

```
torch 2.13.0 → requires: filelock, typing-extensions, setuptools,
                         sympy, networkx, jinja2, fsspec
```

**结论**：

| Python 版本 | 直接用 `--index-url`（AMD 索引）装 torchvision | 说明 |
|---|---|---|
| **3.12 / 3.13 / 3.14** | ✅ **成功（本次实测：3.12.14）** | 索引里有对应的 numpy 轮子 |
| **3.10** | ❌ **预期解析失败（🧠 推断，未实测）** | 索引里没有 cp310 的 numpy，需要额外绕路（见 [3.4](#34-可选python-310-的两步法绕开-numpy-缺包)） |

> **【注意事项】**
> - 「Python 3.10 会失败」是**基于索引实际内容与依赖元数据的推断**，本次移植发现该问题后**直接改用 Python 3.12**，因此未真正跑出该报错。若你实际用 3.10 执行并成功，说明索引已补齐 cp310 的 numpy，欢迎回补本文档。
> - Python 3.10 能用的最新 numpy 是 **2.2.6**（numpy 2.3 起不再支持 3.10）。
> - 这也侧面说明：**AMD 自己 CI 实测的组合是 Python 3.12+**。
> - 如果你已经有一个 Python 3.10 的环境（比如为了兼容老项目），要么按 3.4 的两步法绕路，要么新建一个 3.12 环境——**强烈推荐后者，干净且不污染老环境**。

---

## 三、详细操作步骤

### 3.1 步骤总览

| 步骤 | 操作 | 预计耗时 |
|---|---|---|
| 步骤 1 | 创建 conda 环境（Python 3.12） | 1–3 分钟 |
| 步骤 2 | 一条命令安装 PyTorch + ROCm | 3–10 分钟（下载约 1.2 GB） |
| 步骤 3 | 验证 GPU 可用性 | 1 分钟 |
| 步骤 4 | （仅 Python 3.10）numpy 绕路 | 1 分钟 |

---

### 3.2 步骤 1：创建独立的 conda 环境

**操作逻辑**：把 ROCm + PyTorch 关进一个独立环境，避免与现有环境（尤其是已有 CUDA 版 torch 的环境）冲突。ROCm 运行时体积不小，**不要装进 base 环境**。

```powershell
# 创建名为 rocm-torch 的环境，指定 Python 3.12
conda create -n rocm-torch python=3.12 -y

# 确认环境创建成功
conda env list
```

预期输出：

```
# conda environments:
#
rocm-torch               C:\Users\<用户名>\.conda\envs\rocm-torch
base                 *   D:\Mysoftware\anaconda3
```

> **【注意事项】**
> - **不要用 Python 3.10**，原因见 [2.4](#24-关键决策为什么必须避开-python-310)。
> - **不要装进 base 环境**：将来升级/卸载 conda 会连带毁掉整个 ROCm 栈。
> - conda 在国内建议配置镜像源（本文实测使用中科大源 `mirrors.ustc.edu.cn`，速度良好）。
> - ⚠️ **如果报 `NoWritableEnvsDirError: No writeable envs directories configured`**：这是 conda 对 `envs` 目录**没有写权限**导致的（在受限沙箱/低权限账户下会出现）。解决办法见 [4.5](#45-c-类权限--目录类) 。
> - 不想用 conda 的话，`python -m venv` 完全可以替代，且更轻量——这也是 AMD 官方推荐的方式。用 venv 时后续命令把 `C:\...\envs\rocm-torch\python.exe` 换成 `<venv>\Scripts\python.exe` 即可。

---

### 3.3 步骤 2：一条命令安装 PyTorch + ROCm

**操作逻辑**：AMD 的稳定索引把「ROCm 运行时 + PyTorch + 你显卡专属的 GPU 内核包」都准备好了，用 `--index-url` 指向它，并通过 `[device-xxx]` 扩展选出自己显卡对应的内核包。

```powershell
# 直接使用环境的绝对路径调用 python，确保操作的是目标环境的 pip
& "C:\Users\<用户名>\.conda\envs\rocm-torch\python.exe" -m pip install `
    --index-url https://stable.repo.amd.com/rocm/whl-next/ `
    "torch[device-gfx1031]" `
    "torchvision[device-gfx1031]" `
    torchaudio
```

**参数逐一解释**：

| 参数 | 作用 |
|---|---|
| `--index-url https://stable.repo.amd.com/rocm/whl-next/` | 指向 **AMD 官方稳定通道**（ROCm 10.0.0）。该索引是**聚合索引**，不仅含 AMD 包，也镜像了 torch 所需的全部依赖（numpy / sympy / filelock / pillow…），所以可以单独使用 |
| `torch[device-gfx1031]` | `[device-*]` 是**多架构发布模式**的核心：告诉 pip 除了装架构无关的 host 代码，还要装**我这张卡专属的 GPU 内核**（会连带拉取 `amd-torch-device-gfx1031` + `rocm-sdk-device-gfx1031`） |
| `torchvision[device-gfx1031]` | torchvision 同样有专属 GPU 内核包 `amd-torchvision-device-gfx1031` |
| `torchaudio` | 无架构专属包，直接装 |

**实际下载内容与体积**（本次实测）：

| 包 | 大小 |
|---|---|
| `rocm-sdk-core` | **758.1 MB** ← 大头 |
| `rocm-sdk-libraries` | 116.8 MB |
| `torch` | 113.4 MB |
| `rocm-sdk-device-gfx1031` | 52.2 MB |
| `amd-torch-device-gfx1031` | 44.8 MB |
| numpy / pillow / sympy / networkx 等 | 约 30 MB |
| torchaudio / torchvision / 其他 | 约 5 MB |
| **合计** | **约 1.2 GB 下载量，安装后占 3.65 GB** |

> **【注意事项】**
> - 🔴 **千万不要用 `pip install torch`（不带 `--index-url`）**。PyPI 上的 `torch` 是 CPU/CUDA 版，会覆盖掉 ROCm 版。
> - 🔴 **`--index-url` 会替换掉 PyPI**，只对**这一条命令**生效，不影响后续 `pip install` 其他包。
> - ⚠️ **`--index-url` 与 `--extra-index-url` 不要混用**（除非精确锁版本）。原因是 PEP 440 中 `2.13.0+rocm10.0.0` 被视为**大于** `2.13.0`，但因为本地版本号的存在，两个索引的候选会一起参与解析，容易被 PyPI 的裸版本抢走。若确实要混用，必须写成 `"torch[device-gfx1031]==2.13.0+rocm10.0.0"` 这种**带 `+rocm10.0.0` 的精确版本**。
> - ⚠️ **AMD 索引不支持 PEP 658 元数据**，pip 必须**下载完整 wheel 才能读取其依赖元数据**。因此你会看到 pip 在「Collecting」阶段就把 758 MB 的 `rocm-sdk-core` 下完——**这是正常的，不是卡死**。
> - ⚠️ `torch` 与 `torchvision` 的版本必须匹配。ROCm 10.0.0 稳定通道的对应关系是 **torch 2.13.0 ↔ torchvision 0.28.0 ↔ torchaudio 2.11.0.2**。乱配会导致 import 报错。
> - 💡 想用最新（但**不稳定**）的每日构建，把索引换成 `https://nightly.repo.amd.com/rocm/whl-next/` 即可（当时是 torch 2.15.0 + ROCm 10.2.0a）。**生产环境建议先用 stable。**
> - 💡 装好后 pip 会提示一堆 `WARNING: The scripts ... are installed in '...\Scripts' which is not on PATH`。**这是正常的**，`hipInfo.exe` / `rocm-sdk.exe` 等工具用绝对路径调用即可。

---

### 3.4 （可选）Python 3.10 的两步法：绕开 numpy 缺包

**适用场景**：你**必须**用某个已存在的 Python 3.10 环境（例如本文中名为 `yolov5` 的环境，实测为 Python 3.10.21）。

**操作逻辑**：既然 AMD 索引里没有 cp310 的 numpy，那就**先从 PyPI 把 numpy 装好**；这样第 2 步 pip 解析 `torchvision` 时发现 `numpy` 已被满足，就不会再去 AMD 索引找它。

```powershell
# ── 第 1 步：先从 PyPI 装 numpy（cp310 能用的最新版是 2.2.6）──
& "C:\Users\<用户名>\.conda\envs\<你的310环境>\python.exe" -m pip install `
    --index-url https://pypi.org/simple/ `
    "numpy==2.2.6"

# ── 第 2 步：再从 AMD 索引装 PyTorch 全家桶（顺序不能反！）──
& "C:\Users\<用户名>\.conda\envs\<你的310环境>\python.exe" -m pip install `
    --index-url https://stable.repo.amd.com/rocm/whl-next/ `
    "torch[device-gfx1031]" `
    "torchvision[device-gfx1031]" `
    torchaudio
```

> **【注意事项】**
> - **两步顺序绝对不能反**。先装 torch 会直接卡在 numpy 解析失败上。
> - 第 2 步**不要**加 `--extra-index-url`，保持「只认 AMD 索引」才能确保拿到的是 `+rocm10.0.0` 版本。
> - numpy 必须锁 **2.2.6**：numpy 2.3 起不再支持 Python 3.10，装 `numpy` 不锁版本会被解析到不兼容的版本。
> - 💡 **更省事的替代方案**：新建一个 Python 3.12 环境，一条命令搞定，还不会让老环境背上 3.65 GB 的 ROCm。本文最终就选择了这条路。

---

### 3.5 步骤 3：验证

```powershell
# ── 基础验证：GPU 是否被识别 ──
& "C:\Users\<用户名>\.conda\envs\rocm-torch\python.exe" -c `
  "import torch; print(torch.__version__); print(torch.cuda.is_available()); print(torch.cuda.get_device_name(0))"
```

预期输出：

```
2.13.0+rocm10.0.0
True
AMD Radeon RX 6750 GRE 12GB
```

详细的验证方法（含算力实测、conv2d、反向传播、混合精度）见 [第五章](#五测试与验证)。

> **【注意事项】**
> - **第一次 `import torch` 会明显偏慢**（需加载大量 ROCm DLL），可能 10–30 秒，属于正常现象。
> - 如果 `torch.cuda.is_available()` 返回 `False`，先看 [4.4](#44-d-类运行时类)。

---

## 四、常见报错与解决（踩坑记录）

> 本章是全文**最重要的部分**。按「坑的类别」分组，每条都给出**症状 → 原因 → 解决**。
>
> **【重要：证据等级标注】** 为避免误导，本文对每条坑标注了证据等级：
>
> | 标记 | 含义 |
> |---|---|
> | 🔬 **本次实测** | 在本次真实移植过程中**亲自撞到并解决**，报错信息为原始输出 |
> | 🧠 **推断** | 基于官方文档、包索引实际内容与语义化版本规则**推理得出**，本次**未实际触发** |
> | 🛡️ **预防性** | 本次未遇到，但属于该路线的**已知风险**，提前给出规避方法 |
> | 📜 **历史** | 来自第三方教程（编译路线），按本文路线**不会遇到**，仅供理解背景 |

### 4.1 A 类：认知类坑（最大、也最容易浪费时间）

#### A-1 ⭐ 最大的坑：以为「必须自己编译 PyTorch」　🔬 **资料考证（本次核心发现）**

**症状**：照着网上（包括 2026 年 2 月那两篇热门博客）的教程，去 clone `ROCm/TheRock`、装 Visual Studio 2022、拉 PyTorch 源码、然后编译几个小时，最后失败。

**原因**：**教程过时了**。那两篇博客写于 2026 年 2 月初，当时：

- AMD 官方 ROCm 包不支持 `gfx103X`
- PyTorch 没有 Windows + ROCm 的官方轮子

而**现在（ROCm 10.0.0 / TheRock 多架构发布模式）这两条都不成立了**。

**时间线证据**：

| 时间 | 事件 |
|---|---|
| 2025-07-10 | TheRock issue #1002 提出「Windows 上怎么给 gfx1031 构建？」 |
| 2025-07-11 | AMD 员工回复：需要改 `therock_amdgpu_targets.cmake`，并附上 hipBLASLt 的构建失败日志 |
| 2025-07-14 | 社区开发者给出补丁思路（往白名单加 `gfx1031`） |
| **2026-02-05** | 博主第一篇博客：**编译失败** |
| **2026-02-06** | 博主第二篇博客：改用「官方 ROCm 包 + PyTorch 降至 2.9.1」，**编译成功** |
| **2026-02-12** | **TheRock issue #1002 关闭（PR #1629 解决）** |
| 现在 | `SUPPORTED_GPUS.md` 中 Windows 的 `gfx1031` 已是 **Release Ready ✅**，预编译轮子直接可用 |

**解决**：**放弃编译路线，直接用预编译轮子**（即本文第三章）。

> **【教训】** 遇到「A 卡 + Windows + PyTorch」的教程时，**先看发布日期**。2026 年 2 月之后的教程才有参考价值。

---

#### A-2 ⭐ 「官方 ROCm」到底在哪？——官方安装页是选择器驱动的　🔬 **本次实测（调取了页面 rst 源码）**

**症状**：在 AMD 官网找到了 [ROCm 安装页](https://rocm.docs.amd.com/en/docs-10.0.0/install/rocm.html)，但里面的内容完全用不上（比如显示的是 Instinct MI355X + Ubuntu + apt）。

**原因**：该页面是**参数驱动的模板页**。URL 里的每个 query 参数都决定显示哪一段内容：

```
?fam=instinct              ← 设备家族（Instinct / Radeon / Ryzen）
&gpu=amd-instinct-mi355x   ← 具体型号
&gfx=gfx950                ← 架构代号（CDNA4 数据中心卡）
&os=ubuntu&ubuntu-ver=26.04← 操作系统
&i=pkgman                 ← 安装方式（apt / dnf / pip / tarball）
```

**一条 MI355X 的链接对你的 Windows + 6750 GRE 毫无意义。**

**解决**：**改 URL 参数**，换成你自己的配置：

```
https://rocm.docs.amd.com/en/docs-10.0.0/install/rocm.html?fam=radeon&w=compute&os=windows&windows-ver=11&i=pip
```

> **【补充知识】** 从该页的 [rst 源码](https://rocm.docs.amd.com/en/docs-10.0.0/_sources/install/rocm.rst) 可见，**Windows 分支下只提供两个安装方式**：`pip` 和 `Tarball`。没有 apt/dnf/zypper。
> 而且 **Tarball 装的是 ROCm SDK 本身，不会给你 PyTorch**。
> ➡️ **所以：Windows 上要用 PyTorch + ROCm，终点必然是 `pip` 装 torch 的 ROCm 轮子。**

---

#### A-3 ⭐ 你的显卡不在官方支持列表里 —— 这是「正常」的　🔬 **本次实测（核对了官方兼容性矩阵）**

**症状**：翻遍 AMD 官方兼容性矩阵，找不到自己的显卡。

**原因**：ROCm 10.0.0 官方兼容性矩阵的 Radeon 部分，**支持的 LLVM target 只有 6 个**：

| 架构 | 官方支持的 target | 对应产品 |
|---|---|---|
| RDNA 4 | `gfx1201`、`gfx1200` | RX 9070 / 9060 系列 |
| RDNA 3 | `gfx1100`、`gfx1101`、`gfx1102` | RX 7900 / 7800 / 7700 / 7600 |
| **RDNA 2** | **只有 `gfx1030`** | Radeon PRO W6000 系列 |

**`gfx1031`（RX 6750 GRE / 6750 XT / 6700 XT）根本不在里面。**

**解决**：走 TheRock 通道 —— 这不是「非官方野路子」，因为：

1. TheRock 就是 **AMD/ROCm 官方 GitHub 组织下** 的仓库，是 ROCm 的主开发仓库
2. AMD 官方文档自己写着：「If your GPU is not listed, it might be community-enabled through TheRock nightly builds」
3. ROCm 10.0.0 官方文档中专门有一篇 [Transition guide (TheRock)](https://rocm.docs.amd.com/en/docs-10.0.0/about/transition-guide-TheRock.html)
4. 包索引域名是 `repo.amd.com`，属于 AMD 官方资产

> **【一句话】** TheRock 的包**是官方的**，只是发布通道（stable/nightly）不同于「正式 release 下载页」。

---

### 4.2 B 类：依赖解析类

#### B-1 ⭐ `numpy` 解析失败（Python 3.10 专属）　🧠 **推断（未实际触发）**

> ⚠️ **证据说明**：本条是**基于包索引实际内容 + PyPI 元数据的推断**。本次移植在发现该问题后**改为新建 Python 3.12 环境**，因此**没有真正跑出这个报错**。报错文本为推断，请以实际为准。

**症状**（预期）：用 Python 3.10 环境执行安装命令时，pip 卡在 numpy 上无法解析（或最终报找不到匹配的 numpy 版本）。

**原因**（已核实）：AMD 索引里的 `numpy` **只有 cp312 / cp313 / cp314**（版本 2.5.2），**没有 cp310**。而 `torchvision` 的元数据里 `numpy` 是**硬依赖**。

**解决**：见 [3.4 两步法](#34-可选python-310-的两步法绕开-numpy-缺包)，或直接改用 Python 3.12+。

> **【注意事项】** Python 3.10 能用的最新 numpy 是 **2.2.6**（numpy 2.3 起放弃 3.10），必须显式锁定。
> 💡 顺带一提：**无论 `torch` 自身是否依赖 numpy，3.4 的两步法都是有效的**——因为 numpy 被预先满足后，第 2 步就不会再去索引里找它。

---

#### B-2 ⭐ 版本被 PyPI 抢走 / 装成了 CPU 版　🛡️ **预防性（本次未触发）**

**症状**：装完之后 `torch.__version__` 显示 `2.13.0` 而不是 `2.13.0+rocm10.0.0`，`torch.cuda.is_available()` 返回 `False`。

**原因**：混用了 `--index-url` 和 `--extra-index-url`，或者干脆忘了加 `--index-url`，导致 pip 从 PyPI 抓了 CPU/CUDA 版的 torch。

**解决**：

```powershell
# 先卸干净
pip uninstall torch torchvision torchaudio -y

# 再用「只认 AMD 索引」的方式重装（注意 --index-url 而非 --extra-index-url）
& "C:\Users\<用户名>\.conda\envs\rocm-torch\python.exe" -m pip install `
    --index-url https://stable.repo.amd.com/rocm/whl-next/ `
    "torch[device-gfx1031]" "torchvision[device-gfx1031]" torchaudio
```

> **【注意事项】** 若确实需要 `--extra-index-url`，必须精确锁版本：`"torch[device-gfx1031]==2.13.0+rocm10.0.0"`。

---

#### B-3 依赖包被别的库「顺手升级」掉　🛡️ **预防性（本次未触发，但风险真实存在）**

**症状**：在环境里装了别的库（比如 `ultralytics`）之后，ROCm 版 torch 被替换成了 PyPI 版。

**原因**：某些库的 `install_requires` 里写了较高的 `torch` 版本约束，pip 会从 PyPI 升级以满足它。

**解决**：

```powershell
# 每次装完新库，立刻复查
python -c "import torch; print(torch.__version__)"
# 必须仍为 2.13.0+rocm10.0.0

# 万一被替换，用 --force-reinstall 救回来
& "C:\Users\<用户名>\.conda\envs\rocm-torch\python.exe" -m pip install `
    --index-url https://stable.repo.amd.com/rocm/whl-next/ `
    --force-reinstall `
    "torch[device-gfx1031]" "torchvision[device-gfx1031]" torchaudio
```

---

### 4.3 C 类：安装过程 / 终端类

#### C-1 pip 看起来「卡住」不动　🔬 **本次实测**

**症状**：pip 在 `Collecting rocm-sdk-core` 阶段长时间没反应，或者下载进度条走得很慢。

**原因**：AMD 的包索引**不支持 PEP 658 元数据**，pip 必须**把 wheel 完整下载下来**才能读取依赖信息。`rocm-sdk-core` 单个体积就有 **758 MB**。

**解决**：**耐心等**。总下载量约 1.2 GB，实测网速 9–10 MB/s 时约 3 分钟；网速慢的话十几分钟都正常。

> **【注意事项】** 不要因为「看起来卡住」就 Ctrl+C 中断——中断后重新跑会重新校验已下载的缓存，更浪费时间。

---

#### C-2 ⚠️ PowerShell 误报「安装失败」（exit code 1）　🔬 **本次实测**

**症状**：pip 明明打印了 `Successfully installed ...`，但命令退出码却是 **1**，终端还飘红：

```
python.exe : WARNING: The scripts ... are installed in '...\Scripts' which is not on PATH.
    + CategoryInfo          : NotSpecified: (...) [], RemoteException
    + FullyQualifiedErrorId : NativeCommandError
```

**原因**：**PowerShell 的经典陷阱**。当原生命令（native command）向 **stderr** 写入任何内容时，PowerShell 会把它包装成 `NativeCommandError` 并设置非零退出码——**即使命令实际完全成功**。pip 的 PATH 提示、MIOpen 警告、编译器的 `warning:` 都会触发。

**解决**：**以 pip 自己的输出为准**，看有没有 `Successfully installed`。想避免误判可以：

```powershell
# 方案一：把 stderr 一起重定向，PowerShell 就不再视为错误
& "C:\...\python.exe" -m pip install ... 2>&1 | Out-String

# 方案二：显式忽略退出码，只检查输出
$out = & "C:\...\python.exe" -m pip install ... 2>&1
if ($out -match 'Successfully installed') { "✅ 安装成功" }
```

> **【教训】** 用 PowerShell 调 pip / conda / 编译器时，**不要用退出码判断成败**，要看输出内容。

---

#### C-3 `hipInfo.exe` / `rocm-sdk.exe` 命令找不到　🔬 **本次实测**

**症状**：`hipInfo` 或 `rocm-sdk` 提示「不是内部或外部命令」。

**原因**：这些工具装在环境的 `Scripts` 目录，而该目录**不在 PATH** 上（pip 安装时也会警告这一点）。

**解决**：用绝对路径，或先手动加入 PATH：

```powershell
# 绝对路径调用
& "C:\Users\<用户名>\.conda\envs\rocm-torch\Scripts\rocm-sdk.exe" targets
& "C:\Users\<用户名>\.conda\envs\rocm-torch\Scripts\hipInfo.exe"

# 或者临时加入 PATH
$env:PATH = "C:\Users\<用户名>\.conda\envs\rocm-torch\Scripts;" + $env:PATH
```

---

#### C-4 中文输出乱码　🔬 **本次实测**

**症状**：终端里中文显示成 `�����Ѵ�ͨ`。

**原因**：控制台代码页与 UTF-8 不一致。

**解决**：

```powershell
chcp 65001                              # 切换到 UTF-8 代码页
$OutputEncoding = [Console]::UTF8       # PowerShell 输出编码
```

> **【注意事项】** 这只影响**显示**，不影响功能。脚本里尽量用英文打印可以发现更省心。

---

### 4.4 D 类：运行时类

#### D-1 ⭐ `MIOpen Error: create_directories: Access is denied` → `miopenStatusUnknownError`　🔬 **本次实测（最有价值的坑）**

**症状**：`torch.cuda.is_available()` 是 `True`，矩阵乘法也正常，但一跑 `conv2d` 就崩：

```
MIOpen Error: create_directories: Access is denied.: "C:\Users\<用户名>\.miopen\db"
Traceback (most recent call last):
  ...
  File "...\torch\nn\modules\conv.py", line 560, in _conv_forward
    return F.conv2d(...)
RuntimeError: miopenStatusUnknownError
```

**原因**：**MIOpen 需要创建自己的内核调优缓存目录** `C:\Users\<用户名>\.miopen\db`。如果该目录**创建失败**（权限不足、HOME 指向只读位置、或运行在受限沙箱/服务账户下），MIOpen 初始化失败，`conv2d` 直接抛 `miopenStatusUnknownError`。

> ⚠️ **特别注意**：这个报错**极具误导性**，看起来像「卷积算子不支持」，实际是**纯目录权限问题**。因为矩阵乘法（rocBLAS）不依赖这个缓存，所以只有 `conv2d` 会炸——很容易误判成「这张卡的卷积用不了」。

**解决**：

```powershell
# 手工创建缓存目录（普通用户权限下会自动创建，无需干预）
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.miopen\db" | Out-Null

# 或者用环境变量把缓存指到别处
$env:MIOPEN_USER_DB_PATH = "D:\miopen-cache"
$env:MIOPEN_CUSTOM_CACHE_DIR = "D:\miopen-cache"
```

验证缓存目录已生成：

```powershell
Get-ChildItem "$env:USERPROFILE\.miopen"
# 应看到 cache 和 db 两个子目录
```

---

#### D-2 `MIOpen: Warning [OpenRuntimeLibraryForDevice] CK grouped conv library not found for device gfx1031`　🔬 **本次实测（无害）**

**症状**：跑 conv 相关代码时打印这条警告（或中文乱码版本）。

**原因**：AMD 没有为 `gfx1031` 单独编译 **Composable Kernel（CK）的 grouped-conv 优化库**。MIOpen 找不到它，于是**自动回退到默认实现**。

**解决**：**无需处理，这是正常的。**

> **【影响评估】** 功能完全正常，只是**部分分组卷积（grouped conv）会慢一些**。YOLO 系列、常规 CNN 训练/推理不受实质影响。

---

#### D-3 `warning: xnack 'Off' was requested for a processor that does not support it!`　🔬 **本次实测（无害）**

**症状**：编译 GPU 内核时反复打印这条。

**原因**：`XNACK`（retry-on-page-fault）是 **CDNA 数据中心卡**（MI 系列）的特性。RDNA2 消费卡**本来就不支持**。内核编译工具链在生成目标参数时会带上这个 request，然后提示不支持。

**解决**：**无需处理，这是正常的。**

---

#### D-4 ⭐ `torch.cuda.is_available()` 返回 `False`　🛡️ **排查清单（本次未遇到，GPU 一次识别成功）**

**排查顺序**（按命中率从高到低）：

| 序号 | 检查项 | 命令 / 动作 |
|---|---|---|
| 1 | **驱动是否够新** | 确认 AMD Software ≥ 26.8.1，装完整版 Adrenalin，**重启** |
| 2 | **装的是不是 ROCm 版 torch** | `python -c "import torch; print(torch.__version__)"`，必须是 `+rocm10.0.0` 结尾 |
| 3 | **device 包是否装对** | `pip list \| findstr /I "amd-torch-device rocm-sdk"`，必须看到 `gfx1031`（**不是** `gfx1030` 或别的） |
| 4 | **是否有远程桌面/虚拟显示器抢占** | 关闭向日葵 / ToDesk / UU远程 / GameViewer 等，拔掉虚拟显示器；有条件的直接在本机物理显示器前操作 |
| 5 | **是否有 HIP SDK 残留** | 见 [2.3 ③](#-清理-hip-sdk--旧-rocm-残留)，特别注意空的 `C:\Program Files\AMD\ROCm` 目录 |
| 6 | **是否被环境变量屏蔽** | `Get-ChildItem env: \| Where-Object { $_.Name -match 'HIP\|ROCM\|HSA' }`，清理异常值 |
| 7 | **打开详细日志** | `$env:AMD_SERIALIZE_KERNEL=1`、`$env:HIP_LAUNCH_BLOCKING=1` 后重跑，看真实报错位置 |

> **【经验】** 本次移植中，第 4 项（`GameViewer Virtual Display Adapter` 虚拟显示器）在系统里存在但**未造成影响**（GPU 正常识别）。它属于「值得怀疑但不一定有问题」的因素——GPU 正常时无需处理，出问题时可优先排查。

---

### 4.5 E 类：权限 / 目录类

#### E-1 ⭐ `NoWritableEnvsDirError: No writeable envs directories configured`　🔬 **本次实测**

**症状**：

```
NoWritableEnvsDirError: No writeable envs directories configured.
  - C:\Users\<用户名>\.conda\envs
  - D:\Mysoftware\anaconda3\envs
  - C:\Users\<用户名>\AppData\Local\conda\conda\envs
```

**原因**：conda 对上述**所有**候选 `envs` 目录都没有写权限。

常见触发场景：

- 运行在**受限沙箱 / 容器**里，文件系统只允许写入指定工作目录
- **企业安全软件**（如某些 EDR / 勒索防护）锁定了用户目录
- 用**低权限服务账户**运行
- **磁盘满**或目录被 ACL 限制

**解决**：

```powershell
# 1) 确认目录是否存在、能否写入
$d = "$env:USERPROFILE\.conda\envs"
New-Item -ItemType Directory -Force -Path $d | Out-Null
"test" | Out-File "$d\_wtest.tmp"          # 能写就说明权限没问题
Remove-Item "$d\_wtest.tmp"

# 2) 如果确实写不了，换一个可写位置作为 envs 目录
conda config --add envs_dirs D:\conda-envs

# 3) 或者用 -p 指定完整路径创建环境
conda create -p D:\conda-envs\rocm-torch python=3.12 -y
# 之后所有命令用 D:\conda-envs\rocm-torch\python.exe
```

> **【注意事项】**
> - 本文实测中，该错误由**受限沙箱**引起，在放宽文件系统权限后即创建成功——**这是执行环境限制，不是 conda 或驱动的问题**。
> - 用 `-p` 指定路径时，conda 不会把环境注册进 `conda env list` 的命名环境，需要用完整路径激活：`conda activate D:\conda-envs\rocm-torch`。
> - 换 `envs_dirs` 时确认目标盘**剩余空间 ≥ 15 GB**。

---

### 4.6 F 类：历史坑 📜 **来自那两篇博客的编译路线，现已不需走，但值得了解**

> 这些是「自己编译 PyTorch」路线上的坑。**按本文路线走不会遇到**，但了解它们有助于理解为什么「零编译」方案是正确的选择。

| # | 报错 / 现象 | 根本原因 | 正确做法 |
|---|---|---|---|
| F-1 | `找不到 VCToolsRedist`，手动设 `$env:VCINSTALLDIR` / `$env:VCToolsRedistDir` 后又报别的 | PyTorch 编译依赖**数十个** MSVC 环境变量（`INCLUDE` / `LIB` / `PATH`…），手工设置只能解决一个 | 用 **「x64 Native Tools Command Prompt for VS 2022」**，一次性加载完整环境 |
| F-2 | ⭐ `error: use of undeclared identifier 'Wno'` / `'error'` / `'missing'` / `'prototypes'` | 为压制警告设了 `CFLAGS="-Wno-error=missing-prototypes ..."`。该字符串被写进**自动生成的头文件**，`CAFFE2_BUILD_STRINGS` 展开后变成 `""-Wno-error=..."` —— **非法的 C++ 字符串拼接** | **绝对不要在 `CFLAGS` / `CXXFLAGS` 里塞 `-Wno-error=...`**。真需要压警告，改 CMake 配置而不是环境变量 |
| F-3 | 编译耗时数小时无果；清理 `pytorch/build` 后重来 | 中间产物缓存残留导致构建逻辑混乱 | 报错涉及生成文件（如 `RegisterMeta_0.cpp`）时，删除 `build` 目录重新构建 |
| F-4 | PyTorch 2.11 编译反复失败 | 版本太新，Windows + ROCm 路径上的适配滞后 | 降级到**已被验证的版本**（博主最终用 2.9.1 成功） |
| F-5 | 用了第三方预构建 ROCm 包（`d2awnip2yjpvqn.cloudfront.net`）后编译失败 | 第三方包版本过旧，与 PyTorch 不匹配 | **只用官方索引**：`stable.repo.amd.com` / `nightly.repo.amd.com` / `rocm.nightlies.amd.com` |
| F-6 | 命令长度超限 / 路径过长报错 | Windows 路径长度限制 | 源码放在**短路径**（如 `C:/b/pytorch`），`git clone` 加 `--depth=1`，并开启 `LongPathsEnabled` |
| F-7 | `[hipBLASLt] error: directive requires gfx90a+` / `.amdhsa_accum_offset 80` | hipBLASLt 的 `LayerNormGenerator.py` / `AMaxGenerator.py` 中的架构白名单不含 `gfx1031` | 往白名单里加 `gfx1031`（[issue #1002](https://github.com/ROCm/TheRock/issues/1002) 里社区给出的补丁）；**该问题已由 PR #1629 修复，现在无需处理** |

---

## 五、测试与验证

### 5.1 四层验证法

移植是否成功，**不能只看 `pip list`**。建议按以下四层逐级验证，任一层失败就按 [第四章](#四常见报错与解决踩坑记录) 排查。

#### 第 1 层：包装上了吗？

```powershell
& "C:\Users\<用户名>\.conda\envs\rocm-torch\python.exe" -m pip list |
    Select-String -Pattern "torch|rocm|numpy|pillow"
```

必须看到的**关键包**：

| 包 | 期望版本 |
|---|---|
| `torch` | `2.13.0+rocm10.0.0` |
| `torchvision` | `0.28.0+rocm10.0.0` |
| `torchaudio` | `2.11.0.2+rocm10.0.0` |
| `amd-torch-device-gfx1031` | `2.13.0+rocm10.0.0` ← **专属 GPU 内核，必须有** |
| `amd-torchvision-device-gfx1031` | `0.28.0+rocm10.0.0` |
| `rocm-sdk-core` | `10.0.0` |
| `rocm-sdk-libraries` | `10.0.0` |
| `rocm-sdk-device-gfx1031` | `10.0.0` |

#### 第 2 层：PyTorch 认到 GPU 了吗？

```powershell
& "C:\Users\<用户名>\.conda\envs\rocm-torch\python.exe" -c @"
import torch
print('torch      :', torch.__version__)
print('hip/rocm   :', torch.version.hip)
print('available  :', torch.cuda.is_available())
print('device cnt :', torch.cuda.device_count())
print('device 0   :', torch.cuda.get_device_name(0))
p = torch.cuda.get_device_properties(0)
print('gcnArch    :', p.gcnArchName)
print('vram       : %.1f GB' % (p.total_memory/2**30))
"@
```

#### 第 3 层：ROCm 运行时层正常吗？

```powershell
# 列出 ROCm SDK 支持的架构（确认列表里有你的 gfx target）
& "C:\Users\<用户名>\.conda\envs\rocm-torch\Scripts\rocm-sdk.exe" targets

# HIP 层直接看设备信息
& "C:\Users\<用户名>\.conda\envs\rocm-torch\Scripts\hipInfo.exe"
```

#### 第 4 层：真跑一遍 GPU 计算（**最有说服力**）

```powershell
& "C:\Users\<用户名>\.conda\envs\rocm-torch\python.exe" -c @"
import torch, torch.nn as nn, time
dev = 'cuda'

# 1) 矩阵乘法 + 算力实测
a = torch.randn(4096, 4096, device=dev); b = torch.randn(4096, 4096, device=dev)
for _ in range(3): c = a @ b          # 预热（首次含内核编译，很慢，必须丢掉）
torch.cuda.synchronize()
t0 = time.perf_counter(); n = 20
for _ in range(n): c = a @ b
torch.cuda.synchronize()
dt = (time.perf_counter() - t0) / n
print('matmul  : %.4f s/iter -> %.1f TFLOPS' % (dt, 2*4096**3/dt/1e12))

# 2) conv2d（CNN / YOLO 的核心算子）
x = torch.randn(8, 3, 640, 640, device=dev)
conv = nn.Conv2d(3, 32, 3, padding=1).to(dev)
with torch.no_grad(): y = conv(x)
print('conv2d  :', tuple(x.shape), '->', tuple(y.shape), 'OK')

# 3) 完整训练一步：前向 + 反向 + 优化器
net = nn.Sequential(nn.Conv2d(3,16,3,padding=1), nn.ReLU(),
                    nn.AdaptiveAvgPool2d(1), nn.Flatten(), nn.Linear(16,10)).to(dev)
opt = torch.optim.SGD(net.parameters(), lr=0.01)
lossf = nn.CrossEntropyLoss()
xb = torch.randn(32,3,64,64, device=dev); yb = torch.randint(0,10,(32,), device=dev)
for _ in range(5):
    opt.zero_grad(); lossf(net(xb), yb).backward(); opt.step()
L = []
for _ in range(60):
    opt.zero_grad(); l = lossf(net(xb), yb); l.backward(); opt.step(); L.append(l.item())
print('train   : loss %.4f -> %.4f' % (L[0], L[-1]))

# 4) 混合精度（A 卡炼丹常用）
with torch.autocast('cuda', dtype=torch.float16):
    o = net(xb)
print('fp16    :', tuple(o.shape), 'OK')

free, total = torch.cuda.mem_get_info()
print('vram    : free %.2f / total %.2f GB' % (free/2**30, total/2**30))
print('ALL OK - GPU 加速已打通')
"@
```

> **【注意事项】**
> - **必须做预热**（`for _ in range(3)`）。首次运算包含 GPU 内核编译，耗时会高一个数量级，直接计入会得出错误的算力结论。
> - 首次跑 `conv2d` 会创建 MIOpen 缓存目录，若报错见 [D-1](#d-1--miopen-error-create_directories-access-is-denied--miopenstatusunknownerror)。

### 5.2 本次移植的实测结果

| 验证项 | 实测结果 | 判定 |
|---|---|---|
| Python 版本 | 3.12.14 | ✅ |
| `torch.__version__` | `2.13.0+rocm10.0.0` | ✅ 是 ROCm 版，不是 CUDA/CPU 版 |
| `torch.version.hip` | `7.15.26333` | ✅ HIP 运行时已挂载 |
| `torch.cuda.is_available()` | `True` | ✅ **核心指标** |
| `torch.cuda.device_count()` | `1` | ✅ |
| 设备名 | `AMD Radeon RX 6750 GRE 12GB` | ✅ 型号识别正确 |
| `gcnArchName` | **`gfx1031`** | ✅ **架构识别与预期完全一致** |
| 显存 | 12.0 GB（`hipInfo` 报 11.98 GB） | ✅ |
| 矩阵乘法 4096³ | **0.0129 s/iter → 10.6 TFLOPS** | ✅ 见下方分析 |
| `conv2d` (8,3,640,640)→(8,32,640,640) | 正常 | ✅ CNN 算子可用 |
| 训练循环（60 步） | loss `2.4124 → 2.3525` | ✅ **反向传播 + 优化器正常** |
| `autocast` FP16 | 正常输出 (32,10) | ✅ 混合精度可用 |
| 显存占用 | free 11.15 / total 11.98 GB | ✅ |
| `rocm-sdk targets` | 输出中含 `gfx1031` | ✅ |
| `hipInfo.exe` | 正常列出设备（时钟 2439 MHz、位宽 192 bit） | ✅ |
| 环境体积 | 3.65 GB | — |

**关于 10.6 TFLOPS 这个数字**：

RX 6750 GRE 12GB（Navi 22，2560 流处理器，核心频率 2439 MHz）的 FP32 理论峰值约为 `2560 × 2 × 2.439 GHz ≈ 12.5 TFLOPS`。实测 10.6 TFLOPS ≈ **理论峰值的 85%**。

> **【判断要点】** 这个数字证明了 **计算确实跑在 GPU 上**，而不是静默回退到 CPU（CPU 跑 4096³ 矩阵乘法是秒级，不可能 0.0129 秒）。
> ⚠️ 注意 `hipInfo` 报的 `multiProcessorCount: 20`，**该字段的统计口径与 Windows 侧工具（如 GPU-Z）显示的核心数不同，不要用它来判断移植是否正常**。以**实测 TFLOPS** 和 `gcnArchName` 为准。

> **[此处需补充：与同价位 NVIDIA 显卡（如 RTX 4060 / 3060 12G）在相同模型训练任务上的实测对比数据]**

---

## 六、总结与展望

### 6.1 本次移植的优点

| 优点 | 说明 |
|---|---|
| **零编译** | 不需要 Visual Studio、不需要拉 PyTorch 源码、不需要编译几小时 |
| **一条命令** | `pip install --index-url ... "torch[device-gfx1031]" ...` 即可完成 |
| **官方通道** | 走 AMD 的 TheRock / `repo.amd.com` 索引，不是第三方魔改包 |
| **可控版本** | stable / nightly 双通道，稳定版与最新版可自由选择 |
| **架构自动匹配** | `[device-*]` 扩展机制只下载**你显卡专属**的 GPU 内核，无需装全架构包（下载量从数百 MB 降到几十 MB） |
| **环境隔离** | 独立的 conda 环境，不与现有 CUDA 环境冲突，可随时删除重来 |

### 6.2 本次移植的局限与不足

| 局限 | 说明与影响 |
|---|---|
| **不在官方兼容性矩阵内** | `gfx1031` 属于 TheRock 通道支持，而非 AMD 正式 release 支持。意味着**没有官方 SLA / 技术支持**，遇到问题需自行提 issue |
| **Triton 在 Windows 不可用** | [triton-windows#2](https://github.com/triton-lang/triton-windows/issues/2) 仍在进行中。因此 **`torch.compile`（inductor 后端）无法使用**，只能跑 eager 模式。对训练性能有实质影响 |
| **无 CK 分组卷积优化库** | `gfx1031` 缺少 Composable Kernel 的 grouped-conv 库，部分分组卷积会回退到较慢的实现 |
| **索引不支持 PEP 658** | pip 必须下载完整 wheel 才能解析依赖，**安装过程偏慢、无法「只解析不下载」做 dry-run** |
| **纯 Python 3.10 用户需绕路** | 索引缺 cp310 的 numpy，Python 3.10 环境需额外两步（见 [3.4](#34-可选python-310-的两步法绕开-numpy-缺包)） |
| **环境体积较大** | 3.65 GB（`rocm-sdk-core` 一个包就 758 MB） |
| **仅验证了单卡单任务** | 本次只验证了 `gfx1031` + 基础算子 + 一个小型训练循环；**多卡、分布式、长时间训练的稳定性未验证** |
| **性能未做深度调优** | 未测 `MIOPEN_FIND_MODE`、`HSA_OVERRIDE_GFX_VERSION`、显存分配策略等调优项 |

> **[此处需补充：长时间训练（数小时以上）的温度、功耗、稳定性表现]**
> **[此处需补充：实际业务模型（如 YOLOv5 完整训练）在本配置下的收敛情况与耗时]**

### 6.3 后续可优化的方向

**短期（立刻可做）**

1. **把验证脚本固化下来**，做成一个 `check-rocm.py`，每次装完新库或升级后跑一遍，快速确认 ROCm 栈没被破坏。
2. **给环境加「防污染」约束文件**：把 torch 三件套的精确版本写进 `constraints.txt`，之后安装其他库时统一用 `pip install -c constraints.txt xxx`，防止 torch 被 PyPI 版顶掉。
3. **备份环境**：`conda create -n rocm-torch-bak --clone rocm-torch`，或记录完整的 `pip freeze` 输出，便于快速回滚。
4. **在环境里补装下游依赖**：如 `ultralytics`、`opencv-python`、`matplotlib` 等（**注意装完复查 torch 版本**，见 [B-3](#b-3-依赖包被别的库顺手升级掉)）。

**中期（值得研究）**

5. **跟进 Triton on Windows 的进展**，一旦可用即可启用 `torch.compile`，训练速度有望明显提升。
6. **MIOpen 调优**：尝试 `MIOPEN_FIND_MODE=FAST` / `NORMAL`，或用 `MIOPEN_FIND_ENFORCE` 做卷积算法自动寻优，把 grouped conv 的性能找回来。
7. **尝试 nightly 通道**（ROCm 10.2.0a + torch 2.15）看新版本是否带来性能改进，但**不要用在生产任务上**。
8. **对比 ZLUDA / DirectML 等替代方案**，评估在 YOLO / SD 类任务上的性能差异，形成选型建议。

**长期（方向性）**

9. **验证其他 gfx target 的通用性**：把本文流程在 `gfx1030` / `gfx1032` / `gfx1100` 等卡上复现，确认 `[device-*]` 机制是否普适，形成一张「显卡 → 可用性」对照表。
10. **建立版本兼容矩阵**：持续记录「ROCm 版本 ↔ torch 版本 ↔ 驱动版本 ↔ Python 版本」的可用组合，避免后续升级踩坑。
11. **关注 `gfx1031` 是否进入官方兼容性矩阵**：一旦进入正式 release 支持，就能获得官方 SLA。
12. **考虑 Linux 双系统方案**：ROCm 在 Linux 上的成熟度和性能通常优于 Windows（`gfx1031` 在 Linux 上甚至只需一个环境变量就能跑），对性能敏感的场景值得评估。

> **[此处需补充：本次移植经验在其他型号 AMD 显卡上的复现结果]**

### 6.4 一句话总结

> **2026 年 2 月还需要「装 VS + 编译几小时 + 打源码补丁」的 Windows + AMD 炼丹，现在只需要一条 `pip install`。**
>
> 核心变化是 AMD 的 TheRock 项目转向了**多架构预编译发布**，并让 `[device-<gfx>]` 扩展机制只下载你显卡专属的 GPU 内核。
>
> 移植成功的三个关键点：**① 别编译；② 用 Python 3.12+；③ 别装 HIP SDK。**

---

## 附录：参考资料

### 官方文档与索引

| 资源 | 链接 |
|---|---|
| TheRock 发布说明（多架构 pip 安装 + GFX 对照表） | https://github.com/ROCm/TheRock/blob/main/RELEASES.md |
| TheRock GPU 支持状态矩阵 | https://github.com/ROCm/TheRock/blob/main/SUPPORTED_GPUS.md |
| TheRock PyTorch 构建说明 | https://github.com/ROCm/TheRock/tree/main/external-builds/pytorch |
| ROCm 10.0.0 安装页（**选择器驱动，记得改 URL 参数**） | https://rocm.docs.amd.com/en/docs-10.0.0/install/rocm.html |
| ROCm 10.0.0 兼容性矩阵 | https://rocm.docs.amd.com/en/docs-10.0.0/compatibility/compatibility-matrix.html |
| AMD 官方 GPU 架构规格表（gfx 代号对照） | https://rocm.docs.amd.com/en/latest/reference/gpu-arch-specs.html |
| AMD AI Ecosystem — Install PyTorch for ROCm | https://rocm.docs.amd.com/projects/ai-ecosystem/en/latest/frameworks/pytorch/install.html |
| AMD Software Adrenalin 26.8.1 发布说明 | https://www.amd.com/en/resources/support-articles/release-notes/RN-RAD-WIN-26-8-1.html |

### 包索引（pip 源）

| 通道 | 索引 URL |
|---|---|
| **稳定通道（推荐）** | `https://stable.repo.amd.com/rocm/whl-next/` |
| 每日构建（可能不稳定） | `https://nightly.repo.amd.com/rocm/whl-next/` |
| 预发布 | `https://rc.repo.amd.com/rocm/whl-next/` |
| 历史归档（per-family 旧索引） | https://rocm.nightlies.amd.com/v2/ |

### 社区资料

| 资源 | 链接 |
|---|---|
| TheRock issue #1002 — Windows 下 gfx1031 构建讨论（**已关闭**） | https://github.com/ROCm/TheRock/issues/1002 |
| Triton on Windows 进度 | https://github.com/triton-lang/triton-windows/issues/2 |
| Mesa `amd_family.c`（查 Navi 代号 → gfx target） | https://gitlab.freedesktop.org/mesa/mesa/-/blob/main/src/amd/common/amd_family.c |
| 曦远Code：Windows + AMD ROCm + PyTorch 折腾经历（**第一篇，编译失败**） | https://www.cnblogs.com/deali/p/19580146 |
| 曦远Code：Windows + AMD 显卡，终于能用 PyTorch 炼丹了（**第二篇，编译成功**） | https://www.cnblogs.com/deali/p/19584890 |

---

## 修订记录

| 日期 | 版本 | 说明 |
|---|---|---|
| [此处需补充：文档日期] | v1.0 | 基于 Windows 11 25H2 + RX 6750 GRE 12GB（gfx1031）+ ROCm 10.0.0 + PyTorch 2.13.0 实测整理 |

> **【免责声明】** 本文档所有命令与结论均基于上表中的实测环境。AMD 的包索引内容会随 nightly 构建变动，**执行前请以官方文档与索引实际内容为准**。文中标注 `[此处需补充...]` 的部分为本次未采集到的数据，**请勿直接沿用**。

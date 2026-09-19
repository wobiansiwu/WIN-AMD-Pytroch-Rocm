# AMD 显卡 + Windows 跑 PyTorch：移植前确认清单

> **配套文档**：`Windows-AMD-ROCm-PyTorch-移植教学.md`（完整原理与踩坑记录）
>
> **本文用途**：移植前花 2 分钟自查这 **6 项**；如果要把机器交给别人协助，直接把 [第二章](#二一键收集脚本) 的输出贴过去即可。

---

## 一、6 项必查

| # | 确认项 | 合格标准 | 不合格的后果 |
|---|---|---|---|
| 1 | **GPU 架构代号（gfx target）** | 必须查到准确值，如 `gfx1031` | 决定 `[device-???]` 参数，**填错必定失败** |
| 2 | **Windows 版本** | Windows 11 **25H2**（build ≥ 26200） | 官方兼容性矩阵外的版本无保证 |
| 3 | **显卡驱动** | AMD Software **Adrenalin 26.8.1** 或更新，且为**完整版** | 太旧 → `torch.cuda.is_available()` 恒为 `False` |
| 4 | **Python 版本** | **≥ 3.10**，强烈建议 **3.12 / 3.13 / 3.14** | 3.10 需额外绕开 `numpy` 缺包 |
| 5 | **HIP SDK / 旧 ROCm 残留** | **必须完全清除** | 有残留 → 运行期报错，需先卸载 |
| 6 | **目标盘可用空间** | **≥ 15 GB** | 环境实测 3.65 GB + pip 缓存约 2 GB |

> **最关键的是第 1 项**。架构代号不能靠猜，也不能用商品名代替——`RX 6750 GRE 12GB` 在官方文档里写作 `RX 6750 XT / 6700 XT`，对应 `gfx1031`。

### gfx 代号速查（常见卡）

| 显卡 | gfx target | device 参数 |
|---|---|---|
| RX 9070 / 9070 XT | `gfx1201` | `device-gfx1201` |
| RX 9060 / 9060 XT | `gfx1200` | `device-gfx1200` |
| RX 7900 XTX / XT | `gfx1100` | `device-gfx1100` |
| RX 7800 XT / 7700 XT | `gfx1101` | `device-gfx1101` |
| RX 7600 | `gfx1102` | `device-gfx1102` |
| **RX 6750 GRE / 6750 XT / 6700 XT** | **`gfx1031`** | **`device-gfx1031`** |
| RX 6900 XT / 6800 XT | `gfx1030` | `device-gfx1030` |
| RX 6600 XT / 6600 | `gfx1032` | `device-gfx1032` |
| RX 6500 XT | `gfx1034` | `device-gfx1034` |
| Ryzen AI Max+ 395 核显 | `gfx1151` | `device-gfx1151` |

**查不到自己的卡？** 用 [TheRock GFX 对照表](https://github.com/ROCm/TheRock/blob/main/RELEASES.md#supported-python-device--install-extras) 或 [AMD 官方架构规格表](https://rocm.docs.amd.com/en/latest/reference/gpu-arch-specs.html)。

---

## 二、一键收集脚本

把下面内容存成 `collect_env.py`，用**任意** Python 跑一遍；或直接复制整段到 PowerShell 里执行。

<details>
<summary>展开脚本（点击）</summary>

```python
import os, shutil, subprocess, winreg

print("=" * 46)
print(" AMD + Windows PyTorch 移植 - 环境信息收集")
print("=" * 46)

# ── 1. 操作系统 ──
try:
    with winreg.OpenKey(winreg.HKEY_LOCAL_MACHINE,
                        r"SOFTWARE\Microsoft\Windows NT\CurrentVersion") as h:
        def q(n):
            try: return winreg.QueryValueEx(h, n)[0]
            except Exception: return "?"
        build = int(q("CurrentBuildNumber") or 0)
        # 注意：Win11 的 ProductName 注册表值仍写 "Windows 10"，需按 build 判断
        name = "Windows 11" if build >= 22000 else "Windows 10"
        print("OS          : %s %s (build %s)  %s" % (
            name, q("DisplayVersion"), build,
            "OK" if build >= 26200 else "!! 需 25H2 / build>=26200"))
except Exception as e:
    print("OS          : ERR", e)

# ── 2. 显卡与驱动 ──
print("--- GPU / driver ---")
k = r"SYSTEM\CurrentControlSet\Control\Class\{4d36e968-e325-11ce-bfc1-08002be10318}"
try:
    with winreg.OpenKey(winreg.HKEY_LOCAL_MACHINE, k) as h:
        i = 0
        while True:
            try: sub = winreg.EnumKey(h, i); i += 1
            except OSError: break
            try:
                with winreg.OpenKey(h, sub) as s:
                    print("  %-42s driver %s" % (
                        winreg.QueryValueEx(s, "DriverDesc")[0],
                        winreg.QueryValueEx(s, "DriverVersion")[0]))
            except Exception: pass
except Exception as e:
    print("  ERR", e)

# ── 3. AMD Software 版本（官方要求 >= 26.8.1）──
print("--- AMD Software ---")
for root in (r"SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall",
             r"SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall"):
    try:
        with winreg.OpenKey(winreg.HKEY_LOCAL_MACHINE, root) as h:
            i = 0
            while True:
                try: sub = winreg.EnumKey(h, i); i += 1
                except OSError: break
                try:
                    with winreg.OpenKey(h, sub) as s:
                        n = winreg.QueryValueEx(s, "DisplayName")[0]
                        if "AMD Software" in n:
                            print("  %s = %s" % (n, winreg.QueryValueEx(s, "DisplayVersion")[0]))
                except Exception: pass
    except Exception: pass

# ── 4. 磁盘空间（目标盘需 >= 15 GB）──
print("--- disk free ---")
for d in "CDEFGH":
    p = d + ":\\"
    if os.path.exists(p):
        t, _, f = shutil.disk_usage(p)
        print("  %s free %.1f GB / total %.1f GB" % (p, f / 2**30, t / 2**30))

# ── 5. Python 环境 ──
print("--- python ---")
for c in (["py", "-0p"], ["python", "-V"], ["conda", "env", "list"]):
    try:
        r = subprocess.run(c, capture_output=True, text=True, timeout=25,
                           shell=c[0] in ("py", "conda"))
        print("  $ %s" % " ".join(c))
        for l in (r.stdout + r.stderr).strip().splitlines():
            print("    " + l)
    except Exception as e:
        print("  $ %s -> %s" % (" ".join(c), e))

# ── 6. HIP SDK / ROCm 残留（必须为空）──
print("--- HIP / ROCm residue ---")
bad = False
for p in (r"C:\Program Files\AMD\ROCm", r"C:\Program Files (x86)\AMD\ROCm"):
    if os.path.exists(p):
        print("  !! 残留目录:", p); bad = True
hits = [p for p in os.environ.get("PATH", "").split(os.pathsep)
        if "rocm" in p.lower() or "hip" in p.lower()]
if hits: print("  !! PATH 残留:", hits); bad = True
ev = [k for k in os.environ if any(x in k.upper() for x in ("HIP", "ROCM", "HSA"))]
if ev: print("  !! 环境变量残留:", ev); bad = True
if not bad: print("  OK - 无残留")

# ── 7. 若当前 python 已装 torch，顺便报状态 ──
print("--- torch (当前 python) ---")
try:
    r = subprocess.run(["python", "-c",
        "import torch;print('version:',torch.__version__);"
        "print('hip:',torch.version.hip);"
        "print('available:',torch.cuda.is_available())"],
        capture_output=True, text=True, timeout=180)
    out = (r.stdout + r.stderr).strip()
    if "ModuleNotFoundError" in out:
        print("  (当前 python 未安装 torch)")
    else:
        print("  " + out.replace("\n", "\n  "))
except Exception:
    print("  (未安装)")

print("=" * 46)
print("请把以上全部内容复制给协助者")
```

</details>

> **提示**：脚本第 7 段首次 `import torch` 可能耗时 10–30 秒（加载 ROCm DLL），属正常现象。

---

## 三、交给协助者时，请一并提供

| 必提供 | 说明 |
|---|---|
| 1. **上面脚本的完整输出**（文字，非截图） | 一次性覆盖全部 6 项 |
| 2. **完整报错文本** | 从第一行到最后一行，**不要只截一半**，尤其不要漏掉 Traceback 的开头 |
| 3. **已执行过的命令** | 便于判断是否已造成环境状态改变 |
| 4. **你想要的用途** | 训练 / 推理 / ComfyUI / YOLO…，会影响版本与依赖建议 |
| 5. **[可选] 是否愿意新建独立环境** | 若必须用现有环境，需说明其 Python 版本与已装关键包 |

---

## 四、信息缺失最常导致的 4 类返工

| 缺失项 | 后果 |
|---|---|
| 没提供 **gfx 代号** | 只能装错 device 包 → `is_available()` 为 `False` |
| 没提供 **AMD Software 版本** | 装完才发现驱动不达标，需回退重来 |
| 没提供 **Python 版本** | Python 3.10 会卡在 `numpy` 解析，白跑一趟 |
| 没检查 **HIP SDK 残留** | 装完运行报错，需卸载重装 |

---

**一句话**：**gfx 代号、Windows build、Adrenalin 版本、Python 版本、HIP 残留、磁盘空间** —— 这 6 项齐了，移植基本一次成功。

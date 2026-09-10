# 变更记录

本文件记录本整合包相对于上游项目所做的修改，以履行
Apache License 2.0 第 4(b) 条的修改声明义务。

---

## 启动体验优化（2026-09-05）

- `启动.bat` 直接打开启动器，由 Electron 负责单实例和重复启动时的窗口激活，省去 PowerShell 启动与窗口辅助类编译。
- 启动器直接检查自己持有的 Python 子进程，不再反复运行 PowerShell 枚举全机进程；页面服务检测间隔缩短到 250 毫秒。
- 端口消息改为按行缓冲解析，支持消息拆分或与日志合并，移除 Python 启动路径上固定的一秒等待。
- 功能页面先显示，页面打开后再在后台准备本地语音引擎，避免 PyTorch 和 VoxCPM 的依赖导入阻塞首屏。
- 页面显示本地配音的准备状态；准备期间可编辑文本、使用云端功能，提前提交的本地任务会自动等待。重复打开页面不会重复初始化。
- 状态栏不再为了检测 CUDA 提前加载 PyTorch；初始化失败时保留功能页面并显示提示。
- 启动器不再对页面加载设 5 秒硬超时。原逻辑在 iframe 5 秒内没加载完就把它从页面上撤下、退回端口探测重来，慢机器或首次无磁盘缓存时会反复打转，表现为长时间白屏；现改为超过 3 秒先把页面本身显示出来，只有 20 秒仍未加载完才重试，且最多两次。
- 本次修改：`win-unpacked/python/webui.py`、`win-unpacked/python/system_status.py`、`win-unpacked/resources/app.asar`（启动器渲染端 `out/renderer`）。语音模型权重仍在首次本地生成时加载。

## v2.15.0 · VoxCPM2 Plus（2026-08-30）

### 相对于 OpenBMB/VoxCPM 上游（commit `856d2fc2a853656e324e491706d1e8a6bfac361c`）

**已修改的文件**（各文件头部均带有 Apache 4(b) 修改声明）：

| 文件 | 修改内容 |
| --- | --- |
| `win-unpacked/python/src/voxcpm/core.py` | 从整合包内置的模型目录加载 ZipEnhancer 降噪模型，不在运行时联网下载 |
| `win-unpacked/python/src/voxcpm/zipenhancer.py` | ZipEnhancer 模型路径默认指向本地内置模型目录 |

**已移除的文件：**

| 文件 | 原因 |
| --- | --- |
| `app.py` | 移除上游示例入口，改用整合包自有入口 |

### 相对于余子越原整合包

**功能改动：**

- 音色管理
- 任务队列并发处理
- 长文本生成的**段间响度对齐**——原实现每段独立生成、拼接前无电平对齐，
  实测段间响度极差 5.5 dB（标准差 1.5 dB），修复后显著收敛。
  此问题此前长期被误判为「长音频音色漂移」，实为音量突变造成的听感错觉。
- 明暗主题适配
- 云端语音合成（MiniMax）接入与交互
- 若干并发与稳定性修复

### 第三方组件调整（2026-08-30，许可证收口）

| 组件 | 变更 | 原因 |
| --- | --- | --- |
| FFmpeg | 由 BtbN `win64-gpl-shared`（`N-122277-g78bfcf003b`）换为 `win64-lgpl-shared`（`n8.1.2-50-g1a748fe2cd`） | 原构建启用 `--enable-gpl`，含 libx264 / libx265 / librubberband 等 GPL-only 组件，随包分发需提供完整对应源码 |
| Rubber Band 4.0.0 | 移除 | GPL-2.0-or-later；变速改由 `audiotsm` 的 WSOLA 承担 |
| `kaldiio` 2.18.1 | 移除 | 附带 NTT 评估用途许可协议，不允许再分发 |
| `funasr` 1.3.9 | 修改：`load_utils.py`、`models/eend/eend_ola_dataloader.py`、`utils/compute_det_ctc.py` 三处的模块级 `import kaldiio` 改为惰性导入 | 配合上一项。`kaldiio` 仅在 `kaldi_ark` 数据类型下被调用，本整合包不使用该类型，功能不受影响。如需该功能可自行 `pip install kaldiio` |

**已知影响：** 移除 Rubber Band 后，减速到 0.8× 左右可能出现轻微电流音；
加速（1.25×）表现正常。详见 README「关于语速调节」。

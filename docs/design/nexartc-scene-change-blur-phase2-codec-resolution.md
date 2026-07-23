# nexartc 场景切换模糊 Phase 2：残余模糊分析 + 编码格式/分辨率可配置

> 日期：2026-07-23（23:53 补充真机日志验证，见 §8/§9）
>
> 状态：前台监测链路已真机验证生效（0 PLI、IDR 延迟 14~38ms）；Phase 2 新增改动（H.265 QP 地板、boost 分辨率缩放、Web 设置页）已编译通过，待部署回归
>
> 前置文档：
> - [`nexartc-cae-scene-change-blur-qp-design.md`](./nexartc-cae-scene-change-blur-qp-design.md)（Phase 0/1：QP 地板、REMB 下限、场景 boost + IDR）
> - [`nexartc-web-cae-h265-support-design.md`](./nexartc-web-cae-h265-support-design.md)（H.265 协商与共享编码器锁）

---

## 1. 结论摘要

**问题**：App 快速切回桌面时画面模糊。H.265 + Phase 0/1（QP 地板 30/34、REMB 1.5M 下限、场景切换 boost 3.5M + IDR）后明显改善，但仍有残余模糊。

**是否需要提高码率？——不需要提高稳态码率，需要的是三件事：**

1. **修复前台监测阻断点**（本轮已修）：root `app_process` 中 `ActivityThread.currentApplication()` 返回 `null`，导致前台任务监测线程从未启动。系统级 Home/App 切换（用户点手机屏内导航栏或手势）完全没有触发 boost + IDR，只能等浏览器 PLI（滞后 4~9 秒）。这是残余模糊的**主要原因**，不是码率不够。
2. **H.265 专用 QP 地板**（本轮已加）：现有 `qp_i_max=30 / qp_p_max=34` 按 H.264 调优。HEVC 同 QP 主观质量≈H.264，但同码率下可以负担**更低的 QP**；沿用 H.264 的上限等于浪费了 HEVC 的压缩效率。新增 `qp_i_max_h265=28 / qp_p_max_h265=32`，用效率换清晰度而不是加码率。
3. **boost 窗口码率上调，稳态不动**（2026-07-24 00:11 更新）：真机日志（§9.1）发现 REMB 稳态爬到 3.6~4.4M 后，boost 封顶 3.5M 低于 REMB，boost 退化为"仅 IDR、零加码"。修复分两步：(a) boost 目标加相对余量下限 **REMB×1.35**；(b) 绝对封顶 `scene_change_boost_bitrate` 3.5M→**4.5M**。仍受 2×REMB 与 `remb_max=6M` 约束、只持续 1.5s、有 2s REMB 恢复地板兜底——与上一轮"固定 5M 冲高翻车"的区别是额外码率**相对、短窗口、有恢复保护**。稳态码率（`override_bitrate`/`remb_max`）保持不动。

**编码格式与分辨率可配置**（本轮已实现）：Web 设置页新增 Video Codec（Auto/H.264/H.265）与 Resolution（仅显示 `720p` / `480p` / `360p` 标签，不显示具体宽高）。分辨率映射在协议层携带：720p→720×1560、480p→536×1160、360p→360×780。

---

## 2. 现状回顾（Phase 0/1 已实现，本文档撰写时均已在 CAE 生效）

| 机制 | 位置 | 状态 |
|------|------|------|
| QP 质量地板（I≤30 / P≤34，旧配置 `-1` 自动迁移） | `CaeConfigManage::GetQpIMax/GetQpPMax` + `ScreenCapture.applyQpBounds` | ✅ 已部署验证 |
| REMB 下限 1.5M / 初始 2.5M | `CaeConfigManage` | ✅ |
| 场景切换 boost（1500ms、2.0×、封顶 3.5M）+ IDR + 2s REMB 恢复地板 | `WebRtcServerTransport::NotifySceneChange` / `QualityBoostLoop` | ✅ |
| 触发源 1：浏览器 PLI | `SetupMediaTracks` PLI handler → `NotifySceneChange("pli")` | ✅ |
| 触发源 2：客户端 Home/Back/Recent 按键消息 | `CaeConnectionAgent::NotifySceneChangeForKeyEvent`（keycode 3/4/187） | ✅（仅覆盖网页工具栏按钮） |
| 触发源 3：视频管道丢帧恢复 | `SendVideoDataHook` → `NotifySceneChange("video_pipe_drop")` | ✅ |
| 触发源 4：Android 前台任务变化监测 | `ScreenCapture.startForegroundMonitor`（100ms 轮询 top activity） | ❌ **未生效 → 本轮已修**（见 §3.1） |
| `iframe_interval=3` | `CaeConfig.ini` | ✅ |

## 3. 残余模糊根因分析

### 3.1 主因：前台监测线程从未启动（本轮修复）

真机日志证据（上一轮联调）：

- 两次系统 Home（`21:12:50.369`、`21:13:01.054`）CAE 均无 `quality boost reason=home_key/foreground_task_change` 日志；
- 关键帧分别延迟 ~4.5s / ~8.9s 才出现（等到 PLI 才触发）；
- 启动日志有 `Foreground monitor unavailable`，根因是 root `app_process` 中 `ActivityThread.currentApplication()==null`，`ActivityManager` 拿不到。

**修复**（`ScreenCapture.java`）：

1. 新增 `getSystemContextFallback()`：优先 `ActivityThread.currentActivityThread()`，取不到再 `ActivityThread.systemMain()`，然后 `getSystemContext()` 取系统 Context——与既有 VirtualDisplay "method 1d" 同一条已验证路径；
2. `startForegroundMonitor()` 在 `getAppContext()` 为 null 时改用系统 Context 获取 `ActivityManager`；仍失败才回退 `ActivityTaskManager.getService()` 反射链。

**验收日志**（重新部署后应出现）：

```text
Foreground monitor using system context (no Application in app_process)
Foreground monitor started: activity=...
Foreground scene changed: reason=home_key from=<app> to=<launcher>
WebRTC quality boost: reason=home_key target=3500 kbps ...
```

### 3.2 次因：H.265 沿用 H.264 的 QP 上限（本轮修复）

模糊的直接机制是"低码率窗口内编码器抬 QP"。HEVC 在同 QP 下细节保留与 H.264 相近甚至更好，但**同码率能负担更低的 QP**。沿用 30/34 意味着 REMB 收紧时 HEVC 仍被允许编到主观可见的糊度。

**修复**：

- `CaeConfigManage::GetQpIMaxH265()/GetQpPMaxH265()`：默认 **28/32**，配置键 `qp_i_max_h265` / `qp_p_max_h265`；`enable_qp_quality_floor=0` 时回退无限制；
- `MediaEngine::OpenVideo()` 按 `codecName==video/hevc` 选择 H.265 或 H.264 的 QP 上限传入 `setQualityConfig`；
- MediaFormat `KEY_VIDEO_QP_*` 对 HEVC 同样有效（API 28+ 标准键）。

**风险与回调**：QP 上限更严 → IDR 略大。已有 boost 窗口 + NACK 吸收；若真机出现 NACK/丢包上升，先放宽 `qp_p_max_h265` 到 34，不要回退到 `-1`。

### 3.3 关于"直接提高码率"的判定

| 选项 | 判定 | 理由 |
|------|------|------|
| 提高 `override_bitrate`（稳态 CBR） | ❌ 不建议 | 稳态桌面静止场景不缺码率；上行带宽浪费，弱网更卡 |
| 提高 `remb_max_bitrate` | ❌ 无效 | 瓶颈不在上限（6M 从未打满） |
| 提高 `remb_min_bitrate` (>1.5M) | ⏸ 观望 | 1.5M 已保底；再抬影响弱网。仅当日志显示 REMB 长期贴 1.5M 且模糊同窗出现才考虑 2M |
| 提高 `scene_change_boost_bitrate` (3.5→4.5M) | ✅ 已做 | 只影响 1.5s 窗口，是"提高码率"的正确位置；配合相对余量下限一起生效 |
| boost 相对余量（REMB×1.35 下限，本轮已加） | ✅ 已做 | 真机日志发现 REMB 超过 3.5M 封顶后 boost 退化为"仅 IDR、零加码"；改为保证切换窗口至少 REMB×1.35（见 §8.2 步骤 8/9） |
| H.265 专用 QP 地板 | ✅ 本轮已做 | 用压缩效率换清晰度，零带宽成本 |
| 修复前台监测 | ✅ 本轮已做 | 让 boost 在切换瞬间（而非 PLI 滞后数秒）生效 |

**推荐验证顺序**：先回归 §3.1 + §3.2 的效果；仍不满意再把 `scene_change_boost_bitrate=4000000` 单变量试验。

---

## 4. 编码格式 / 分辨率可配置设计（本轮已实现）

### 4.1 Web 设置页（`nexartc-cloudPhoneAccess-web/device`）

`index.html` 设置面板新增两项：

| 设置项 | 选项 | 默认 | 存储 |
|--------|------|------|------|
| Video Codec | Auto (Best Available) / H.264 / H.265 | H.264 | `localStorage: cloudphone.device.videoCodec`，URL `?codec=` 优先 |
| Resolution | `720p` / `480p` / `360p`（**仅显示标签，不显示宽高**） | 720p | `localStorage: cloudphone.device.resolution`，URL `?resolution=` 优先 |

- Codec 选择接入既有 `getCodecPreference()`：URL 参数 > 设置页 > 默认 h264。`auto` 时浏览器支持 H.265 则请求 H.265，否则 H.264；沿用既有能力探测、`0x1105` 共享 codec 冲突提示与 fallback 逻辑，无协议变更。
- Resolution 标签在 `main.ts` 内映射为具体尺寸（`RESOLUTION_PRESETS`），UI 不暴露：

| 标签 | 编码分辨率 | 像素比(vs 720p) |
|------|-----------|-----------------|
| 720p | 720×1560 | 1.00 |
| 480p | 536×1160 | 0.55 |
| 360p | 360×780 | 0.25 |

### 4.2 协议与 CAE

- `protocol.ts makeStartMsg()`：`media_config` 追加 `stream_width` / `stream_height`；
- CAE `ParseMediaConfig()` **已支持** `stream_width/stream_height`（`CaeMediaConfig::SetStreamWidth/SetStreamHeight`），无需改动；
- `BuildVideoJsonConfig()` 将请求值 clamp 到面板对齐尺寸（≤1080×2336），编码器在 `OpenMediaStream` 按 `GetStreamWidth/Height` 打开；
- START_SUCCESS 响应回传实际 `stream_width/stream_height`，Web 触控参考分辨率自动跟随，无需触控层改动；
- 三档预设均为 4 的倍数，MediaCodec 对齐无风险（1560 与 780 非 16 对齐但与现网 720×1560 同性质，已验证可用）。

### 4.3 分辨率与码率联动

- 场景 boost 按像素比缩放（本轮已实现，`WebRtcServerTransport::NotifySceneChange`）：`configuredBoostBps × clamp(pixels/720p_pixels, 0.25, 1.0)`，360p 时 boost ≈ 875kbps，避免小分辨率被灌大码率；
- 稳态码率仍由浏览器 REMB 闭环自适应——小分辨率编码复杂度低，REMB 自然收敛到更低目标，不需要服务端另做 ladder；
- 后续可选（未实现，观望）：`remb_min_bitrate` 按像素比缩放（360p 时 1.5M 下限偏高，弱网场景可降到 ~600k）。

### 4.4 多客户端语义（与 H.265 设计一致）

分辨率与 codec 一样是**共享编码器全局属性**：

- 共享流已打开时（引用计数>0），后加入客户端的 `stream_width/height` 不会触发重开编码器，跟随现网分辨率观看；
- 与 §8.3（H.265 设计）"已有会话优先"一致；如需严格化，可复用 `0x1105` 冲突路径扩展 resolution 冲突码（暂不做）。

---

## 5. 本轮改动清单

### CAE（`nexartc-cloud-phone-access-engine`）

| 文件 | 改动 |
|------|------|
| `app/src/main/java/com/nexartc/cae/media/ScreenCapture.java` | 前台监测系统 Context 回退（`getSystemContextFallback()`） |
| `app/src/main/cpp/cae_common/CaeConfigManage.{h,cpp}` | `GetQpIMaxH265/GetQpPMaxH265`（默认 28/32）；启动日志加 `QP_H265=` |
| `app/src/main/cpp/cae_media/media_engine.cpp` | `OpenVideo` 按 codec 选择 QP 上限 |
| `app/src/main/cpp/cae_service/WebRtcServerTransport.cpp` | 场景 boost 按流分辨率像素比缩放；boost 目标增加 REMB×1.35 相对余量下限（修复"REMB 超封顶后 boost 零加码"退化） |
| `app/src/main/cpp/cae_common/CaeConfigManage.cpp` | `scene_change_boost_bitrate` 代码默认 3.5M→4.5M |
| `app/src/main/assets/config/CaeConfig.ini` | 新增 `qp_i_max_h265=28`、`qp_p_max_h265=32`；`scene_change_boost_bitrate=4500000` |

### Web（`nexartc-cloudPhoneAccess-web/device`）

| 文件 | 改动 |
|------|------|
| `index.html` | 设置页新增 Video Codec、Resolution（仅标签）两项 |
| `src/ui.ts` | `codecSelect` / `resolutionSelect` 访问器 |
| `src/main.ts` | codec/resolution 偏好初始化、localStorage、URL 覆盖、日志、`BUILD_VERSION=20260723E` |
| `src/protocol.ts` | `StartCodecOptions.streamWidth/streamHeight`；`media_config` 携带 `stream_width/stream_height` |

### 构建验证

- Web：`npm run build`（tsc + vite）通过；
- CAE：`:app:compileRootDebugJavaWithJavac` 与 `:app:externalNativeBuildRootDebug` 通过。

---

## 6. 真机回归清单

1. 部署新 CAE APK + Web dist；
2. 启动日志确认：
   - `Video quality: ... QP_H265=28/32 ...`
   - `Foreground monitor using system context` + `Foreground monitor started`
3. H.265 连接，云机内播视频 → 手机屏内 Home 键回桌面：
   - CAE 1s 内出现 `Foreground scene changed: reason=home_key` → `quality boost` → `requesting IDR` → `Frame ... key=true`
   - **boost 日志 `target` 必须高于当时 REMB 约 1.35 倍**（如 REMB 4.0M → target≈5400 kbps；若 target==REMB 说明退化修复未生效）
   - boost 到期后确认 REMB 未被反压跌向 1.5M（`quality recovery complete` 后应正常爬升）
   - Web 侧桌面文字 2s 内可读，无 PLI 风暴（`pli` 不持续增长）
4. `QP bounds applied: codec=video/hevc I=[-1,28] P=[-1,32]` 出现且 configure 未降级；
5. 设置页切 480p/360p 重连：START_SUCCESS 返回 536×1160 / 360×780，触控正常，boost 日志 target 按比例缩小；
6. 设置页切 H.264/H.265/Auto 重连：`SDP codec: offer=... answer=...` 符合预期；不支持 H.265 的浏览器强制 H.265 时仍出现阻断提示；
7. 弱网（可选）：确认 QP 收紧后 NACK 无持续风暴，否则放宽 `qp_p_max_h265=34`。

## 7. 决策记录

| 问题 | 决策 |
|------|------|
| H.265 残余模糊是否加码率？ | 否（稳态）；先修前台监测 + HEVC 专用 QP 地板；备选仅动 boost 窗口码率 |
| H.264 切换仍模糊是否加码率？（00:11 复核） | 是，但只加切换窗口：boost 相对余量 REMB×1.35 + 封顶 4.5M；稳态 REMB/override 不动 |
| boost 封顶是否上调？ | 已上调 3.5M→4.5M（00:11）；配合 REMB×1.35 相对余量；仍受 remb_max=6M 约束 |
| 分辨率 UI 是否显示宽高？ | 否，仅显示 720p/480p/360p 标签 |
| 分辨率是否服务端做码率 ladder？ | 否；REMB 自适应 + boost 像素比缩放足够，`remb_min` 缩放留作后续 |
| 多客户端分辨率冲突？ | 沿用"已有会话优先"，后加入者跟随现网分辨率 |

---

## 8. 排查全过程复盘（App ↔ 桌面切换模糊/花屏）

按时间线记录完整排查步骤，便于同类问题复用。

### 8.1 症状

浏览器观看云机画面，云机内在 App（如短视频全屏播放）与桌面之间快速切换时，切换后的画面明显模糊（文字不可读、图标糊边），持续 2~9 秒后才恢复清晰。H.264 更严重，H.265 有改善但仍可见。

### 8.2 排查步骤与每步结论

| 步骤 | 手段 | 发现 | 结论 |
|------|------|------|------|
| 1. 定性模糊机制 | 对照 Web 端 `RTP video` 统计与 CAE `Frame` 日志 | 切换瞬间画面内容剧变，编码器在当前 REMB 目标码率内只能抬 QP 编 P 帧；直到下一个 I 帧（`iframe_interval` 周期或 PLI 触发）画质才恢复 | 模糊 = 场景突变 + 低码率窗口 + 无及时 IDR，不是网络丢包 |
| 2. Phase 0/1：QP 地板 + REMB 下限 + 场景 boost | 配置 `qp_i_max=30/qp_p_max=34`、`remb_min=1.5M`、boost 3.5M + IDR | 明显改善，但系统 Home 键切换仍模糊 | boost 机制正确，但**触发源缺失** |
| 3. 尝试加大 boost 到 5M | 真机实测 | 超大 IDR 导致浏览器 REMB 从 ~3.3M 跌到 1.5M，随后持续模糊 | 冲高码率会被 BWE 反压，方向错误；回退 3.5M 并加 2s REMB 恢复地板 |
| 4. 分析"系统 Home 键为何漏报" | 对照 Web 触控日志与 CAE `NotifySceneChangeForKeyEvent` | Web 工具栏 Home 按钮走 keycode 3 有 boost；**手机屏内导航栏/手势 Home 不经过客户端按键通道**，CAE 无感知 | 需要 CAE 侧前台任务监测（轮询 top activity） |
| 5. 检查前台监测线程为何未启动 | CAE 启动日志 | `Foreground monitor unavailable`：root `app_process` 中 `ActivityThread.currentApplication()==null`，拿不到 `ActivityManager` | **主因确认**：监测线程从未启动，系统级切换只能等浏览器 PLI（滞后 4~9s） |
| 6. 修复 Context 获取 | `ActivityTaskManager.getService()` 反射链（已部署）+ `ActivityThread.getSystemContext()` 回退（Phase 2，待部署） | 重新部署后监测线程启动成功 | 见 §9 日志验证 |
| 7. H.265 残余模糊 | 分析 QP 配置 | HEVC 沿用 H.264 的 QP 上限 30/34，浪费压缩效率 | 新增 `qp_i_max_h265=28/qp_p_max_h265=32`（Phase 2，待部署） |
| 8. H.264 切换过程仍模糊（23:53 日志复盘） | 对照 boost 日志 target 与 REMB | boost 公式 `min(2×REMB, 3.5M封顶)` 再与 REMB 取 max；REMB 已爬到 3.6~4.4M 时封顶低于 REMB，**boost 退化为"仅 IDR、切换窗口零加码"**（日志中 target 恒等于当时 REMB 可证） | boost 目标加相对余量下限 `REMB×1.35`（仍受 2×REMB 与 `remb_max=6M` 约束，2s REMB 恢复地板兜底），把额外码率精确投放在 1.5s 切换窗口而非稳态（Phase 2，待部署） |
| 9. 进一步提高切换窗口码率（00:11） | 用户主观反馈 H.264 切换仍偏糊 | 3.5M 绝对封顶在 REMB 2.6~3.5M 区间也会压制 boost（`min(2×REMB, 3.5M)` 早早触顶） | `scene_change_boost_bitrate` 默认 3.5M→**4.5M**（代码默认 + assets ini 同步）；相对余量取 1.35。示例：REMB=4.0M → boost 目标 5.4M；REMB=3.0M → 4.5M；REMB=2.0M → 4.0M（2×REMB 封顶）。设备运行时 ini 无此键，代码默认直接生效 |

### 8.3 根因汇总

1. **主因**：前台任务监测线程因 root `app_process` 无 `Application` 对象而从未启动 → 系统级 Home/App 切换（手机屏内导航栏、手势）完全没有触发 boost + IDR，只能等浏览器发 PLI（滞后 4~9 秒），期间画面持续模糊。
2. **次因 A（H.264/H.265 通用）**：boost 绝对封顶 3.5M 低于稳态 REMB（3.6~4.4M）时，boost 目标退化为 REMB 本身——切换窗口没有任何额外码率，IDR 和随后的高运动 P 帧只能在稳态码率内抬 QP。
3. **次因 B（H.265）**：HEVC 沿用 H.264 的 QP 上限 30/34，没有用足压缩效率。
4. **反例教训**：直接加大 boost 码率到固定 5M 会产生超大 IDR，触发浏览器 BWE 反压（REMB 崩到下限），比不加更糟；额外码率必须是**相对的、短窗口的**（REMB×1.35、1.5s、封顶 4.5M/2×REMB/remb_max），且配合 REMB 恢复地板。

### 8.4 解决方案汇总

| 方案 | 状态 |
|------|------|
| 场景切换 boost（2.0×、封顶 3.5M、1.5s）+ 立即 IDR + 2s REMB 恢复地板 | ✅ 已部署，已验证 |
| 触发源：客户端按键（keycode 3/4/187）、PLI、视频管道丢帧、**前台任务监测（100ms 轮询）** | ✅ 已部署，已验证 |
| 前台监测 Context：`ActivityTaskManager.getService()` 反射 | ✅ 已部署，已验证 |
| 前台监测 Context：`getSystemContext()` 回退（双保险） | ⏳ 已编码，待部署 |
| H.265 专用 QP 地板 28/32 | ⏳ 已编码，待部署 |
| boost 相对余量下限 REMB×1.35 + 封顶 3.5M→4.5M（修复零加码退化并加量） | ⏳ 已编码，待部署 |
| boost 按流分辨率像素比缩放 | ⏳ 已编码，待部署 |
| Web 设置页 codec / resolution 可配置 | ⏳ 已编码，待部署 |

---

## 9. 真机日志验证（2026-07-23 23:53，device 9898d727）

当前设备运行 21:12 部署的 CAE 构建（含前台监测 + ActivityTaskManager 反射回退；不含 §8.4 中"待部署"项）。用户在今日头条 TikTok 页与 MIUI 桌面之间反复用系统 Home 键/点击图标切换，logcat 证据如下。

### 9.1 前台监测触发 → boost → IDR 全链路（已生效）

```text
23:53:40.373 CaeScreenCapture: Foreground scene changed: reason=home_key
             from=com.ss.android.article.news/...TikTokActivity to=com.miui.home/.launcher.Launcher
23:53:40.373 CAE: NotifySceneChange(): WebRTC quality boost: reason=home_key
             target=3617 kbps duration=1500ms recovery_hold=2000ms floor=3617 kbps ratio=2.00 sessions=1
23:53:40.373 CaeScreenCapture: Key frame requested (caller=forceRequestKeyFrame)
23:53:40.387 CaeScreenCapture: Frame #19556: size=31018 key=true targetBitrate=3617574
```

- **`reason=home_key` 与 `reason=foreground_task_change` 双向切换全部被捕获**（回桌面 = home_key，点图标回 App = foreground_task_change）；
- 场景变化 → IDR 出帧延迟 **14~38ms**（对比修复前等 PLI 的 4~9 秒）；
- IDR 尺寸 25~110KB，处于健康范围（无 5M 时代的超大 IDR）；
- **遗留发现**：所有 boost 日志的 `target` 恒等于当时 REMB（3617/3995/4397），即封顶 3.5M 已低于稳态 REMB，切换窗口实际零加码——这解释了 H.264 切换过程仍可感知的模糊，修复见 §8.2 步骤 8/9。

### 9.2 REMB 未被反压（关键回归点）

切换风暴期间共享编码器码率一路**上行**而非崩塌：

```text
23:53:39  3269 -> 3617 kbps (remb_update)
23:53:42  3617 -> 3995 kbps (remb_update)
23:53:46  3995 -> 4397 kbps (remb_update)
23:53:51  quality boost expired; holding pre-scene REMB floor
23:53:53  quality recovery complete; 4397 -> 3095 kbps (quality_recovery_expired)
23:54:14  3095 -> 3407 -> 3753 -> 4129 kbps (remb_update，正常爬升)
```

- boost 到期 → 2s REMB 地板保持 → 恢复正常 REMB 的三段式状态机按设计运转；
- 恢复后 REMB 从 3095 正常爬回 4129 kbps，无 1.5M 贴底。

### 9.3 PLI 归零

整个观测窗口（数分钟、十余次切换）CAE 侧 **`received PLI` 计数为 0**——浏览器解码端不再因参考帧损坏/画面长期模糊而请求关键帧，说明切换瞬间的 IDR 已经足够及时。

### 9.4 结论与遗留

| 项 | 结论 |
|----|------|
| 前台监测 + boost + IDR 链路 | ✅ 生效，切换检测延迟 ≤100ms（轮询周期），IDR 出帧 ≤40ms |
| REMB 反压回归 | ✅ 无崩塌，恢复地板机制工作正常 |
| PLI 风暴 | ✅ 归零 |
| 待办 | Phase 2 新构建（H.265 QP 地板 28/32、`getSystemContext` 双保险、boost 分辨率缩放、Web 设置页）尚未部署，需重新打包 CAE APK + Web dist 后按 §6 清单回归 |

**主观效果预期**：本次日志对应会话下切换模糊已由"秒级"压缩到"单帧级"；残余的瞬时糊感（IDR 到达前 1~2 帧）属编码固有，部署 H.265 QP 地板后可进一步收敛。

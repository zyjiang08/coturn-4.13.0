# nexartc：App 切回桌面瞬间模糊 / 马赛克优化建议

> 日期：2026-07-24  
> 状态：专家分析 + 对照现网实现的优化建议（设计文档）  
> 问题域：Android 屏幕采集（MediaCodec）在 **Scene Cut（突变场景）** 下的块状效应  
> 关联文档：  
> - [`nexartc-cae-scene-change-blur-qp-design.md`](./nexartc-cae-scene-change-blur-qp-design.md)（QP 地板 / REMB 下限 / boost+IDR）  
> - [`nexartc-scene-change-blur-phase2-codec-resolution.md`](./nexartc-scene-change-blur-phase2-codec-resolution.md)（前台监测 / H.265 QP / boost 余量）  
> - CAE 工程内 [`docs/bitrate_control_analysis.md`](../../../nexartc-cloud-phone-access-engine/docs/bitrate_control_analysis.md)

---

## 1. 现象与本质

### 1.1 现象

云手机内从 App **切回桌面（Launcher）** 的瞬间，远端观看画面出现：

- 图标边缘发虚、壁纸大块马赛克（块状效应）；
- 随后静止数秒后逐渐恢复清晰。

这是屏幕录制里极其经典的 **突变场景（Scene Cut）引发的瞬时编码质量崩溃**，不是 Web 解码器「播不出来」（低延迟路径下 `jbDelay` 往往仍很低）。

### 1.2 瞬间同时发生两件事

| 变化 | 编码影响 |
|------|----------|
| **全局大面积运动**（窗口缩小 / 过渡动画） | 运动估计失效，残差暴涨 |
| **高频复杂纹理涌入**（壁纸、密集图标、文字） | 空间细节熵暴涨，P/B 预测几乎不可用 |

前后帧相似度骤降 → 需要海量比特重建参考 → 若瞬时预算不够，编码器只能 **抬 QP（粗量化）** → 马赛克。

```text
App 内（较平稳 / 可预测）
        │  HOME / 最近任务切换
        ▼
桌面（全局运动 + 高频纹理）
        │
        ├─ 需要：大 IDR + 足够瞬时码率 + 质量地板（限 QP）
        └─ 若只有稳态 CBR 目标：码率饥饿 → QP 飙升 → 糊
```

---

## 2. CBR 场景：为何必糊（与现网高度相关）

nexartc CAE 当前 MediaCodec 路径为 **硬编码 CBR**：

```java
format.setInteger(MediaFormat.KEY_BITRATE_MODE,
        MediaCodecInfo.EncoderCapabilities.BITRATE_MODE_CBR);
```

（`ScreenCapture.buildVideoFormat()`；`enable_cbr` 配置参数在 Java 侧实际被忽略，「CBR always」。）

### 2.1 码率饥饿（Bitrate Starvation）

CBR 强制「这一秒」的输出预算贴近目标（如 3.5–5 Mbps；再叠加 WebRTC REMB 后可能更低）。  
切桌面瞬间的信息量可能需要 **远高于稳态** 的瞬时带宽（量级上可到数倍～十几 Mbps 才「够画清楚」）。预算锁死 → 必然削质量。

### 2.2 QP 飙升

为不突破 CBR 天花板，硬件编码器大幅提高量化参数：

- 图标边缘发虚、壁纸出现大块；
- 画面静止后，P 帧逐步修复 → 肉眼感觉「过一会又清晰了」。

### 2.3 与 WebRTC 叠加的放大效应

云访问链路还有 **REMB/BWE 共享目标码率**：

- 双人观看时 `shared_target` 可被压到亚 Mbps（见运维日志分析）；
- 场景 boost 若 **封顶低于当前 REMB**，会退化成「只插 IDR、零加码」——糊仍在。

> Android `BITRATE_MODE_CBR` 往往是「目标码率」而非绝对恒定，硬件允许一定波动；但 **对 Scene Cut 仍远不够**，不能当成 VBR 峰值余量。

---

## 3. VBR 场景：为何仍会糊

即便改成普通 VBR，切桌面仍可能糊，原因通常不是「VBR 没用」，而是配置与算法限制：

### 3.1 无 Lookahead / 反应滞后

多数手机硬编缺少足够前瞻。突变发生后，前若干帧仍按「上一场景」的低复杂度估计跑；等 RC 认定 Scene Cut 并上调码率，已过去十几帧——糊已经发生。

### 3.2 MaxRate 过低

VBR 仍有峰值上限。若 `平均 8 Mbps、峰值仅 10 Mbps`，切桌面仍触顶，重演 CBR 饥饿。

**经验**：平均目标若约 6 Mbps，峰值宜给到 **3～4×**（约 18–24 Mbps）才有「半秒桌面纹理」的喘息空间——且需网络/NACK 能吃掉尖峰。

### 3.3 与 CQ 的区别

CQ / VBR_HQ（恒定质量优先）让编码器优先守住质量，瞬时码率可突破「舒适区」。更适合录屏主观质量，但云访问还要考虑上行与多会话 BWE，不能无限制冲高。

---

## 4. 通用四步解法（行业对照）与 nexartc 映射

| 方案 | 行业做法 | nexartc 现状 | 建议 |
|------|----------|--------------|------|
| **一、CQ / 质量优先** | `BITRATE_MODE_CQ` + 可接受平均码率 | 已支持 `bitrate_mode=cbr\|vbr\|cq`（默认 cbr；CQ 不支持则回退 CBR） | 画质优先可试 `cq`/`vbr`；公网多会话仍建议 cbr + boost |
| **二、强制关键帧** | 监听到切桌面立刻 `PARAMETER_KEY_REQUEST_SYNC_FRAME` | ✅ 已有：前台 Activity 监测 → `home_key` / `foreground_task_change` → `NotifySceneChange` → boost + IDR | 继续加固触发覆盖与延迟（§5.2） |
| **三、缩短 GOP** | 更短 `KEY_I_FRAME_INTERVAL` | 配置项存在；过短增带宽与 IDR 风暴风险 | 作兜底，优先事件驱动 IDR（§5.3） |
| **四、拉高峰值缓冲** | VBR `KEY_MAX_BIT_RATE` = 3～4× avg | CBR 路径曾讨论 `max-bitrate`；boost 窗口已做短时抬码 | 在 **boost 窗口** 给足峰值，勿盲目抬稳态 CBR（§5.4） |

另：nexartc 已落地且应保留的「质量地板」：

- `qp_i_max` / `qp_p_max`（及 H.265 专用更严地板）；
- `remb_min_bitrate`；
- `scene_change_boost_*` + 恢复 hold（防 REMB 立刻打穿）。

---

## 5. 针对 nexartc 的优化建议（按优先级）

### 5.1 P0 — 守住已验证路径（先回归再加码）

目标：切桌面时日志必须出现完整链路，且 boost **真正加码**：

```text
Foreground scene changed: reason=home_key|foreground_task_change ...
WebRTC quality boost: reason=... target=XXXX kbps ...
→ IDR / IRAP 发出
→ VIDEO_PIPE / ENC kbps 在窗口内明显高于稳态
```

检查清单：

1. root 设备前台监测可用（`CaeForeground` 线程；无权限时静默失效是历史坑）。  
2. `scene_change_boost_bitrate` **高于** 当前 REMB，并保留相对余量（**≥ REMB×2**，再受 `remb_max` 约束）。  
3. QP 地板生效（H.264 / H.265 分档）。  
4. 单人会话验证（双人 `shared_target` 过低会掩盖一切画质优化）。

### 5.2 P0 — 强制关键帧：覆盖「切桌面」全路径

已有 Accessibility / 前台任务监测；建议补强：

| 触发 | 说明 |
|------|------|
| `home_key` / Launcher 包名 | 已有 |
| 最近任务 / 手势上滑进桌面 | 确认是否被 `foreground_task_change` 覆盖 |
| 客户端 HOME / 多任务键注入 | `NotifySceneChangeForKeyEvent` 路径保持 |
| PLI | 已映射到 `NotifySceneChange("pli")`；注意节流，避免 IDR 风暴 |

实现要点（与行业方案二一致）：

```java
Bundle bundle = new Bundle();
bundle.putInt(MediaCodec.PARAMETER_KEY_REQUEST_SYNC_FRAME, 0);
mediaCodec.setParameters(bundle);
```

注意部分 MTK 机型对 sync-frame 参数响应弱，需依赖「输出侧识别 IDR + 必要时 recreate」的既有兜底（见 `CaeEngineControl` 注释）。

### 5.3 P1 — GOP / Intra-Refresh：兜底而非主武器

| 做法 | 建议 |
|------|------|
| 全局 `iframe_interval` 收到 0.5s | 慎用：稳态带宽↑、弱网 IDR 风暴 |
| 场景窗口内临时缩短 GOP | 更优：仅 boost 期间提高关键帧密度 |
| Intra-Refresh | 可与周期 IDR 二选一为主；并存时部分机型行为未定义（见 bitrate_control_analysis） |

推荐：**事件驱动 IDR + 短时 boost** 为主；GOP 缩短仅作「监测失效」时的保底配置档。

### 5.4 P1 — 峰值空间：放在 boost，不放在稳态 CBR

| 不推荐 | 推荐 |
|--------|------|
| 把稳态 `override_bitrate` 从 3.5M 盲目拉到 8M+ | 提高 `scene_change_boost_bitrate` / ratio，窗口 1～2s |
| 无上限 CQ 上生产公网 | CQ 仅作「画质优先」实验 flavor |
| MaxRate ≈ Avg | 若试 VBR/CQ：Max ≥ 3× Avg，且受 `remb_max` 与会话数约束 |

多会话时：评估 **shared_target 地板**（避免第二路把共享编码器打到 0.75 Mbps，任何 Scene Cut 优化都会失效）。

### 5.5 P2 — 码率模式：CBR / VBR / CQ（Web 客户端配置）

**主配置入口：Web 连接页「码率模式」**（`bitrateModeSelect`），经 START `media_config:bitrate_mode=` 与 WebRTC JSON `bitrate_mode` 下发；CAE 写入 runtime override，在 `OpenVideo` 时生效。

```text
Web: cbr | vbr | cq  →  media_config bitrate_mode=...
CAE: SetRuntimeBitrateModeOverride → ScreenCapture KEY_BITRATE_MODE
```

- 未下发时回退 `CaeConfig.ini` 的 `bitrate_mode` / `enable_cbr`。  
- **vbr/cq**：配合 `max_bitrate_ratio` 写 `KEY_MAX_BIT_RATE`；机型不支持 CQ 时回退 CBR。  
- URL 快捷参数：`?bitrate_mode=cq`。

### 5.6 P2 — 分辨率短暂降载（已有 Phase2 方向）

切桌面若仍糊：boost 窗口内允许 **短时降编码分辨率**（或降 fps），用像素预算换纹理清晰，再恢复。详见 Phase2 文档；与 CQ 二选一试验，避免同时改太多变量。

---

## 6. 推荐参数档（在现有 CBR 体系内）

面向「回桌面清晰」的 **生产默认方向**（数值需按机型回归）：

| 参数 | 建议方向 | 理由 |
|------|----------|------|
| `KEY_BITRATE_MODE` | 维持 CBR（默认） | 与 BWE/多会话可控 |
| `override_bitrate` | 3.5～5 Mbps（720p 竖屏） | 稳态够用即可 |
| `qp_i_max` / `qp_p_max` | H.264：≤30/34；H.265：更严（如 28/32） | 质量地板 |
| `remb_min_bitrate` | ≥1.5 Mbps（720p） | 防止合法糊死 |
| `scene_change_boost_ms` | 1000～2000 ms | 覆盖过渡动画 |
| `scene_change_boost_ratio` | **2.0（REMB×2）** | 相对下限已与 ratio 对齐 |
| `scene_change_boost_bitrate` | 与 `remb_max` 对齐（如 6 Mbps） | 避免绝对封顶把 REMB×2 掐死 |
| `iframe_interval` | 保持数秒级；依赖事件 IDR | 避免全局过密 GOP |
| 双人 `shared_target` | 设合理地板（产品决策） | 否则 Scene Cut 无解 |

---

## 7. 验证方法

### 7.1 主观

1. 单人连接，720p，H.264 / H.265 各测一轮；  
2. 云机内打开复杂 App → 按 Home 回桌面；  
3. 观察切换瞬间图标/壁纸是否仍有明显马赛克，以及恢复时间是否 &lt; 0.5s。

### 7.2 日志（CAE）

必须对齐时间戳核对：

| 关键字 | 期望 |
|--------|------|
| `Foreground scene changed: reason=home_key` | 切换当帧附近出现 |
| `WebRTC quality boost: reason=home_key` | target 有意义地高于切换前 |
| `NAL ... IDR` / H.265 IRAP | boost 后立即出现 |
| `VIDEO_ENC_OUT` / `VIDEO_PIPE_OUT` kbps | 窗口内抬升 |
| `WebRTC-PLI` | 理想为 0 或很少（说明发送端已自愈） |

### 7.3 对照实验（单变量）

1. 仅关 boost → 糊应明显变差；  
2. 仅松 QP 地板 → 糊应变差；  
3. 双人同看 → 若 shared 过低，单人优化全部失效（先修 BWE 地板再谈 Scene Cut）。

---

## 8. 结论

1. **App→桌面模糊的本质**是 Scene Cut 下预测失效 + 瞬时信息量爆炸，编码器在预算不足时抬 QP。  
2. **现网默认是 CBR**，与「码率饥饿 → QP 飙升」模型完全吻合；VBR 若 MaxRate/反应不足同样会糊。  
3. nexartc 已具备正确方向的组合拳：**前台场景检测 + 强制 IDR + 短时码率 boost + QP 地板 + REMB 下限**；优化重点是让这条链路 **必触发、真加码、峰值够、多会话不挖坑**。  
4. **CQ** 是画质向的有力选项，建议作为可配置实验模式，而不是在未解决多会话 BWE 前替换默认 CBR。  
5. **不要**指望只靠抬稳态 CBR 解决切桌面糊——尖峰应花在 **场景窗口**，稳态保持可控。

---

## 9. 后续工作（可选实现拆分）

| ID | 项 | 优先级 |
|----|----|--------|
| SC-1 | 回归：home_key → boost→IDR 全链路（H.264/H.265、单人） | P0 |
| SC-2 | `bitrate_mode=cbr\|vbr\|cq` 已落地；设置页可选 | done |
| SC-3 | boost 窗口临时 MaxBitrate / 复杂度提示（机型相关键） | P1 |
| SC-4 | 多会话 shared_target 地板策略 | P0（若产品要双人清晰） |
| SC-5 | 监测失效时的短 GOP 保底档 | P2 |

评审通过后，实现项以独立 PR 落地，并回写 Phase2 文档的验证表。

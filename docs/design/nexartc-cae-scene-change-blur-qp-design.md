# nexartc CAE 场景切换模糊与 QP / 码率优化设计

> 日期：2026-07-23
>
> 状态：分析完成，待按阶段实施
>
> 适用范围：`nexartc-cloud-phone-access-engine` 屏幕采集编码（H.264 / H.265）+ WebRTC REMB/BWE；浏览器侧低延迟抖动缓冲不在本文主路径
>
> 关联文档：
> - [`nexartc-web-cae-h265-support-design.md`](./nexartc-web-cae-h265-support-design.md)
> - [`nexartc-logging-design.md`](./nexartc-logging-design.md)
> - [`nexartc-webrtc-ice-udp-mux-logical-separation.md`](./nexartc-webrtc-ice-udp-mux-logical-separation.md)

---

## 1. 结论摘要

在「云手机内播放视频 → 返回桌面」这类**场景突变**过程中，远端画面会出现短暂或持续发糊，同时常见：

- RTP 发送/接收码率从数 Mbps 跌到数百 kbps；
- 统计 fps 跌到个位数（如 ~2）；
- PLI 次数上升；
- 视频 `jbDelay` 仍很低（如 ~1ms），说明**低延迟抖动缓冲已生效，糊不是播不出来，而是编码质量塌了**。

根因组合：

1. 场景突变需要高质量 IDR/IRAP 重建参考；
2. REMB/BWE 已把目标码率压低；
3. 当前配置 `qp_*=-1`（不限制 QP），编码器优先保码率而抬高 QP；
4. `iframe_interval` 偏长且未开 intra-refresh 时，质量恢复慢。

**可以、也应该限制 QP**（设置 `qp_i_max` / `qp_p_max`）作为质量地板；但单靠限 QP 不够，还需：

- 提高 `remb_min_bitrate`，避免合法地编码到「糊死」区间；
- 场景切换 / PLI 后**短时抬码率 + 强制 IDR**；
- 适度缩短关键帧间隔或启用 intra-refresh。

优化目标：

> 允许短时码率尖峰，换取场景切换清晰；稳态再跟随 REMB。

---

## 2. 复现与观测

### 2.1 复现路径

1. Web 连接 CAE（H.264 或 H.265 均可复现）；
2. 云手机内播放视频（高运动 / 高纹理）；
3. 返回桌面（Launcher / 静态或低运动 UI）；
4. 观察远端画面：切换瞬间或随后数秒发糊；日志中码率与 fps 下降、PLI 上升。

### 2.2 典型统计（H.265 实测片段）

| 阶段 | 视频码率 | fps | PLI | 视频 jbDelay | 画面 |
|------|----------|-----|-----|--------------|------|
| 视频播放中 | ~3–4 Mbps | ~50–60 | 0 | ~1ms | 正常 |
| 回桌面后 | ~0.4–0.8 Mbps | ~2–4 | 上升 | ~1ms | 模糊 |

含义：

- **低延迟路径正常**（jbDelay 低）；
- **编码端质量不足**（码率被打穿 + QP 无上限）。

### 2.3 相关配置现状（`CaeConfig.ini` `[video]`）

| 项 | 典型现状 | 影响 |
|----|----------|------|
| `override_bitrate` | 3500000 | CBR 名义目标 |
| `enable_cbr` | 1 | 码率受控 |
| `enable_intra_refresh` | 0 | 无渐进刷新 |
| `iframe_interval` | 10 | 周期关键帧稀疏 |
| `max_bitrate_ratio` | 2.0 | IDR 峰值预算有限 |
| `qp_i_max` / `qp_p_max` | **-1** | **无质量地板** |
| `remb_min_bitrate` | 800000 | 下限偏低（720p 竖屏桌面偏紧） |
| `remb_initial_bitrate` | 1000000 | 开局保守 |
| `remb_max_bitrate` | 6000000 | 上限充足 |

代码侧 `ScreenCapture` 已支持 QP bounds（`KEY_VIDEO_QP_*`）；配置为 `-1` 时不施加上限。注释写明 WebRTC 下不限 QP 是为避免 IDR 过大导致 UDP 丢包——这与「切换清晰」存在权衡，需用短时抬码率 + NACK 消化尖峰，而不是永久不设质量地板。

---

## 3. 根因分析

```text
播视频（高码率、高运动）
    → 回桌面（场景突变，参考帧失效）
    → 需要 IDR/IRAP
    → 同时 REMB 可能已把目标码率压到数百 kbps～1Mbps
    → qp_*=-1：编码器抬 QP 保码率
    → 关键帧本身糊 → 后续 P 帧慢恢复
    → 桌面相对静止：部分机型降帧 → 统计 fps≈2
    → 解码侧发起 PLI → 再次要 IDR（若仍低码率则再次糊）
```

### 3.1 场景突变与关键帧预算

从高运动视频切到桌面时，运动估计/参考相关性骤降，必须依赖关键帧。关键帧在低目标码率下只能：

- 提高 QP，或
- 产生极大的瞬时码率尖峰。

当前 `max_bitrate_ratio=2.0` 且 REMB 目标已很低时，IDR 清晰度不足。

### 3.2 QP 无上限

`qp_i_max=-1`、`qp_p_max=-1` 时，CBR/REMB 收紧会直接反映为 QP 上升。桌面细文字、图标边缘对高 QP 最敏感，表现为「整屏发糊 / 块效应」。

**限制 max QP 可以防止质量打穿地板**——这是对本问题的直接、有效手段之一。

### 3.3 REMB / 共享编码器

WebRTC 路径下浏览器 REMB 会驱动共享编码器目标码率（见 `WebRtcServerTransport` BWE）。若：

- `remb_min` 过低，或
- 切换瞬间估计偏悲观，

则编码器会长时间停在「合法但不可读」的码率区。限 QP 后，编码器可能短时超目标码率；需接受尖峰或同步抬 `remb_min` / 切换窗口目标码率。

### 3.4 关键帧间隔与 intra-refresh

- `iframe_interval=10`：无 PLI 时最差约 10s 才有周期 IDR；
- `enable_intra_refresh=0`：无法用渐进刷新平滑恢复；
- PLI 已有节流（如 300ms），但若 IDR 落在低码率窗口，仍糊。

### 3.5 与 H.265 的关系

H.265 同等画质通常更省码率，但**场景突变 + 低码率 IDR** 时同样会糊；问题不独属于 H.265。H.264 / H.265 共用本方案。H.265 下限 QP 的收益往往更明显（同 QP 主观更好，但仍需码率预算打 IDR）。

### 3.6 与「低延迟」的关系（三类含义勿混用）

口语里的「低延迟」在本项目中至少有三层，**彼此独立**：

| 层级 | 配置 / 机制 | 作用 | 是否影响「发糊」 |
|------|-------------|------|------------------|
| A. 播放侧 / SDP | `webrtc_low_latency`、去 LS、`playout-delay`、`playoutDelayHint=0` | 降低浏览器 **jbDelay** | 否；糊时 jbDelay 仍可很低 |
| B. 采集侧 | `Surface.setFrameRate`（绕开部分 VSync） | 少等显示刷新，约省数 ms | 否；不决定 QP/码率 |
| C. 编码器侧「快速出帧」 | MediaCodec `KEY_LATENCY` / `KEY_PRIORITY` / `FEATURE_LowLatency` / 厂商 low-latency 等 | 缩短编码流水线缓冲、尽快产出 access unit | 否；与质量地板正交 |

**结论（与实测一致）：**

- 场景切换发糊时，视频 `jbDelay` 仍可 ~1ms → **A 已生效，不是根因**；
- 本文 P0/P1（限 QP、抬 `remb_min`、切场景 boost + IDR）解决的是**编码质量**，不依赖关闭 A；
- **C（编码器快速出帧）当前未真正打开**：`ScreenCapture.buildVideoFormat()` 对 H.264 / H.265 共用同一套参数，已有 CBR、`KEY_REPEAT_PREVIOUS_FRAME_AFTER`、Surface `setFrameRate`，但**未设置** `KEY_LATENCY`、`KEY_PRIORITY`（realtime）、`KEY_OPERATING_RATE`、`FEATURE_LowLatency` 或厂商 low-latency key；
- 配置项 `webrtc_low_latency` **只影响 A（WebRTC SDP/播放）**，**不会**打开 C；
- H.265 **没有**独立的编码器低延迟开关；与 H.264 同路径、同缺口；
- **QP min/max** 与 C 独立：QP 管主观质量地板，C 管出帧时延。

后续若增加 `encoder_low_latency=1` 一类开关，应写在 `buildVideoFormat()` / `configure` 降级路径中，并与本文 QP/码率方案解耦验收。细节见 [`nexartc-web-cae-h265-support-design.md`](./nexartc-web-cae-h265-support-design.md) §10.5.1。

---

## 4. 设计目标与非目标

### 4.1 目标

1. 「视频 → 桌面」切换后，**1～2 秒内**恢复可接受的桌面清晰度（文字可读）；
2. 稳态仍跟随 REMB，不长期顶满 `remb_max`；
3. 通过 **QP 上限** 保证最低主观质量；
4. 切换窗口允许可控的码率尖峰，并用现有 NACK/PLI 机制吸收；
5. H.264 / H.265 行为一致、可配置、可观测。

### 4.2 非目标

- 不引入第二路编码器做「桌面专用码率」；
- 不做服务端转码 / 双码流；
- 不修改 coturn 媒体转发；
- 不保证所有机型 MediaCodec 都完整支持全部 QP key（需降级探测）；
- 不把低延迟模式作为糊屏根因去关闭。

---

## 5. 优化方案

### 5.1 方案总览

| 优先级 | 手段 | 作用 | 风险 |
|--------|------|------|------|
| P0 | 配置 `qp_i_max` / `qp_p_max` | 质量地板，直接减轻糊 | IDR/帧变大，短时丢包 |
| P0 | 提高 `remb_min_bitrate` | 避免稳态码率过低 | 弱网下可能更卡 |
| P1 | PLI/场景切换短时抬码率 + IDR | 关键帧有预算 | 实现复杂度、尖峰 |
| P1 | 缩短 `iframe_interval` 或开 intra-refresh | 加快恢复 | 平均码率/平滑度权衡 |
| P2 | 场景复杂度检测主动 IDR | 不依赖客户端 PLI | 误触发、机型差异 |
| P2 | H.265 专用初始码率/QP 表 | 更好发挥 HEVC | 配置面变多 |

### 5.2 P0：限制 QP（必须做）

**回答：可以限制 QP，且应当限制。**

建议起步值（720×1560 / ~3.5Mbps 量级，可按机型调）：

| 参数 | 建议值 | 说明 |
|------|--------|------|
| `qp_p_max` | 32～36 | P 帧质量地板；桌面滑动/切页 |
| `qp_i_max` | 28～32 | 关键帧清晰度 |
| `qp_i_min` / `qp_p_min` | -1 或轻度下限 | 避免无谓浪费 |

原则：

- 上限越严 → 切换越清晰 → 尖峰越大；
- 若出现 UDP 丢包 / NACK 风暴，先略放宽 `qp_p_max`（如 36），而不是回到 `-1`；
- MediaCodec `configure` 失败时保留现有「去掉 QP 重试」降级，并打日志。

配置示例：

```ini
qp_i_max=30
qp_p_max=34
qp_i_min=-1
qp_p_min=-1
```

### 5.3 P0：提高 REMB 下限

| 参数 | 现状建议调整 |
|------|----------------|
| `remb_min_bitrate` | 800000 → **1500000～2000000** |
| `override_bitrate` | 保持 2.5M～3.5M（H.265 可略低） |
| `max_bitrate_ratio` | 2.0 → 切换窗口可用 **2.5～3.0** |

720p 竖屏桌面在 &lt;1Mbps 时文字可读性差；提高 min 是「防糊」的稳态手段。

### 5.4 P1：切换窗口「抬码率 + IDR」

触发条件（满足其一即可）：

1. 收到 PLI（已有链路）；
2. 可选：编码输出复杂度/帧大小相对滑动窗口突增或突降超过阈值（场景切变启发式）。

动作（建议 1～2 秒窗口）：

1. 将编码器目标码率临时设为  
   `min(remb_max, max(当前目标, override_bitrate 或 remb_initial～override))`；
2. 调用已有 `ForceRequestIframe()`（保留节流，避免 PLI 风暴）；
3. 窗口结束后回到 REMB 聚合目标。

伪流程：

```text
onPLI / onSceneCut
  → boostBitrate(duration=1..2s)
  → ForceRequestIframe()
  → (optional) temporarily raise max_bitrate_ratio
  → after timer: restore REMB target
```

与限 QP 的配合：

- 限 QP 保证即使 boost 不足也不至于 QP=40+；
- boost 保证限 QP 时 IDR 有足够比特，减少「QP 触顶但仍不够」的时间。

### 5.5 P1：关键帧与 intra-refresh

| 选项 | 建议 | 适用 |
|------|------|------|
| `iframe_interval` | 10 → **2～5** | WebRTC 云手机交互 |
| `enable_intra_refresh` | 桌面长会话可试 **1** | 减少周期性大 IDR 尖峰，恢复更「匀」 |

注意：intra-refresh 与「瞬时最清晰 IDR」策略不同；切换清晰仍优先 **boost + IDR**，intra-refresh 作补充。

### 5.6 P2：观测与验收指标

日志字段（建议对齐 logging 设计）：

- `encoder_bitrate_target_kbps`
- `encoder_qp_i_max` / `encoder_qp_p_max`（配置生效值）
- `boost_active` / `boost_reason=pli|scene_cut`
- `iframe_req` / `pli_count`
- 可选：关键帧后 1s 内平均帧大小、是否触 QP 上限（若机型可查）

Web Stats 验收：

| 场景 | 期望 |
|------|------|
| 视频 → 桌面 | 糊感 &lt; 2s；文字可读 |
| 切换后 3s | 视频码率回升到 ≥ remb_min；fps 回到合理区间（非长期 ~2，除非真静止且编码器降帧） |
| PLI | 允许短时上升，但不应持续风暴 |
| jbDelay | 低延迟模式下视频仍应保持低延迟（与本次优化无关回归） |
| 弱网 | 可接受略卡，不可接受长期马赛克桌面 |

---

## 6. 分阶段实施

### Phase 0：配置护栏（先做、可只改 ini）

1. 设置 `qp_i_max` / `qp_p_max` 起步值；
2. 上调 `remb_min_bitrate`；
3. 将 `iframe_interval` 调到 2～5；
4. 复测「视频 → 桌面」。

完成标准：主观糊明显减轻；若尖峰丢包再微调 QP。

### Phase 1：切换窗口逻辑

1. PLI 触发 bitrate boost + IDR；
2. boost 与共享编码器 / 多客户端 fan-out 规则对齐（全会话共享一个编码器时，boost 影响所有观看端——可接受）；
3. 日志与 Stats 打点。

### Phase 2：精细化

1. 场景切变启发式（减少纯靠 PLI 的滞后）；
2. H.264 / H.265 分档默认 QP 与 min bitrate；
3. 机型 QP key 能力探测与文档化；
4. A/B：限 QP vs 不限 QP 的切换清晰度与丢包率。

---

## 7. 代码与配置改动面（实施时）

| 模块 | 改动 |
|------|------|
| `CaeConfig.ini` / Settings | QP、remb_min、iframe_interval、可选 boost 开关与时长 |
| `CaeConfigManage` | 读取/默认值；WebRTC 下允许 QP 上限（取消「永远 -1」的产品默认） |
| `ScreenCapture.java` | 已有 `applyQpBounds`；确认 H.265 路径同样生效；configure 失败降级日志 |
| `WebRtcServerTransport` / BWE | PLI 回调中触发 boost；与 `ForceRequestIframe` 协作 |
| `CaeEngineControl` | 统一 `SetBitrate` / IDR 入口，避免多处打架 |
| Web device Stats（可选） | 展示 boost / codec / 码率，便于验收 |

---

## 8. 风险与缓解

| 风险 | 影响 | 缓解 |
|------|------|------|
| QP 上限导致 IDR 过大 | UDP 丢包、花屏 | boost 窗口 + NACK；略放宽 qp_i_max；控制 iframe 频率 |
| remb_min 过高 | 弱网卡顿 | 按 ice_mode/relay 分档 min；文档说明 |
| 多客户端共享编码 | 一端 PLI boost 影响全部 | 可接受；或仅在单会话时强 boost |
| 部分机型忽略 QP key | 配置无效 | 探测 + 日志；依赖 remb_min + boost |
| 静止桌面 fps≈2 | 误判为故障 | 文档说明「静止降帧」；以主观清晰度与 PLI 恢复为准 |

---

## 9. 决策记录

| 问题 | 决策 |
|------|------|
| 能否用限 QP 避免切换模糊？ | **能，作为 P0 质量地板** |
| 是否只改 QP？ | **否**；必须配合 remb_min 与切换抬码率/IDR |
| 是否关闭低延迟？ | **否**；指播放侧/SDP（层级 A）；jbDelay 正常，与糊正交 |
| 编码器快速出帧（层级 C）是否已开？ | **否**；H.264/H.265 均未设 MediaCodec latency/priority 等；与糊正交，另项跟踪 |
| 是否默认永久高码率？ | **否**；稳态仍 REMB，仅切换窗口抬升 |

---

## 10. 小结

「播视频回到桌面发糊」是典型的**场景突变 × 低目标码率 × 无 QP 地板**问题，不是播放侧/SDP 低延迟（层级 A）失效，也不是编码器快速出帧（层级 C）未开导致。

推荐路径：

1. **限 QP**（`qp_i_max` / `qp_p_max`）；
2. **提高 `remb_min_bitrate`**；
3. **PLI/切场景短时抬码率 + IDR**；
4. 缩短关键帧间隔或按需开 intra-refresh；
5. 用日志与「视频→桌面」用例做验收。

按 Phase 0 → Phase 1 实施后，切换模糊应明显收敛，同时保留 WebRTC 带宽自适应能力。

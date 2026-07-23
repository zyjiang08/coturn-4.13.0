# nexartc Chrome / CAE H.264 Profile-Level 诊断与修复设计

> 日期：2026-07-23
>
> 状态：P0 修复已实现；Chrome 145/Linux + Xiaomi MI 9 已完成真实链路验证，Safari 真机矩阵仍待补齐
>
> 适用范围：`nexartc-cloudPhoneAccess-web/device/`、`nexartc-cloud-phone-access-engine/` WebRTC H.264 链路
>
> 关联文档：
> - [`nexartc-web-cae-h265-support-design.md`](./nexartc-web-cae-h265-support-design.md)
> - [`nexartc-logging-design.md`](./nexartc-logging-design.md)
> - [`nexartc-turn-mode-a-implementation.md`](./nexartc-turn-mode-a-implementation.md)

---

## 1. 结论摘要

本次 Chrome 联调不能得出“Chrome 不支持 H.264 Profile 4.1”的结论，原因有两点：

1. **4.1 是 Level，不是 Profile。**准确描述应为 `High@Level 4.1`；
2. Chrome 的 `RTCRtpReceiver.getCapabilities('video')` 在本次环境中只列出 Level 3.1 的默认 `profile-level-id`，但对应 fmtp 同时声明了 `level-asymmetry-allowed=1`。这表示收发方向可以使用不同 Level，不能把 capability 中的 `1f` 直接当成解码器最大 Level。

本次问题的确定性根因在 CAE：

> CAE 请求 Android `MediaCodec` 输出 High@L3.1，实际 SPS 为 High@L4.2；WebRTC SDP 仍声明 High@L3.1，造成 SDP 与码流不一致。

当前码流约为 `720x1560@60fps`。按 H.264 Level 的 MaxFS/MaxMBPS 约束计算，该负载不满足 Level 3.1，也超过 Level 4.1 的宏块率上限；硬件编码器自动输出 Level 4.2 是合理行为。因此不能通过把 SDP 字符串强制改成 `640029` 来修复。

P0 修复采用以下原则：

1. Web 显式上报每个 H.264 capability 是否允许 Level Asymmetry；
2. CAE 根据分辨率和帧率计算编码端最低所需 Level；
3. CAE 从首个 SPS 读取编码器实际 `profile-level-id`，并在创建 SDP Offer 前等待该值；
4. SDP Offer 必须使用实际 SPS 的 Profile/Level，不再使用“请求给 MediaCodec 的 Level”冒充实际 Level；
5. 浏览器允许 Level Asymmetry 时，Answer 可返回较低 Level，但 Profile 必须与实际码流兼容；
6. 共享编码器已有会话时，新客户端按实际 SPS 做兼容性门禁；
7. 编码器实际 Profile 与浏览器能力不兼容时，按 High → Main → Baseline 降级并重启一次一个候选，而不是继续宣告错误 SDP；
8. 无法获得 SPS、没有可兼容 Profile 或实际 Level 超出非对称关闭的接收能力时，拒绝创建 Offer并返回明确错误。

---

## 2. 日志事实

### 2.1 浏览器 capability

初始问题日志（macOS/Safari 兼容性回归）显示：

```text
Browser H.264 capabilities: Base/Main/High
profile-level-id=42001f/42e01f/4d001f/f4001f/64001f
```

随后在本机 Chrome 145/Linux 中直接读取 `RTCRtpReceiver.getCapabilities('video')`，
`packetization-mode=1` 的实际接收能力为：

```text
42001f, 42e01f, 4d001f, f4001f
level-asymmetry-allowed=1
```

该 Chrome 环境没有广告 `64xxxx` High Profile，因此 CAE 正确选择了双方都支持的
Main，而不是强行宣告 High。这里的 `f4001f` 也不能未经 profile 映射校验就当作
High；Web 和 CAE 只接受已知且可验证的 Profile 映射。

`profile-level-id` 为 3 字节十六进制值：

```text
profile_idc | profile_iop | level_idc
```

关键值解释：

| `profile-level-id` | 含义 |
|---|---|
| `64001f` | High@L3.1 |
| `640029` | High@L4.1 |
| `640c29` | Constrained High@L4.1 |
| `64002a` | High@L4.2 |
| `640c1f` | Constrained High@L3.1 |

初始 Safari capability 中列出的值最后一个字节都是 `1f`，因此该浏览器 API 没有直接列出
L4.1/L4.2；本机 Chrome 的 capability 也采用同样的默认 Level。Chrome 同一 fmtp 包含：

```text
level-asymmetry-allowed=1
```

这允许 Offer 与 Answer 使用不同的 Level。当前日志只能证明浏览器声明了非对称能力，不能单凭该日志断言每个硬件解码器都能处理 L4.1/L4.2；因此不能用 Answer 的 `1f` 推断发送端必须输出 L3.1，也不能把 Chrome 版本号当成硬件能力保证。最终结果仍以 `setRemoteDescription()`、Answer 的视频 m-line 和实际解码统计为准。

### 2.2 SDP 协商

浏览器日志：

```text
SDP Offer: H.264 profile=High@L3.1 profile-level-id=64001f
SDP Answer: H.264 profile=High@L3.1 profile-level-id=64001f
```

该结果只能证明浏览器接受了 CAE 声明的 `64001f`。它不能证明 RTP 中 SPS 也是 `64001f`。

### 2.3 CAE 编码器实际输出

设备 logcat：

```text
setH264Level: Level 3.1 (level_idc=31)
Forced H.264 High level_idc=31
Encoder SPS profile-level-id=64002a
expected_profile=High
expected_level_idc=31
mismatch: profile=true constraints=true level=false
```

在 WebRTC 请求前，默认 L4.0 配置也出现相同结果：

```text
setH264Level: Level 4.0 (level_idc=40)
Encoder SPS profile-level-id=64002a
expected_level_idc=40
mismatch: ... level=false
```

这证明 Xiaomi 设备上的硬件编码器会根据实际负载把 Level 提升为 4.2，并忽略较低的 `MediaFormat.KEY_LEVEL` 请求。

修复后的真实运行日志为：

```text
H.264 encoder level target: level_idc=42 required for 720x1560@60fps
H.264 encoder actual SPS: generation=1 profile-level-id=64002a
WebRTC H.264 encoder switch: profile high -> main, level 42 -> 42
H.264 encoder actual SPS: generation=2 profile-level-id=4d002a
WebRTC H.264 SPS/SDP aligned: profile=main receiver_id=4d001f actual_sps=4d002a required_level=42 level_asymmetry=1
WebRTC SDP: H.264 main profile-level-id=4d002a
WebRTC answer H.264 level asymmetry: offer=4d002a answer=4d001f allowed=1
```

这证明 Offer 已使用实际 SPS，而不是使用浏览器默认的 L3.1 或旧的编码器请求值。

### 2.4 Chrome Level 结论

`640029` 表示 `High@L4.1`，不是“High Profile 4.1”；`64002a` 表示
`High@L4.2`。Chrome 是否能接收 L4.1/L4.2 取决于操作系统、硬件解码器、驱动和
WebRTC 构建。`RTCRtpReceiver.getCapabilities()` 返回的默认 Level 不是可靠的发送端
最大 Level，尤其在 `level-asymmetry-allowed=1` 时更不能这样解释。

本次修复的判断顺序是：

1. Web 上报 packetization-mode=1 的 Profile/Level 和 Level Asymmetry 能力；
2. CAE 按分辨率/帧率配置编码器，并从首个实际 SPS 读取 Profile/Level；
3. Offer 使用实际 SPS 的 ID；
4. Answer 仅在 Profile 兼容、且 Level 非对称规则满足时接受；
5. 最后用 `decoded`、`fps`、`pli` 和 `m=video` 端口确认运行时结果。

### 2.5 浏览器统计

浏览器已经成功建立视频轨道，并非 SDP m-line 被拒绝：

```text
connectionState=connected
RTP video: decoded>0, dropped=0, lost=0
```

但统计中出现低解码帧率和 PLI：

```text
fps=2.2 ... pli=2
fps=5.2 ... pli=2
```

这不是“Chrome 不支持 High Profile”的直接证据。SDP/实际 SPS 不一致、编码器输出帧率抖动、关键帧恢复和 BWE 调整都可能影响该指标，应先消除确定性的 Profile-Level 错配再判断浏览器性能。

---

## 3. Level 负载计算

H.264 Level 至少受以下两项约束：

- `MaxFS`：单帧最大宏块数；
- `MaxMBPS`：每秒最大宏块数。

当前输出为 `720x1560@60fps`：

```text
PicWidthInMbs  = ceil(720 / 16)  = 45
FrameHeightMbs = ceil(1560 / 16) = 98
FS             = 45 * 98         = 4,410 MB/frame
MBPS           = 4,410 * 60      = 264,600 MB/s
```

常用 Level 限制：

| Level | `level_idc` | MaxFS | MaxMBPS | 当前负载 |
|---|---:|---:|---:|---|
| 3.1 | 31 | 3,600 | 108,000 | 单帧和宏块率都超限 |
| 4.0 | 40 | 8,192 | 245,760 | 60fps 宏块率超限 |
| 4.1 | 41 | 8,192 | 245,760 | 60fps 宏块率超限 |
| 4.2 | 42 | 8,704 | 522,240 | 满足 |

因此：

- `720x1560@60fps` 最低需要 Level 4.2；
- `720x1560@30fps` 可落在 Level 4.0/4.1 范围；
- Level 3.1 即使降低到 30fps，当前单帧宏块数仍超过 MaxFS，需要同步降低分辨率，例如不高于约 `720x1280@30fps`。

---

## 4. 代码根因

### 4.1 Web 把 capability Level 当成最大接收 Level

Web 当前从 `RTCRtpReceiver.getCapabilities('video')` 收集 `profile-level-id`，但没有把 `level-asymmetry-allowed` 单独上报。CAE 因而只能把 `64001f` 当成完整上限。

影响：

- Chrome 明明允许 Level Asymmetry，CAE 仍把编码器请求降到 L3.1；
- 多客户端门禁要求新客户端 capability Level 大于等于当前 Level，可能错误拒绝支持同一 Profile 且允许非对称 Level 的 Chrome；
- 浏览器 Answer 返回 L3.1 时，运维容易误判发送码流也必须是 L3.1。

### 4.2 CAE 使用“请求值”生成 SDP

`WebRtcServerTransport` 从浏览器 capability 选择 `negotiated_h264_profile_level_id`，再把该值同时用于：

1. `CaeConfigManage::SetRuntimeH264LevelOverride()`；
2. Android `MediaFormat.KEY_LEVEL`；
3. SDP `profile-level-id`。

该实现隐含假设：硬件编码器严格服从 `KEY_LEVEL`。设备日志已证明该假设不成立。

### 4.3 SPS 校验只有日志，没有控制面反馈

`ScreenCapture` 已能解析 SPS 并输出：

```text
Encoder SPS profile-level-id=...
```

但解析结果没有传回 `WebRtcServerTransport`，也不会阻止错误 Offer。于是 CAE 可以在已经发现 mismatch 后继续声明旧 Level。

### 4.4 共享流门禁使用会话声明值，不是编码器实际值

当前共享编码器锁读取首个 `PeerSession.requestContext.negotiated_h264_profile_level_id`。如果首个会话声明值错误，后续客户端门禁、SDP 和发送链路都会继承错误值。

---

## 5. 修复设计

### 5.1 Web capability 扩展

WebRTC Request 新增：

```json
{
  "supported_h264_profile_level_ids": ["64001f", "4d001f", "42e01f"],
  "h264_level_asymmetry_profile_level_ids": ["64001f", "4d001f", "42e01f"]
}
```

第二个数组只包含同时满足以下条件的 capability：

- `video/H264`；
- `packetization-mode=1`；
- `level-asymmetry-allowed=1`；
- 存在合法 `profile-level-id`。

日志增加：

```text
H.264 level asymmetry ids=64001f/4d001f/42e01f
```

### 5.2 CAE 最低 Level 计算

CAE 在创建第一个 H.264 PeerConnection 前读取：

- `stream_width`；
- `stream_height`；
- `fps`。

计算当前负载所需的最低 `level_idc`，并把该值作为编码器目标，而不是直接使用浏览器 capability 中的 `1f`。

对于 `720x1560@60fps`：

```text
required_level_idc=42
```

### 5.3 实际 SPS 状态

`CaeEngineControl` 新增线程安全状态：

```text
actual_h264_profile_level_id
codec_generation
condition_variable
```

行为：

1. 每次打开或重启 H.264 编码器前清空状态并增加 generation；
2. 视频 callback 从 codec config 或含 SPS 的 IDR 中解析实际值；
3. 首次获得合法值后保存并唤醒等待者；
4. `WebRtcServerTransport` 在创建 Offer 前最多等待 2000ms；
5. 超时则拒绝 Offer，不能继续使用猜测值。

### 5.4 Offer 选择规则

第一个会话：

```text
浏览器支持的 Profile
  ∩ CAE 配置允许的最高 Profile
  → 计算 required Level
  → 配置/重启编码器
  → 等待实际 SPS
  → 验证实际 Profile 与浏览器 Profile 兼容
  → SDP 使用实际 SPS profile-level-id
```

如果浏览器对该 Profile 允许 Level Asymmetry：

- 不要求 capability 中的 `level_idc` 大于等于实际 SPS Level；
- Offer 使用实际 SPS Level；
- Answer 可以返回浏览器自己的较低 Level；
- CAE 必须记录 `offer_actual` 与 `answer_receiver`，不能把 Answer Level 写回共享编码器。

如果不允许 Level Asymmetry：

- 浏览器声明 Level 必须大于等于实际 SPS Level；
- 否则尝试下一个共同 Profile或拒绝。

### 5.5 Profile 降级

硬件可能忽略 Constrained High 等 Profile 约束。实际 SPS 与浏览器能力不兼容时按以下顺序尝试：

```text
High → Main → Baseline
```

每个候选都必须：

1. 浏览器明确广告该 Profile；
2. 重启编码器后重新读取 SPS；
3. SPS Profile 与浏览器 capability 兼容；
4. SDP 使用该次重启得到的实际值。

最多尝试每个 Profile 一次，避免无限重启。

### 5.6 多客户端共享流

已有会话时禁止重启或切换编码器。新客户端按当前实际 SPS 判断：

- Profile 不兼容：拒绝；
- Profile 兼容且该 capability 允许 Level Asymmetry：允许加入；
- Profile 兼容但不允许非对称，且客户端 Level 低于实际 Level：拒绝；
- 错误提示必须包含当前实际 `profile-level-id`。

### 5.7 日志口径

旧日志：

```text
encoder_level=31
```

该字段实际表示请求值，容易误导。修复后使用：

```text
configured_level_idc=42
required_level_idc=42
actual_sps_profile_level_id=64002a
offer_profile_level_id=64002a
answer_profile_level_id=64001f
level_asymmetry=1
```

---

## 6. 错误处理

| 场景 | 行为 |
|---|---|
| 2000ms 内没有 SPS | 返回 `CAE_VIDEO_NEGOTIATION_FAILED`，不创建 Offer |
| 实际 Profile 与候选不兼容 | 尝试下一个共同 Profile |
| 所有 Profile 均失败 | 返回 `CAE_H264_PROFILE_NOT_SUPPORTED` |
| 非对称关闭且实际 Level 过高 | 返回 `CAE_H264_PROFILE_NOT_SUPPORTED`，提示所需实际 ID |
| 已有共享流且新客户端不兼容 | 拒绝新客户端，不影响已有会话 |
| Answer Profile 与 Offer 不兼容 | 拒绝 Answer并关闭新 PeerConnection |
| Answer 仅 Level 不同且允许非对称 | 接受并记录双向 Level |

---

## 7. 验收标准

### 7.1 Chrome

在 `720x1560@60fps` 设备上应看到（Chrome 145/Linux 实测）：

```text
required_level_idc=42
actual_sps_profile_level_id=64002a
offer_profile_level_id=4d002a
answer_profile_level_id=4d001f
level_asymmetry=1
```

并满足：

- Answer 的 `m=video` 端口非 0；
- 视频持续解码；
- CAE 不再输出 SDP/SPS mismatch；
- 新 Chrome 客户端不会只因 capability 写着 L3.1 被错误拒绝。

实测结果：

```text
connectionState=connected
video=720x1560 readyState=4 paused=false
RTP video: decoded>0 dropped=0 lost=0
```

该实测验证的是 Chrome 的 Main@L4.2 共同 Profile。Chrome 是否支持 High@L4.1
必须在实际广告 `64xxxx` capability、且编码负载真实满足 L4.1（例如 30fps）时单独
验证；不能用当前 60fps 码流伪造 L4.1。

### 7.2 Safari

- 如果硬件能输出 Safari 广告的 High/Constrained High，则保持最高共同 Profile；
- 如果硬件实际 Profile 不兼容，则自动降到 Main 或 Baseline；
- 不允许继续用 `640c1f` 宣告实际 `64002a` 码流；
- Low Latency SDP、`playout-delay` 协商和 `playoutDelayHint=0` 行为保持不变。

### 7.3 Level 4.1 专项

验证 High@L4.1 必须使用真实符合 L4.1 的码流，例如 `720x1560@30fps`：

1. SPS 必须为 `640029` 或兼容的 High@L4.1；
2. SDP Offer 必须与 SPS 一致；
3. Chrome Answer 视频 m-line 非 0；
4. 连续统计中 decoded FPS 与编码 FPS 接近；
5. 不允许通过只改 SDP 的方式把 L4.2 码流伪装成 L4.1。

---

## 8. 非目标与后续项

本次 P0 不实现：

- 同时维护 H.264 多 Profile/多 Level 编码器；
- 针对每个观看端转码；
- 通过修改 SPS 字节伪造较低 Level；
- 根据 Chrome 版本或 User-Agent 强行开启 L4.1/L4.2；
- 把浏览器文件播放能力当作 WebRTC 接收能力。

后续可增加：

1. 编码器能力缓存：记录设备型号、codec name、分辨率/fps 与实际 SPS 的映射；
2. 当浏览器不允许当前 Level 时，自动选择 `30fps` 或低分辨率档位；
3. H.264 SPS parser 和 Level calculator 的独立单元测试；
4. Chrome/Safari/Edge、macOS/iOS/Android/Windows 的 Profile-Level 兼容矩阵。

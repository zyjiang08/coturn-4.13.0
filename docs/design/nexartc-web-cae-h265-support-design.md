# nexartc Web / CAE H.265（HEVC）支持详细设计

> 日期：2026-07-23
>
> 状态：Phase 0 核心链路已实施，Phase 1 RTP/SDP 加固待继续
>
> 适用范围：`nexartc-cloudPhoneAccess-web/device/`、`nexartc-cloud-phone-access-engine/` WebRTC 媒体链路
>
> 关联文档：
> - [`nexartc-turn-mode-a-implementation.md`](./nexartc-turn-mode-a-implementation.md)
> - [`nexartc-webrtc-ice-udp-mux-logical-separation.md`](./nexartc-webrtc-ice-udp-mux-logical-separation.md)
> - [`nexartc-install-deploy-guide.md`](./nexartc-install-deploy-guide.md)
> - [`nexartc-logging-design.md`](./nexartc-logging-design.md)
> - [`cae-stun-public-ip-discovery.md`](./cae-stun-public-ip-discovery.md)
> - [`nexartc-cae-scene-change-blur-qp-design.md`](./nexartc-cae-scene-change-blur-qp-design.md)（QP / 码率与「低延迟」分层）

---

## 1. 结论摘要

当前仓库已经具备 H.265 的主要“骨架”：

1. Web 可通过 `?codec=h265` 在 `CMD_START` 中发送 `frame_type=h265`；
2. CAE 可选择 Android `MediaCodec` 的 `video/hevc` 编码器；
3. `WebRtcServerTransport` 可生成 `H265/90000` SDP，并使用 `H265RtpPacketizer`；
4. 视频链路已识别 VPS/SPS/PPS 和 H.265 IRAP NAL；
5. PLI、NACK、REMB、共享编码 fan-out、ICE UDP Mux 均可复用现有机制。

但现状仍不能定义为“可生产发布的 H.265 支持”。关键原因是：

- Web 没有在请求 H.265 前检查 `RTCRtpReceiver` 的真实能力；
- CAE 把 H.265 硬件能力固定写成支持，没有查询设备上的真实 `MediaCodec`；
- `ProcessMedia()` 无论请求 H.264 还是 H.265 都执行 `IsSupportH265()` 门禁；真实探测上线后，无 HEVC 设备会误伤纯 H.264 START；
- 多客户端会直接改写全局 `CaeMediaConfig::m_videoFrameType`，可能造成“编码器实际 H.265、SDP/packetizer 已切回 H.264”的严重错配；
- `WebRtcServerTransport` 还通过进程静态 `s_isH265Codec` 决定 NAL/Send 语义，错配范围不只在 SDP；
- H.265 编码器被配置成 Main Profile + **High Tier Level 4**，但 SDP 没有声明对应 tier，兼容性不稳定；
- 编码器配置失败后的降级路径仍会重复加入同一组 HEVC profile/level 参数；
- codec config 使用函数静态缓存，H.264/H.265 切换时可能把旧 SPS/PPS/VPS 拼到新码流；
- Web 对 H.265 Offer 只做字符串日志，没有 Answer 选中 codec 校验和 H.264 自动回退；
- 缺少 H.265 RTP 分片、参数集、协商、多客户端和浏览器矩阵测试。

### 推荐方案

采用：

> **原生 WebRTC H.265 可选加速 + H.264 强制兜底 + 一路共享编码器 codec 锁定。**

具体原则：

1. H.265 不是 WebRTC 必选 codec，不允许仅凭浏览器版本或 User-Agent 决定；
2. Web 以 `RTCRtpReceiver.getCapabilities('video')` 为第一判据；
3. CAE 以真实 `MediaCodecList` 探测和实际 `configure()` 成功为最终判据；
4. 首个流媒体客户端决定共享编码器的 active codec；后续客户端只能复用该 codec，不能静默切换；
5. H.265 任一环节失败时，仅在不影响已有观看端的条件下自动回退 H.264；
6. 第一阶段不引入 WebCodecs + DataChannel 自定义视频传输，不维护两套拥塞控制和重传协议；
7. coturn、Signal Hub、ICE 策略和 UDP Mux 不需要 codec 专项改造，只补充 codec 字段透明转发和日志关联。

代码对照结论：**P0 判定成立**。Phase 0 是启用真实 capability 前的必要安全前置，不是可与 H.265 灰度并行补做的优化项。

---

## 2. 文档评审结果

### 2.1 `docs/design` 对 H.265 的覆盖情况

现有 `coturn-4.13.0/docs/design/` 文档主要解决：

| 文档 | 主要内容 | 与 H.265 的关系 |
|------|----------|-----------------|
| `turn-rest-api-signaling.md` | TURN REST、Hub、出站注册 | codec 无关；信令可透明携带新字段 |
| `nexartc-turn-mode-a-implementation.md` | Mode A、Hub、TURN、ICE | 仅要求 H.264 profile 回归，没有 H.265 协商设计 |
| `nexartc-webrtc-ice-udp-mux-logical-separation.md` | 单 UDP 端口、多 PC 生命周期 | codec 无关；共享编码器约束对 H.265 更重要 |
| `cae-stun-public-ip-discovery.md` | STUN 公网 IP | codec 无关 |
| `nexartc-install-deploy-guide.md` | 安装、配置、联调 | 没有 H.265 capability、fallback 和验收说明 |
| `nexartc-logging-design.md` | 四类日志、跨端排障 | 缺少 selected codec、encoder、RTP codec 关联字段 |
| `cae-supervisor-remote-admin-design.md` | CAE 自愈和远程管理 | 可扩展上报 encoder capability，但不是首期阻塞项 |
| `RTC_Token.zip` | 多语言 RTC Token 示例 | 与视频 codec 无关 |

结论：现有设计对网络面描述充分，但缺少一份 codec 协商和编码实现的设计真源，本文补齐该空缺。

### 2.2 现有 H.265 文档与代码的口径不一致

`nexartc-cloud-phone-access-engine/docs/bitrate_control_analysis.md` 已记录 2026-03-15 的 H.265 骨架实现，但存在以下过期信息：

| 项 | 旧口径 | 当前审查结论 |
|----|--------|--------------|
| Chrome 支持 | `Chrome 107+` | Chrome 107 是普通 HEVC 播放能力口径，不等于 WebRTC H.265；Chrome WebRTC H.265 从 M136 起按平台硬件能力启用，仍必须运行时探测 |
| Web 默认 codec | 文档称默认 H.265 | 当前 `device/src/main.ts` 默认 H.264，仅 `?codec=h265` 才请求 H.265 |
| CAE 默认 codec | 配置注释称默认 H.264 | APK assets 与 Settings 默认值实际偏向 H.265，部署口径冲突 |
| 设备 H.265 能力 | 认为 `support_h265=1` 即支持 | 当前开源 MediaEngine 固定返回 `H265:1`，没有真实探测 |
| 端到端完成度 | 记录为已支持 | 尚无能力协商、可靠回退、共享 codec 锁和浏览器矩阵测试 |

本文实施后，应同步修订上述旧文档、APK 配置注释、Settings 默认值和部署手册，避免继续把“HEVC 文件播放支持”误写成“WebRTC H.265 支持”。

---

## 3. 标准与浏览器能力边界

### 3.1 标准边界

- RFC 7742 要求 WebRTC 浏览器实现 VP8 和 H.264 Constrained Baseline；H.265 不属于强制 codec。
- RFC 7798 定义 H.265 RTP payload：`H265/90000`、单 NAL、Aggregation Packet、Fragmentation Unit 等。
- 因为 H.265 不是必选 codec，服务端不能假设所有 `RTCPeerConnection` 都能接收 H.265。
- H.265 涉及专利和授权，产品上线前需由法务确认终端分发、硬件编码和商业使用的许可边界。

### 3.2 浏览器现状

| 浏览器族 | 设计判断 | 实现要求 |
|----------|----------|----------|
| Chrome / Edge | Chrome Status 将 M136 标记为 WebRTC H.265 启用里程碑，但实际仍取决于平台硬件、OS、GPU、容器和启动参数 | 该版本号只作为工程观察，不是支持 SLA；必须查询 receiver capability |
| Chrome Android / WebView | 取决于系统 MediaCodec、设备硬件与 Chromium 构建 | 必须运行时查询；还需真机解码验证 |
| Safari / WebKit | 已存在 WebRTC H.265 实现，但仍受设备和系统约束 | 同样以 capability + SDP Answer 为准 |
| Firefox | H.265 不是可依赖的 WebRTC codec | 默认按不支持处理；若未来 capability 明确出现再允许 |

本机审查实测：Linux 上 Google Chrome 145 的
`RTCRtpReceiver.getCapabilities('video')` 未返回 `video/H265`，即使浏览器版本远高于 M136，也不能在该环境使用原生 WebRTC H.265。

因此，本文中的 M136 只描述 Chromium 功能发布节点，不构成“该版本及以上必然支持”的产品承诺、准入条件或兼容性 SLA。

### 3.3 Web 能力判定规则

判定优先级：

1. `RTCRtpReceiver.getCapabilities('video')` 是否包含 `video/H265` 或 `video/HEVC`；
2. 可选调用 `navigator.mediaCapabilities.decodingInfo({type:'webrtc', ...})` 获取 `smooth` / `powerEfficient`；
3. 最终以远端 Offer 能否 `setRemoteDescription()`、Answer 是否保留 H.265 payload 为准；
4. User-Agent 只用于日志和问题归类，不参与 codec 选择。

建议 Web 数据结构：

```ts
export type VideoCodecName = 'h264' | 'h265';
export type VideoCodecPreference = 'auto' | VideoCodecName;

export interface BrowserVideoCodecCapabilities {
  supportedVideoCodecs: VideoCodecName[];
  preferredVideoCodec: VideoCodecPreference;
  h265WebRtcReceive: boolean;
  h265Smooth?: boolean;
  h265PowerEfficient?: boolean;
  userAgent: string;
  isMobile: boolean;
}
```

能力检测必须大小写无关，并忽略 RTX/RED/ULPFEC 等非媒体 codec：

```ts
const codecs = RTCRtpReceiver.getCapabilities('video')?.codecs ?? [];
const h265WebRtcReceive = codecs.some((codec) => {
  const mime = codec.mimeType.toLowerCase();
  return mime === 'video/h265' || mime === 'video/hevc';
});
```

---

## 4. 当前端到端链路

```text
Web URL ?codec=h264|h265
  → main.ts getCodecParam()
  → WssConnection(codec)
  → protocol.ts makeStartMsg()
  → CMD_START media_config=frame_type=<codec>
  → CaeControlCmdHandleThread::ParseMediaConfig()
  → 全局 CaeMediaConfig::SetVideoFrameType()
  → CaeConnectionAgent::ProcessMedia()
  → CaeEngineControl::OpenMediaStream()
  → MediaEngine::OpenVideo(codecName=video/avc|video/hevc)
  → ScreenCapture / MediaCodec
  → JNI VideoFrameCallback
  → CaePipeManager / SendVideoDataHook
  → WebRtcServerTransport::Send()
  → H264RtpPacketizer 或 H265RtpPacketizer
  → 浏览器 RTCRtpReceiver
```

信令顺序是：

```text
CMD_START
  → CAE 已打开共享编码器
  → START_SUCCESS
  → Web 发送 WebRTC Request(type=1)
  → CAE 创建 PeerConnection + Offer
```

因此 codec 必须在 `CMD_START` 之前由 Web 完成能力判断，并由 CAE 在打开编码器前完成最终选择。不能等 Offer 到达后才发现浏览器不支持 H.265。

### 4.1 已有实现可复用项

| 模块 | 已有能力 | 处理意见 |
|------|----------|----------|
| Web START | `frame_type=h264|h265` | 保留并扩展 capability 字段 |
| CAE MediaCodec | `video/avc` / `video/hevc` | 保留，增加真实 capability 和渐进降级 |
| 参数集缓存 | SPS/PPS 缓存并前置到 IDR | 改成按 codec、按 encoder 实例的 VPS/SPS/PPS 缓存 |
| NAL 检测 | Annex-B / length-prefixed IDR 扫描 | 提取成统一 parser，补齐单测 |
| SDP | `addH264Codec()` / `addH265Codec()` | 增加 H.265 fmtp 和 Answer 校验 |
| RTP | `H265RtpPacketizer` | 保留，增加 RFC 7798 golden test |
| RTCP | NACK、PLI、REMB | codec 无关，直接复用 |
| Mode A MUX | 单端口多 PC | codec 无关，保留生命周期屏障 |

---

## 5. 现状问题与优先级

| 优先级 | 问题 | 风险 |
|--------|------|------|
| P0 | Web 不检查 H.265 receiver capability | 不支持的浏览器直接在 `setRemoteDescription` 失败 |
| P0 | `MediaEngine::GetMediaFeatures()` 固定返回 `H265:1` | 不支持 HEVC 的设备仍进入 H.265 启动路径 |
| P0 | `ProcessMedia()` 对所有 codec 无条件执行 `IsSupportH265()` | 改成真实能力后，无 HEVC 设备上的 H.264 START 也会失败 |
| P0 | `ParseMediaConfig()` 直接修改全局 codec | 第二个客户端可让共享编码器与 SDP/packetizer 错配 |
| P0 | `s_codecConfig` / `s_isH265Codec` 为进程静态状态 | codec 切换或并发连接会污染参数集、NAL 判断、Send 和 packetizer 语义 |
| P0 | H.265 无自动 H.264 fallback | 任一端能力误判即整次连接失败 |
| P0 | assets 与 Settings 默认 H.265，但运营配置强制 H.264 High | 预期测试 H.264 High 时可能实际启动 HEVC，发布口径失真 |
| P0 | `CAE_H265_NOT_SUPPORTED` 未加入 `CaeMsgCode.cpp` 消息表 | 错误码已发出但客户端拿不到稳定错误文案 |
| P1 | HEVC 配置使用 High Tier Level 4，SDP 未声明 | 部分硬件/浏览器只接受 Main Tier，码流与 SDP 语义不一致 |
| P1 | configure fallback 会再次设置同一 HEVC level | “去掉 profile 重试”实际上没有去掉 H.265 profile/level |
| P1 | Offer 只加 `H265/90000`，Answer 未校验 | 可能在浏览器拒绝 codec 后仍继续发送 H.265 RTP |
| P1 | Web Stats 不输出实际 codec | `decoded=0` 时无法快速判断 codec 错配 |
| P1 | H.265 packetizer 没有仓库级测试 | FU 分片、NAL header、marker bit 回归无法发现 |
| P2 | H.264/H.265 共用一套默认码率口径 | 无法稳定体现 H.265 的带宽收益 |
| P2 | 旧文档和配置默认值互相冲突 | 部署人员无法判断实际 codec 策略 |

---

## 6. 设计目标与非目标

### 6.1 目标

1. Web 支持 `auto`、`h264`、`h265` 三种偏好；生产默认具备安全回退。
2. 浏览器只有明确广告 H.265 receiver capability 时才可请求 H.265。
3. CAE 只有真实硬件编码能力满足当前分辨率/帧率时才选择 H.265。
4. H.265 采用 Main Profile、Main Tier、低延迟单层码流，并与 SDP 一致。
5. 编码输出统一为可验证的 Annex-B access unit，再交给 RTP packetizer。
6. 首个客户端锁定共享编码器 codec；多客户端不允许静默切换或混发。
7. H.265 失败时自动或显式回退 H.264，且最多重试一次，避免循环。
8. host / hybrid / p2p / relay 和 UDP Mux 行为与 codec 解耦。
9. 日志能够从 Web capability 一直关联到 CAE encoder、SDP、RTP 和浏览器实际解码 codec。

### 6.2 非目标

- 不在一台 CAE 上同时维护 H.264 与 H.265 两路编码器。
- 不为不同观看端做实时转码。
- 不在首期使用 WebCodecs + DataChannel/WebSocket 自定义视频协议。
- 不修改 coturn 的媒体处理；TURN 只转发 SRTP/UDP 数据。
- 不改变现有 ICE UDP Mux、PublicIpResolver 和 Mode A host 端口设计。
- 不保证 H.265 在所有 Chrome、Safari、Android 和 WebView 环境可用。

---

## 7. 总体架构

```text
┌──────────────────────────────────────────────────────────────┐
│ Web device                                                   │
│  BrowserVideoCodecCapabilities                               │
│  preference(auto/h264/h265) → requested codec                │
│  START + WebRTC Request capability report                    │
└──────────────────────────────┬───────────────────────────────┘
                               │ WSS / Hub transparent relay
┌──────────────────────────────▼───────────────────────────────┐
│ CAE CodecCoordinator                                        │
│  per-client request + server capability + active stream lock │
│  → selected codec / fallback reason / reject                 │
└──────────────────────────────┬───────────────────────────────┘
                               │ immutable ActiveVideoCodec
┌──────────────────────────────▼───────────────────────────────┐
│ MediaCodec pipeline                                         │
│  capability probe → configure → runtime format               │
│  normalize AU → codec-scoped VPS/SPS/PPS cache → IRAP        │
└──────────────────────────────┬───────────────────────────────┘
                               │ Annex-B access unit
┌──────────────────────────────▼───────────────────────────────┐
│ WebRtcServerTransport                                       │
│  SDP(H264 or H265) → RTP packetizer → NACK/PLI/REMB          │
│  shared encoder fan-out → N PeerConnections                  │
└──────────────────────────────┬───────────────────────────────┘
                               │ SRTP over host / TURN
┌──────────────────────────────▼───────────────────────────────┐
│ Browser RTCRtpReceiver → hardware decoder → <video>          │
└──────────────────────────────────────────────────────────────┘
```

核心对象：

```cpp
enum class VideoCodec {
    H264,
    H265,
};

struct VideoCodecCapabilities {
    bool h264Encoder = true;
    bool h265Encoder = false;
    bool h265Hardware = false;
    bool h265SurfaceInput = false;
    bool h265Cbr = false;
    bool h265MainProfile = false;
    std::string h265EncoderName;
};

struct ActiveVideoCodecState {
    VideoCodec codec = VideoCodec::H264;
    std::string profile;
    std::string tier;
    int levelId = 0;
    std::string encoderName;
    uint64_t generation = 0;
};
```

`generation` 在每次编码器重建时递增，用于拒绝上一代 codec config 和异步帧，防止切换期间旧帧串入新会话。

---

## 8. Codec 选择策略

### 8.1 Web 本地选择

| 用户偏好 | 浏览器支持 H.265 | Web 请求 codec | 行为 |
|----------|------------------|----------------|------|
| `auto` | 是 | `h265` | CAE 可继续验证并回退 |
| `auto` | 否 | `h264` | 不尝试 H.265 |
| `h264` | 任意 | `h264` | 强制 H.264 |
| `h265` | 是 | `h265` | 显式请求 H.265 |
| `h265` | 否 | 不连接或按开关回退 | UI 明确提示，不允许静默伪装成功 |

建议产品行为：

- `auto`：允许回退，默认推荐；
- `h264`：永远兼容；
- `h265`：用于测试或明确要求 HEVC 的场景，默认不静默回退，便于发现环境不支持。

### 8.2 CAE 最终选择

```text
SelectCodec(clientRequest, serverCapabilities, activeStream):
  if activeStream exists:
      if client supports activeStream.codec:
          return activeStream.codec
      return ACTIVE_CODEC_CONFLICT

  if request == h265:
      if client supports h265 && server supports h265:
          return h265
      if allowFallback && client supports h264 && server supports h264:
          return h264
      return H265_NOT_SUPPORTED

  return h264
```

### 8.3 多客户端 codec 锁

当前系统是“一路 MediaCodec 编码，N 路 PeerConnection fan-out”，因此：

- `activeCodec` 的生命周期必须与共享编码器一致；
- 首个 streaming client 打开编码器时锁定 codec；
- 后续客户端不能通过 `CMD_START` 修改全局 codec；
- active H.264 时，支持 H.264 的 H.265 客户端应复用 H.264；
- active H.265 时，不支持 H.265 的客户端应返回 codec conflict，不得重启编码器影响已有客户端；
- 最后一个 streaming client 离开并关闭 MediaCodec 后，codec 锁才释放。

推荐默认策略是“已有会话优先”，不做全局自动降级。若未来产品要求“新来的 H.264-only 客户端优先”，必须单独设计全体 PC 重新协商和编码器无缝切换，不属于本文首期范围。

---

## 9. 信令与协议扩展

### 9.1 兼容原则

- 保留现有 `frame_type=h264|h265`，旧 CAE 仍可识别；
- 新字段使用现有 `media_config` 子参数格式追加；
- Hub 只透明转发二进制帧和 WebRTC JSON，无需理解 codec；
- 新 CAE 遇到旧 Web 时，按 `frame_type` + 默认 H.264 capability 处理；
- 新 Web 遇到旧 CAE 时，若没有收到 selected codec 字段，以 SDP 为最终判据。

### 9.2 `CMD_START` 扩展

Web 在连接前完成 capability 检测，发送：

```text
media_config=
  frame_type=h265:
  preferred_video_codec=auto:
  supported_video_codecs=h264,h265:
  codec_fallback=1:
  definition=HD
```

字段：

| 字段 | 必填 | 说明 |
|------|------|------|
| `frame_type` | 是 | Web 本地已解析后的请求 codec，保持旧协议兼容 |
| `preferred_video_codec` | 否 | 原始用户偏好：auto/h264/h265 |
| `supported_video_codecs` | 否 | receiver 真实能力；旧客户端缺失时按只支持 H.264 处理更安全 |
| `codec_fallback` | 否 | 1=CAE 可从 H.265 降到 H.264；显式 H.265 模式建议为 0 |

这些字段必须先写入当前连接的 `CaeParamStorage`，不能在 parser 阶段直接修改全局 `CaeMediaConfig`。

### 9.3 `START_SUCCESS` 扩展

`BuildResponseVideoJsonConfig()` 返回实际选中的 codec：

```json
{
  "frame_type": "h265",
  "selected_video_codec": "h265",
  "codec_fallback": false,
  "codec_fallback_reason": "",
  "codec_profile": "main",
  "codec_tier": "main",
  "codec_level_id": 120,
  "encoder_name": "c2.vendor.hevc.encoder",
  "stream_width": 720,
  "stream_height": 1600
}
```

Web 必须保存 `selected_video_codec`，后续用于 Offer 校验、UI、Stats 和失败回退。

### 9.4 WebRTC Request（type=1）扩展

```json
{
  "type": 1,
  "preferred_video_codec": "auto",
  "requested_video_codec": "h265",
  "supported_video_codecs": ["h264", "h265"],
  "preferred_h264_profile": "high",
  "supported_h264_profiles": ["baseline", "main", "high"],
  "user_agent": "...",
  "is_mobile": false
}
```

CAE 处理 type=1 时应交叉检查：

- `requested_video_codec` 是否与 START 阶段记录一致；
- `selected_video_codec` 是否仍是 active codec；
- H.264 时才进入现有 H.264 profile 协商；
- H.265 时忽略 H.264 profile 字段。

### 9.5 Offer（type=2）扩展

```json
{
  "type": 2,
  "sdp": "...",
  "video_codec": "h265",
  "codec_profile": "main",
  "codec_tier": "main",
  "codec_level_id": 120
}
```

H.264 Offer 可继续保留 `h264_profile` 兼容字段。

### 9.6 错误码

保留现有：

- `CAE_H265_NOT_SUPPORTED = 0x1102`
- `CAE_H264_PROFILE_NOT_SUPPORTED = 0x1103`

建议按未占用值新增：

| 建议值 | 名称 | 含义 |
|--------|------|------|
| `0x1104` | `CAE_VIDEO_CODEC_NOT_SUPPORTED` | 请求 codec 不在客户端或服务端能力交集 |
| `0x1105` | `CAE_ACTIVE_VIDEO_CODEC_CONFLICT` | 当前共享编码器 codec 与新客户端能力冲突 |
| `0x1106` | `CAE_VIDEO_NEGOTIATION_FAILED` | SDP Answer 未接受已选 codec |

同时补齐 `CaeMsgCode.cpp` 中现有 `CAE_H265_NOT_SUPPORTED` 的消息映射。

当 active codec 为 H.265，而新用户浏览器上报的 `supported_video_codecs` 不包含 H.265 时，CAE 必须在 START 阶段返回 `0x1105`，不能返回泛化的 `0x1102`，也不能重启共享编码器。建议错误扩展字段：

```json
{
  "code": 4357,
  "reason": "active_codec_not_supported",
  "active_video_codec": "h265",
  "client_supported_video_codecs": ["h264"],
  "retry_after_session_end": true,
  "message": "Active H.265 stream is not supported by this browser"
}
```

其中 `4357` 为 `0x1105` 的十进制值。旧协议若不能携带扩展 JSON，至少保证错误码和 `CaeMsgCode` 文案稳定；Web 中文文案由错误码和字段本地化生成，不依赖服务端英文字符串匹配。

---

## 10. CAE 详细实现

### 10.1 真实 MediaCodec 能力探测

新增建议：

```text
app/src/main/java/com/nexartc/cae/media/MediaCodecCapabilityProbe.java
app/src/main/cpp/cae_media/VideoCodecCapabilities.h/.cpp
```

Java 侧使用 `MediaCodecList(MediaCodecList.ALL_CODECS)` 枚举 encoder，并检查：

1. `codecInfo.isEncoder()`；
2. 支持 `MediaFormat.MIMETYPE_VIDEO_HEVC`；
3. color format 包含 `COLOR_FormatSurface`；
4. 当前 `encode_width × encode_height × fps` 被 `VideoCapabilities` 支持；
5. profileLevels 至少包含 HEVC Main Profile；
6. CBR 能力是否存在；
7. API 29+ 记录 `isHardwareAccelerated()` / `isSoftwareOnly()`；
8. 记录 encoder name、最大尺寸、帧率、bitrate range、profile/level。

能力探测结果应在 MediaEngine 初始化完成后缓存，并通过 C++ 返回。安全默认值：

- H.264：true；
- H.265：false；
- 探测异常：H.265=false，不再使用当前“异常时默认 H.265=true”的策略。

最终能力仍以 `MediaCodec.configure()` 为准，因为部分厂商 capability 表不准确。

### 10.2 配置语义

建议统一 `[video]`：

```ini
[video]
# h264 | h265 | auto
# 第一阶段生产保持 h264；完成灰度后可改 auto
video_codec=h264

# 是否允许客户端能力覆盖 auto；显式 h264 时永远不启用 h265
video_codec_allow_client_override=1

# auto/h265 请求失败时是否可回退 h264
video_codec_fallback=1

# HEVC 固定为 Main Profile + Main Tier，降低 WebRTC 兼容风险
h265_profile=main
h265_tier=main
h265_level_id=120

# H.265 初始目标码率可按 H.264 等效值的 70% 起步，最终仍受 REMB 控制
h265_bitrate_ratio=0.70
```

实施期必须统一：

- APK assets 默认值；
- Settings Activity 默认值；
- `EngineConfigPatcher`；
- `test/device_runtime_config.ini`；
- 安装部署手册。

当前 `CaeConfig.ini` 的 `video_codec=h265`、`CaeSettingsActivity` 的缺省/非法值归一化为 `h265`，但同一配置又启用了 `h264_profile=2` 和 `force_high_profile=1`。Phase 0 必须先把生产默认统一为 `h264`，与现行 H.264 High 运营策略一致；`auto` 只能在 capability、锁和 fallback 闭环完成后灰度启用。

升级场景还要定义配置优先级和旧 SharedPreferences 迁移：由历史默认值写入的 H.265 不应继续覆盖新的生产安全默认；管理员明确选择的 H.265 则保留，并在能力不足时给出明确错误或按策略回退。

### 10.3 请求参数必须改为 per-connection

`CaeParamStorage` 增加：

```cpp
std::string requestedVideoCodec;
std::string preferredVideoCodec;
std::set<std::string> supportedVideoCodecs;
bool codecFallbackAllowed = true;
std::string selectedVideoCodec;
std::string codecFallbackReason;
```

`CaeControlCmdHandleThread::ParseMediaConfig()`：

- 只校验并写入 `m_paramStorage`；
- 不再立即调用 `CaeMediaConfig::SetVideoFrameType()`；
- 非法值直接返回参数错误；
- 旧客户端只传 `frame_type` 时，补出安全的 capability 集合。

这一步是修复多客户端错配的 P0 前置条件。

### 10.4 CodecCoordinator

建议在 `CaeConnectionAgent` 内封装或新增：

```text
cae_service/VideoCodecCoordinator.h/.cpp
```

职责：

- 维护 active codec、generation、encoder capability；
- 在 `ProcessMedia()` 前完成选择；
- 仅当当前连接请求且最终选中 H.265 时检查 `IsSupportH265()`；H.264 路径不得依赖 HEVC capability；
- 首客户端打开编码器时设置 `CaeMediaConfig`；
- 后续客户端只验证兼容，不改全局 codec；
- 最后一个流媒体客户端关闭后释放 active codec；
- 向 START_SUCCESS 和 WebRTC Request 提供 immutable codec snapshot；
- 将同一 snapshot/generation 传给编码回调、NAL parser、参数集缓存、`SetupMediaTracks()`、RTP packetizer 和 `Send()`，禁止这些路径重新读取可变全局 frame type。

CodecCoordinator 不能只锁住 `CaeMediaConfig`。现有 `s_isH265Codec` 同时参与 NAL 类型、关键帧和 Send 路径判断，必须删除并改成 encoder/session 实例状态；`s_codecConfig` 必须归属 encoder generation。任何 session 的 codec 或 generation 与 active snapshot 不一致时立即拒绝发送，而不是尝试沿用当前静态标志。

锁顺序必须固定：

```text
codec coordinator mutex
  → media stream lifecycle mutex
  → WebRTC session mutex
```

不得在持有 `WebRtcServerTransport::m_mutex` 时重启 MediaCodec，继续遵守现有 H.264 profile 重启路径的死锁约束。

### 10.5 MediaCodec H.265 配置

首选配置：

```text
MIME                  video/hevc
Profile               HEVC Main
Tier                   Main Tier
Level                  4.0（level-id=120，按分辨率/fps 可调整）
Input                  COLOR_FormatSurface
Bitrate mode           CBR（设备不支持时允许 VBR，但要记录）
B frames               0
I-frame interval       与 WebRTC 配置一致
Intra refresh          默认关闭，避免与定期 IRAP 冲突
Parameter sets         每个 IRAP 前保证 VPS/SPS/PPS in-band
Encoder low-latency    厂商/API 支持时启用（见 §10.5.1），失败时去掉该参数重试
```

当前 `HEVCHighTierLevel4` 应改为 Main Tier。High Tier 不是云手机低延迟场景的必要条件，还会缩小 decoder 兼容面。

#### 10.5.1 「低延迟」分层与编码器快速出帧（现状）

产品/联调中「低延迟」易与 WebRTC 配置项混淆。本项目至少分三层：

| 层级 | 含义 | 配置 / 代码 | H.265 现状 |
|------|------|-------------|------------|
| A. 播放侧 / SDP | 降低浏览器抖动缓冲（jbDelay） | `webrtc_low_latency`、去 LS、`playout-delay`、`playoutDelayHint=0` | 与 codec 无关；**已实现** |
| B. 采集侧 | 少等 VSync | `Surface.setFrameRate`（API 30+） | H.264/H.265 共用；**已实现** |
| C. 编码器「快速出帧」 | MediaCodec 少缓冲、尽快产出 AU | `KEY_LATENCY` / `KEY_PRIORITY` / `FEATURE_LowLatency` / 厂商 key 等 | **未实现**（H.264 同样未开） |

要点：

1. **`webrtc_low_latency` 只驱动层级 A**，不会打开层级 C；不能据此认为「H.265 已开编码低延迟」。
2. H.265 与 H.264 共用 `ScreenCapture.buildVideoFormat()` / `configure`，**没有**独立的 HEVC encoder-low-latency 开关。
3. **QP（`qp_i_*` / `qp_p_*`）与层级 C 正交**：QP 管质量地板（见场景切换模糊文档）；C 管编码流水线时延。两者均对 H.264/H.265 共用配置键。

**层级 C — 当前已有（偏实时，但非完整 low-latency）：**

| 设置 | 说明 |
|------|------|
| `BITRATE_MODE_CBR` | 码率受控 |
| `KEY_REPEAT_PREVIOUS_FRAME_AFTER` | 无新帧时重复，避免饿死 |
| Surface `setFrameRate` | 采集侧绕开部分 VSync（约省 4–8ms） |
| 未主动开 B 帧 | 仅在设置 QP 时顺带写 B 的 QP key |

**层级 C — 当前缺失（「快速出帧」应对齐的目标）：**

| 参数 | 作用 | 现状 |
|------|------|------|
| `MediaFormat.KEY_LATENCY`（API 30+） | 限制编码流水线延迟帧数 | 未设 |
| `KEY_PRIORITY` = 0（realtime） | 实时优先级 | 未设 |
| `KEY_OPERATING_RATE` | 提高编码吞吐 | 未设 |
| `FEATURE_LowLatency` / 厂商 low-latency | 低延迟编码通路 | 未用 |

**实施建议（与 §10.6 降级一致）：**

- 新增可选配置（建议名 `encoder_low_latency`，默认开启或与产品策略对齐），在 `buildVideoFormat()` 中写入上表参数；
- configure 失败则按 §10.6 去掉 latency/vendor 参数重试，保证 H.265/H.264 仍能起来；
- 日志区分：`webrtc_low_latency`（A）与 `encoder_low_latency`（C），避免排障混淆；
- 验收指标：编码端到 RTP 发送的帧时延 / encode time，**不是** jbDelay（jbDelay 只验证 A）。

关联：[`nexartc-cae-scene-change-blur-qp-design.md`](./nexartc-cae-scene-change-blur-qp-design.md) §3.6。

### 10.6 渐进式 configure 降级

H.265 configure 不能只做“一次完整参数 + 一次相同参数重试”。建议顺序：

1. Main/Main Tier/Level + CBR + **encoder** low-latency + QP + vendor 参数；
2. 去掉 QP 和 vendor 参数；
3. 保留 Main Profile，去掉显式 level/tier，让 encoder 选择；
4. 去掉 **encoder** low-latency，只保留核心参数；
5. 若允许 fallback，关闭 HEVC codec 实例，创建 H.264 encoder；
6. 每次失败记录失败阶段、encoder name、异常类型，不打印敏感信息。

说明：步骤中的 low-latency 指 **§10.5.1 层级 C（MediaCodec 快速出帧）**，不是 `webrtc_low_latency`（层级 A）。

只有 H.264 fallback 成功后才能发送 START_SUCCESS，并将实际 codec 返回 Web。

能力门禁和启动回退必须按当前选中 codec 分支：

```text
selected=H264
  → 跳过 IsSupportH265
  → 直接启动 H.264

selected=H265
  → 检查真实 HEVC capability
  → configure H.265
  → 失败且无既有 active session、允许 fallback
      → 关闭失败的 HEVC 实例
      → 清空该 generation 的参数集和异步帧
      → generation++，锁定 H.264，重新打开编码器
      → 成功后才返回 START_SUCCESS(selected=H264)
  → 已有 active H.265 session
      → 拒绝不兼容 newcomer，不重启共享编码器
```

### 10.7 编码输出标准化

新增统一 `EncodedVideoAccessUnit`：

```cpp
struct EncodedVideoAccessUnit {
    VideoCodec codec;
    uint64_t generation;
    int64_t ptsUs;
    bool randomAccess;
    bool codecConfigOnly;
    std::vector<uint8_t> annexB;
};
```

标准化规则：

1. 同时接受 3-byte / 4-byte Annex-B start code 和 4-byte length-prefixed NAL；
2. 在 MediaEngine 边界统一转换为 4-byte Annex-B；
3. H.265 参数集为 VPS=32、SPS=33、PPS=34；
4. H.265 random access 为 IDR_W_RADL=19、IDR_N_LP=20、CRA=21；
5. 参数集缓存属于当前 encoder 实例，不得使用函数静态变量；
6. codec 或 generation 变化时立即清空缓存；
7. 每个 random-access AU 若未携带完整 VPS/SPS/PPS，则按 VPS→SPS→PPS→IRAP 顺序前置；
8. codec-config-only AU 可更新缓存，但不作为普通画面发送；
9. 参数集不完整时不发送 IRAP，并重新请求关键帧，避免向浏览器发送不可解码起始帧。

当前 `s_codecConfig` 和 `s_isH265Codec` 都应移除或改为受锁保护的实例状态。

### 10.8 SDP 生成

H.265 最小推荐 SDP：

```sdp
m=video 9 UDP/TLS/RTP/SAVPF 96
a=rtpmap:96 H265/90000
a=fmtp:96 profile-id=1;tier-flag=0;level-id=120;tx-mode=SRST
a=rtcp-fb:96 nack
a=rtcp-fb:96 nack pli
a=rtcp-fb:96 goog-remb
```

约束：

- payload type 继续使用当前动态 PT 96；
- profile-id=1 表示 HEVC Main Profile；
- tier-flag=0 表示 Main Tier；
- tx-mode=SRST 对应单 RTP stream；
- 不依赖 SDP 中 `sprop-vps/sps/pps`，参数集走 in-band；
- SDP fmtp 必须来自实际 active codec descriptor，不能与 MediaCodec 配置分叉。

`rtc::Description::Video::addH265Codec()` 已支持 profile/fmtp 字符串，应使用带 fmtp 的重载。

### 10.9 Answer 校验

`HandleRemoteAnswer()` 增加 video m-section 解析：

1. 找到当前 video payload type；
2. selected codec=H.265 时，Answer 必须包含 `a=rtpmap:<pt> H265/90000`；
3. selected codec=H.264 时继续执行 profile-level-id 校验；
4. Answer 拒绝 video（port=0）或移除 H.265 时，不允许 Track 进入发送态；
5. 返回 `CAE_VIDEO_NEGOTIATION_FAILED` 并触发受控 fallback 或断开当前客户端。

不能仅使用 `sdp.find("H265")`，必须按 media section 和 payload type 解析，避免误匹配无关文本。

### 10.10 RTP packetization

现有 `H265RtpPacketizer` 可继续使用，但必须补验证：

- 输入固定为 Annex-B；
- 单 NAL 小于 MTU 时直接发送原 NAL；
- 大 NAL 使用 RFC 7798 FU type 49；
- FU indicator 保留 forbidden bit、layer id、temporal id；
- FU header 的 S/E 位只在首/尾分片设置；
- RTP marker 只出现在 access unit 最后一包；
- 同一 AU 所有 RTP packet 时间戳一致；
- PT、SSRC、clock rate 与 SDP 一致；
- SRTP/NACK cache 继续复用当前 2048 packet 配置。

在完成 packetizer golden test 前，不建议把 H.265 设为生产默认。

### 10.11 RTCP、码率和关键帧

NACK、PLI、REMB 对 codec 无特殊要求，但需调整命名和数据：

- `ForceRequestIframe()` 对 H.265 实际请求 IRAP；日志统一写 `keyframe/IRAP`；
- PLI 节流继续使用 per-session 300ms、global 1000ms；
- 新客户端 Track ready 后请求一次 IRAP；
- 本地 queue overflow 后等待 H.265 IRAP，不发送依赖旧参考帧的 P frame；
- H.265 可用 H.264 等效码率的约 60%–75% 作为初始值，但 REMB 输入仍是实际网络 bps，不应再重复乘比例；
- 建议增加 codec-specific initial/min/max 配置或在初始化时只缩放 nominal target。

---

## 11. Web 详细实现

### 11.1 Codec 模型统一

新增 `VideoCodecName` / `VideoCodecPreference`，不要继续把 H.265 作为 H.264 profile 逻辑的特殊分支。

建议文件职责：

| 文件 | 改动 |
|------|------|
| `device/src/webrtc.ts` | 通用 codec capability、SDP codec 解析、Answer 校验 |
| `device/src/main.ts` | codec preference、selected codec、一次性 fallback 状态机 |
| `device/src/protocol.ts` | START capability 字段 |
| `device/src/wss.ts` | 传递完整 codec request，不只传字符串 |
| `device/src/stats.ts` | requested/selected/negotiated codec 统计 |
| `device/index.html` | Auto/H.264/H.265 选择和 capability 提示 |

### 11.2 UI 与 URL

建议：

```text
?codec=auto   默认自动选择
?codec=h264   强制兼容模式
?codec=h265   强制 HEVC 验证模式
```

UI：

- 下拉框：Auto（推荐）、H.264、H.265；
- capability 不含 H.265 时，H.265 选项置灰；
- URL 强制 `h265` 但环境不支持时，显示明确错误；
- 当前共享流已锁定 H.265、但本浏览器不支持 H.265 时，显示阻断式提示：**“当前已有用户使用 H.265 观看，你的浏览器不支持 H.265。请更换支持 H.265 的浏览器，或等待当前观看会话结束后重试。”**；
- H.265 会话隐藏 H.264 profile 选择，badge 改为 `Codec: H265 Main`；
- H.264 会话保持现有 Base/Main/High UI。

该提示应保留“重试”和“关闭”操作，不显示自动回退成功状态，也不进入播放器空白页。页面可以展示 `active_video_codec=H.265`，但不得展示其他用户身份、连接 ID 或观看人数。

灰度期可继续让页面默认 H.264；完成测试后再把无参数默认改为 Auto。

### 11.3 START 前能力检测

`startTest()` 当前已在创建 `WssConnection` 前调用 `buildWebRtcRequest()`，可在同一阶段：

1. 读取 codec preference；
2. 查询 browser capability；
3. 解析本地 requested codec；
4. 生成 START codec request；
5. 再连接 Hub/WSS。

这样不需要新增额外网络 round trip。

### 11.4 START_SUCCESS 处理

Web 当前只读取 `stream_width` / `stream_height`，应同时读取：

- `selected_video_codec`；
- `codec_fallback`；
- `codec_fallback_reason`；
- `codec_profile/tier/level`；
- `encoder_name` 仅用于调试展示，不作为逻辑判据。

若 CAE 返回 H.265，但本地 capability 不含 H.265，Web 不应继续发 WebRTC Request，应立即停止并以协议错误上报。

### 11.5 Offer 校验

新增 SDP parser，按 video m-section 返回：

```ts
interface SdpVideoCodec {
  payloadType: number;
  name: 'H264' | 'H265' | string;
  clockRate: number;
  fmtp?: string;
}
```

检查：

- Offer codec 与 START_SUCCESS 的 selected codec 一致；
- H.265 clock rate 为 90000；
- H.265 profile/tier 不高于客户端支持范围；
- H.264 继续执行现有 profile 检查；
- `setRemoteDescription()` / `createAnswer()` / `setLocalDescription()` 分别捕获异常并标记失败阶段。

### 11.6 Answer 校验

Web 在发送 Answer 前确认：

- video m-section 未被拒绝；
- Answer 保留 selected codec 的 payload type；
- H.265 会话不误选 H.264；
- H.264 profile 回调只在 H.264 会话触发。

Offer 中出现字符串 `H265` 只表示服务端声称支持，不等于浏览器最终接受。

### 11.7 自动回退

Web 维护：

```ts
interface CodecFallbackState {
  attempted: boolean;
  initialCodec: VideoCodecName;
  reason?: string;
}
```

可触发回退的条件：

1. H.265 `setRemoteDescription()` 失败；
2. Answer 拒绝 H.265；
3. 已收到持续 RTP bytes，但 H.265 `framesDecoded=0` 且 PLI 持续增长；
4. CAE 明确返回 H.265 encoder/configure 不支持。

回退规则：

- 仅 `auto` 或 `codec_fallback=1` 可自动回退；
- 每次用户连接最多回退一次；
- 关闭旧 PC 和 WSS，重新以 H.264 START 建连；
- 记录原始失败阶段和 codec；
- 若 CAE 返回 active H.265 shared-stream conflict，则不允许为新客户端重启全局编码器，只向用户说明当前共享流为 H.265；
- 不能在同一个已生成 H.265 Offer 的 PC 上通过 SDP munging 强改 H.264，因为服务端 Track/packetizer/编码器仍是 H.265。

`CAE_ACTIVE_VIDEO_CODEC_CONFLICT` 是业务阻断而不是网络异常：Web 收到后必须停止 WSS 自动重连和 H.264 fallback，清理当前未完成的 PC/WSS，并展示 §11.2 的中文提示。用户主动点击“重试”时才重新发起 capability 检测和 START；CAE 不承诺当前 H.265 会话何时结束，因此不设置定时轮询。

Offer/Answer 阶段属于“晚失败”：此时 CAE 可能已经打开 H.265 编码器。处理必须区分：

1. **无既有 active WebRTC session**：失败连接先关闭 PC/WSS，CAE 释放该连接的流引用、关闭 HEVC 并进入 IDLE；新的 H.264 START 再递增 generation、锁定 H.264 并打开新编码器；
2. **已有 H.265 active session**：失败的新客户端只能返回 `CAE_ACTIVE_VIDEO_CODEC_CONFLICT` 或协商失败，不得触发全局降级、关闭编码器或中断已有观看端。

Web 只能发起一次新的 H.264 START，不能单方面修改 CAE 全局 codec。CAE 必须以 coordinator 中的 active session/refcount 为准决定“可回退”还是“拒绝 newcomer”。

### 11.8 Stats

当前 Stats 只输出 inbound RTP 码率、fps、丢包和 decode time。应通过 `inbound-rtp.codecId` 查找对应 `codec` report，并输出：

```text
[FLOW] video codec selected: H265/90000 pt=96 impl=hardware
[STAB] video health codec=H265 decoded=... lost=... pli=... fps=...
```

页面统计增加：

- requested codec；
- CAE selected codec；
- SDP negotiated codec；
- codec payload type；
- decoder implementation / power efficient（浏览器暴露时）；
- fallback count 和 reason。

---

## 12. 多客户端与 Mode A 约束

### 12.1 与 ICE UDP Mux 的关系

H.264/H.265 都通过同一个 `rtc::Track` / SRTP / ICE socket 发送，codec 不改变：

- UDP 50000 单端口；
- ufrag / 五元组 demux；
- `SendGate`；
- `m_rtpTeardownPause`；
- peersRemain 时同步 close；
- host / hybrid / relay candidate 选择。

因此本文不修改 [`nexartc-webrtc-ice-udp-mux-logical-separation.md`](./nexartc-webrtc-ice-udp-mux-logical-separation.md) 的生命周期方案。

### 12.2 与共享编码 fan-out 的关系

```text
one MediaCodec(H264 or H265)
  → one normalized AU
  → GetActiveConnIds()
  → N × WebRtcServerTransport::Send()
  → N × codec-matched RTP packetizer/session
```

所有 session 必须绑定同一个 active codec generation。连接建立后若发现 session codec 与 active generation 不一致，应拒绝发送并关闭该 session，不能“尽量发送”。

### 12.3 断连恢复

一端离开、其余继续时：

- active codec 不变；
- 编码器不重启；
- teardown 全局暂停结束后请求 H.264 IDR 或 H.265 IRAP；
- 剩余端 wait-keyframe gate 同时支持两种 NAL 类型；
- 最后一端离开才清理 codec config、active codec 和 generation。

---

## 13. 状态机与失败策略

### 13.1 CAE codec 状态机

```text
IDLE
  ├─ select H264 → STARTING_H264 → ACTIVE_H264
  └─ select H265 → STARTING_H265
                       ├─ success → ACTIVE_H265
                       └─ fail + fallback → STARTING_H264 → ACTIVE_H264

ACTIVE_H264 / ACTIVE_H265
  ├─ compatible join → keep active
  ├─ incompatible join → reject newcomer
  └─ last client leave → STOPPING → IDLE
```

`ACTIVE_*` 期间不接受动态 codec 修改命令。现有 SET_MEDIA_CONFIG 只能动态修改码率、静音等参数；codec 修改必须返回“需要重建媒体会话”。

### 13.2 失败矩阵

| 失败点 | 检测方 | 行为 |
|--------|--------|------|
| 浏览器 capability 无 H.265 | Web | auto 选 H.264；强制 H.265 提示错误 |
| CAE 无 HEVC encoder | CAE | 允许时回退 H.264，否则 0x1102 |
| H.264 START，但 HEVC capability=false | CAE | 跳过 H.265 门禁，正常启动 H.264 |
| HEVC configure 失败且无 active session | CAE | 渐进降级；允许时关闭 HEVC、递增 generation 后启动 H.264 |
| HEVC configure/协商失败且已有 H.265 session | CAE | 拒绝 newcomer，不改变共享编码器和已有会话 |
| 参数集不完整 | CAE | 暂停发送、请求 IRAP、超时后 fallback/报错 |
| Offer codec 与 selected codec 不同 | Web | 协议错误，断开 |
| 浏览器拒绝 H.265 SDP | Web + CAE | Answer 不发送或返回 negotiation failed；一次性 H.264 重连 |
| active H.265，新客户端仅 H.264 | CAE + Web | CAE 返回 `0x1105` 且不影响已有会话；Web 停止自动重连并提示“当前已有用户使用 H.265 观看，你的浏览器不支持 H.265” |
| packetsReceived>0 但 decoded=0 | Web | dump codec/PLI/SDP；满足阈值后一次性 fallback |
| H.265 RTP packetizer 异常 | CAE | 发送失败计数；触发 IRAP，持续失败则关闭 session |

---

## 14. 日志与可观测性

沿用 `[FLOW]` / `[FUNC]` / `[STAB]` / `[EXC]`。

### 14.1 Web

```text
[FLOW] codec capability: recv=[H264,H265] preference=auto requested=H265
[FLOW] START_SUCCESS selected_codec=H265 fallback=0 profile=main tier=main level=120
[FLOW] SDP offer codec=H265/90000 pt=96
[FLOW] SDP answer codec=H265/90000 pt=96
[FLOW] video codec selected: H265 codecId=...
[STAB] video health codec=H265 decoded=... lost=... pli=... fps=...
[EXC] H265 negotiation failed stage=setRemoteDescription ... fallback=H264 attempt=1
```

### 14.2 CAE

```text
[FLOW] codec request conn=... preferred=auto requested=h265 supported=h264,h265 fallback=1
[FUNC] codec capability h265=1 hw=1 surface=1 cbr=1 encoder=c2....
[FLOW] active codec selected=h265 generation=7 reason=first_stream
[FUNC] MediaCodec configured codec=h265 profile=main tier=main level=120 attempt=1
[FLOW] SDP offer conn=2000 codec=H265 pt=96 fmtp=...
[STAB] video AU codec=H265 gen=7 vps=1 sps=1 pps=1 irap=1 bytes=...
[EXC] codec conflict conn=... active=h265 client_supported=h264
```

禁止逐帧完整打印 NAL；保留前若干帧、IRAP、异常和周期采样即可。

### 14.3 Hub / coturn

- Hub 可在 join 审计日志记录客户端声明 codec，但不做选择；
- coturn 无需感知 codec；排障仍看 allocation、丢包和带宽；
- 所有日志至少带 `deviceId`、session/conn id、selected codec，便于跨端关联。

---

## 15. 测试方案

### 15.1 Web capability 与协商测试

新增建议：`test/test_h265_webrtc.mjs`，使用现有 Puppeteer 环境。

| 用例 | 期望 |
|------|------|
| receiver capability 无 H.265 + auto | START 请求 H.264 |
| receiver capability 有 H.265 + auto | START 请求 H.265 |
| 强制 H.264 | 无论 capability 如何均请求 H.264 |
| 强制 H.265但 capability 无 | 页面阻止或按明确开关回退 |
| H.265 Offer + H.265 capability | Answer 保留 H.265 PT |
| H.265 Offer + capability 无 | 在 setRemoteDescription 前后得到明确失败，不进入假连接 |
| START selected 与 Offer 不一致 | 协议错误 |
| H.265 协商失败 | 自动 H.264 重连恰好一次 |
| active H.265 + 新浏览器仅支持 H.264 | 收到 `0x1105`，显示指定中文提示，不自动 fallback、不循环重连 |
| Stats codecId | 输出 H264/H265 实际 codec |

本机 Chrome 145 Linux 应作为“版本高但无 H.265 capability”的固定回归环境。

### 15.2 CAE 单元测试

新增 host-side 测试建议：

```text
test/cae/test_video_codec_selection.cpp
test/cae/test_h26x_annexb_parser.cpp
test/cae/test_h265_rtp_packetizer.cpp
test/cae/test_codec_config_generation.cpp
```

覆盖：

1. codec selection 矩阵；
2. HEVC capability=false 时 H.264 START 成功且不返回 `0x1102`；
3. active codec 多客户端冲突；
4. 无 active session 时 H.265→H.264 fallback；
5. 已有 H.265 session 时 newcomer 失败但旧会话不断流；
6. Annex-B 3/4-byte 与 length-prefixed 转换；
7. H.265 VPS/SPS/PPS/IDR/CRA 识别；
8. H.264→H.265→H.264 时参数集缓存清空；
9. generation 变化时旧帧丢弃；
10. H.265 单 NAL RTP；
11. 大 NAL FU type 49 分片和重组；
12. FU header layer/temporal id 保真；
13. SDP fmtp 与 active codec descriptor 一致；
14. `CaeMsgCode::GetMsg(CAE_H265_NOT_SUPPORTED)` 返回稳定文案。

### 15.3 Android 真机测试

设备矩阵至少包括：

| 类型 | 要求 |
|------|------|
| H.265 硬编支持设备 | 验证 Main/Main Tier、720×1600@60、动态码率、IRAP |
| 无 H.265 或 capability 不完整设备 | 验证 H.264 fallback |
| Qualcomm | 验证 codec config、request-sync-frame |
| MediaTek | 验证 KEY_FRAME flag 缺失时 NAL 扫描 |
| root backend | SurfaceFlinger/VirtualDisplay 路径 |
| nonroot backend | MediaProjection 路径 |

每台设备记录：encoder name、capability、实际 output format、configure attempt、平均 encode time、码率、温度和功耗。

### 15.4 浏览器矩阵

| 环境 | 期望 |
|------|------|
| Chrome/Edge 支持硬件 HEVC | auto 走 H.265 |
| Chrome Linux 无 HEVC capability | auto 走 H.264 |
| Chrome Android 支持 HEVC | auto 走 H.265，真机硬解 |
| Safari 支持 HEVC | H.265 协商和播放 |
| Firefox 无 H.265 capability | auto 走 H.264 |
| WebView | 按 capability 动态选择 |

### 15.5 网络与多客户端矩阵

每种 codec 都执行：

- host；
- hybrid；
- relay；
- 2–3 客户端同时观看；
- 断一端、其余继续；
- 新客户端 codec 不兼容；
- 5%/10% 丢包下 NACK/PLI 恢复；
- 快速重连 10 次；
- 横竖屏切换；
- 动态 REMB 降码率和恢复。

### 15.6 验收指标

功能门禁：

1. selected codec 与 MediaCodec、SDP、RTP Stats 四处一致；
2. H.265 首个可解码帧前具备 VPS/SPS/PPS；
3. H.265 不支持环境自动回退 H.264，不出现无限重试；
4. 多客户端不能改变 active codec；
5. 一端断开不影响其余端；
6. H.264 全量回归通过。

性能建议门禁：

| 指标 | 目标 |
|------|------|
| 720×1600@60 编码稳定性 | 30 分钟无 encoder reset / crash |
| 解码帧率 | 稳态接近目标 fps，按设备能力分档 |
| 首帧 | 与 H.264 基线相比无明显退化，回退场景可接受一次重连 |
| 同画质带宽 | H.265 相比 H.264 有可重复的下降，目标区间 25%–40%，以实测为准 |
| PLI 恢复 | 收到 PLI 后在一个 IRAP 周期内恢复 |
| CAE 稳定性 | 无 pure virtual、UAF、codec config 串流 |

---

## 16. 分阶段实施

### Phase 0：安全护栏（必须先做）

1. Web capability 检测，M136 仅用于日志，不作为准入条件；
2. Web 与 CAE 生产默认统一为 H.264，与现行强制 H.264 High 策略对齐；
3. CAE 接入真实 MediaCodec capability，并修复 `IsSupportH265()` 无条件门禁，确保无 HEVC 设备仍能启动 H.264；
4. START 参数改为 per-connection，不再由 parser 直接改写全局 codec；
5. active codec 锁覆盖 encoder、SDP、packetizer、NAL/Send 和 session generation；
6. 删除/实例化 `s_isH265Codec`，codec config 改为 encoder 实例状态并增加 generation；
7. 明确 H.265 失败回退边界：无 active session 才可关闭并重开 H.264；已有 H.265 session 时拒绝 newcomer；
8. 补齐 `CAE_H265_NOT_SUPPORTED` 消息映射和 codec 冲突/协商错误码；
9. 修订 Settings、assets、配置迁移、日志和旧文档；
10. 增加“无 HEVC 的 H.264 回归”和“已有会话不被 fallback 中断”自动化用例。

完成前不得把 H.265 设为生产默认。

本次已落地的 Phase 0 核心实现：

- Web 在 `CMD_START` 的 `media_config` 中上报 `preferred/requested/supported_video_codecs` 和 `codec_fallback`；
- CAE 按连接保存 codec 请求，并在共享流打开后锁定 active codec；
- active H.265 且新浏览器不支持 H.265 时返回 `CAE_ACTIVE_VIDEO_CODEC_CONFLICT (0x1105)`，不重启已有编码器；
- Web 收到该错误后停止自动重连和 H.264 fallback，展示“当前已有用户使用 H.265 观看，你的浏览器不支持 H.265”并提供手动重试；
- CAE 通过 Android `MediaCodecList` 探测 HEVC 硬编码能力，生产默认恢复为 H.264；
- H.265 Main/Main Tier 配置和 codec-scoped callback 参数集缓存已接入。

### Phase 1：原生 WebRTC H.265 正式接通

1. Main/Main Tier MediaCodec 配置和渐进降级；
2. Annex-B 标准化；
3. H.265 fmtp；
4. Answer codec 校验；
5. RTP packetizer golden tests；
6. Web codec UI、Stats 和一次性 fallback；
7. Chrome/Safari/Android 真机矩阵。

### Phase 2：灰度与调优

1. 设备 capability 上报；
2. 按 device/browser/OS 灰度；
3. H.264/H.265 A/B 码率和画质对比；
4. codec-specific initial bitrate；
5. 温度、功耗、编码延迟监控；
6. 根据故障率决定 Auto 默认是否优先 H.265。

### Phase 3：可选 WebCodecs 兜底评估

只有在“浏览器不支持 WebRTC H.265，但业务强制要求 H.265”时再评估：

- CAE 通过专用 DataChannel/WebTransport 发送 Annex-B AU；
- WebCodecs `VideoDecoder` 解码并渲染 canvas；
- 自行处理拥塞、丢帧、关键帧请求、队列和音视频同步。

该方案复杂度和维护成本显著高于原生 RTP，不建议作为首期实现。

---

## 17. 代码改动清单

### 17.1 Web

| 文件 | 改动 |
|------|------|
| `device/src/webrtc.ts` | 通用 codec capability、SDP parser、H.265 Answer 校验 |
| `device/src/main.ts` | codec preference、selected codec、fallback 状态机、Stats codec |
| `device/src/protocol.ts` | START capability 字段 |
| `device/src/wss.ts` | `CodecRequest` 替代裸 codec 字符串 |
| `device/src/stats.ts` | codec/fallback 统计字段 |
| `device/src/ui.ts` | codec badge 与错误提示 |
| `device/index.html` | codec selector / capability 状态 |
| `test/test_h265_webrtc.mjs` | Puppeteer capability 与协商回归 |

### 17.2 CAE

| 文件 | 改动 |
|------|------|
| `CaeConfigManage.*` | codec policy、fallback、H.265 profile/tier/level、真实 capability 口径 |
| `CaeParamStorage.*` | per-client codec request/capability/result |
| `CaeControlCmdHandleThread.cpp` | 不再直接改全局 codec |
| `CaeConnectionAgent.*` | CodecCoordinator、按 selected codec 执行能力门禁、共享 codec lock、分阶段 fallback 和冲突处理 |
| `CaeMediaConfig.*` | immutable active codec snapshot、响应 JSON |
| `CaeEngineControl.*` | encoder generation、codec-scoped config cache、fallback |
| `media_engine.*` | capability bridge、normalized AU |
| `ScreenCapture.java` | MediaCodec probe、Main Tier、渐进配置、实际 output format |
| `WebRtcServerTransport.*` | H.265 fmtp、Answer 校验、移除全局 `s_isH265Codec` |
| `CaeMsgCode.*` | H.265 消息映射和 codec 错误码 |
| `CaeConfig.ini` | 统一默认和新增 H.265 配置 |
| `CaeSettingsActivity.java` / `EngineConfigPatcher.java` | 默认 H.264、非法值安全归一化、旧配置迁移和优先级 |

### 17.3 文档

实施后同步：

- `nexartc-install-deploy-guide.md`：codec 配置、浏览器 capability、联调 URL；
- `nexartc-logging-design.md`：codec 日志和排障矩阵；
- `nexartc-turn-mode-a-implementation.md`：回归项增加 H.265；
- `nexartc-cloud-phone-access-engine/docs/bitrate_control_analysis.md`：修正浏览器版本和完成度；
- APK assets、测试 ini、Settings UI 的默认值和注释。

---

## 18. 风险与缓解

| 风险 | 影响 | 缓解 |
|------|------|------|
| 浏览器 capability 随硬件/OS 变化 | 同版本行为不同 | 运行时探测，禁止 UA 白名单 |
| MediaCodec capability 表不准确 | 探测通过但 configure 失败 | configure 为最终判据，渐进降级 |
| H.265 专利许可 | 商业风险 | 上线前法务评估，保留 H.264 默认/回退 |
| High Tier/level 不兼容 | SDP 成功但解码失败 | Main Profile + Main Tier + 实际 level |
| 参数集缺失或跨 codec 污染 | 持续黑屏/PLI storm | encoder-scoped cache + generation + golden test |
| 多客户端 codec 冲突 | 重启共享编码器导致全体卡断 | 首客户端 codec 锁，拒绝不兼容 newcomer |
| H.265 硬编耗时或发热 | 帧率下降、系统降频 | capability 分档、温度/encode time 灰度指标 |
| H.265 大 IRAP 丢包 | 首帧慢、PLI | 参数集内带、NACK cache、码率起步和 PLI 节流 |
| 自动回退循环 | 重连风暴 | 每次连接最多一次，记录 fallback state |
| libdatachannel H.265 回归 | RTP 不可解码 | RFC 7798 golden test，必要时固定补丁版本 |

---

## 19. 最终验收 Checklist

### Web

- [ ] capability 无 H.265 时不请求 H.265
- [ ] capability 有 H.265 时 Auto 可请求 H.265
- [ ] START_SUCCESS 显示实际 selected codec
- [ ] Offer / Answer codec 与 selected codec 一致
- [ ] H.265 失败最多回退一次 H.264
- [ ] 收到 active H.265 codec conflict 时展示指定中文提示，并停止 fallback/自动重连
- [ ] Stats 输出实际 inbound codec
- [ ] H.265 会话不显示 H.264 profile 误导信息

### CAE

- [ ] H.265 capability 来源于真实 MediaCodec 枚举
- [ ] 设备无 HEVC capability 时 H.264 START 正常成功
- [ ] configure 失败可安全回退 H.264
- [ ] active codec 在共享流期间不可被第二客户端改写
- [ ] codec config 按 encoder generation 隔离
- [ ] NAL、Send、SDP 和 packetizer 均使用同一 immutable codec snapshot，不依赖进程静态 codec 标志
- [ ] H.265 使用 Main Profile / Main Tier
- [ ] SDP fmtp 与实际 encoder descriptor 一致
- [ ] Answer 未接受 H.265 时不发送 H.265 RTP
- [ ] Annex-B / length-prefixed 输入均正确标准化
- [ ] VPS/SPS/PPS 在首个 IRAP 前完整发送
- [ ] H.265 FU packetization 测试通过
- [ ] `CAE_H265_NOT_SUPPORTED` 可通过 `CaeMsgCode::GetMsg()` 返回稳定文案

### 集成

- [ ] H.264 全量回归通过
- [ ] Chrome 支持环境 H.265 出画
- [ ] Chrome Linux 无 H.265 环境自动 H.264
- [ ] Safari 支持环境 H.265 出画
- [ ] Firefox 自动 H.264
- [ ] host / hybrid / relay 均通过
- [ ] 双客户端同 codec 通过
- [ ] active H.265 + H.264-only newcomer 被明确拒绝且已有端不断
- [ ] H.264-only newcomer 收到 `0x1105`，页面提示当前 H.265 占用且不泄露其他用户信息
- [ ] 无 active session 的 H.265 晚失败可完成一次 H.264 重连
- [ ] assets、Settings 和运行时配置生产默认均为 H.264
- [ ] 断一端后其余端通过 IRAP 恢复
- [ ] 30 分钟稳定性无 crash、UAF、纯虚调用和 PLI storm

---

## 20. 参考资料

- RFC 7742 — WebRTC Video Processing and Codec Requirements: `https://www.rfc-editor.org/rfc/rfc7742`
- RFC 7798 — RTP Payload Format for HEVC: `https://www.rfc-editor.org/rfc/rfc7798`
- Chrome Status — H265 codec support in WebRTC（Feature 5153479456456704）: `https://chromestatus.com/feature/5153479456456704`
- MDN — `RTCRtpReceiver.getCapabilities()`: `https://developer.mozilla.org/docs/Web/API/RTCRtpReceiver/getCapabilities_static`
- MDN — Codecs used by WebRTC: `https://developer.mozilla.org/docs/Web/Media/Guides/Formats/WebRTC_codecs`

---

## 21. 小结

H.265 对本项目的价值主要是：在云手机高动态画面下，以更低带宽获得接近或更好的主观质量，并降低 TURN relay 成本。但它不能替代 H.264 的兼容基线。

当前仓库已经完成编码、NAL 识别、SDP 和 RTP 的基础接线，真正缺少的是“能力协商、共享状态正确性、可靠回退和测试闭环”。按本文 Phase 0 → Phase 1 实施后，H.265 才能从调试开关升级为可灰度、可观测、可回滚的生产能力。

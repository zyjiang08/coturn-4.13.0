# WebRTC ICE 模式（host / p2p / hybrid / relay）

本文说明云手机 WebRTC 通路上的 ICE 策略：默认 **host**（与现网 Mode A / WAN DNAT 行为一致），以及如何在 Web 与 CAE 两端对齐切换到 **p2p** / **hybrid** / **relay**。

## 1. 模式一览

| 模式 | 浏览器 | CAE | 适用场景 |
|------|--------|-----|----------|
| **host**（默认） | 不上 TURN；可保留 STUN（便于 srflx / 诊断） | 仅发送 `typ host`；Offer 标注 `a=ice-lite`；PeerConnection **不挂** STUN/TURN | 同网直连，或公网 **UDP DNAT** 到 `webrtc_port_range_*`（常见 50000） |
| **p2p** | 仅 STUN（无 TURN） | 挂 STUN；丢弃 `typ relay`；可发 host/srflx | 验证对称 NAT / 是否可直连，**无 TURN 兜底** |
| **hybrid** | STUN + TURN | STUN + TURN；优先 host/srflx，失败走 relay | 跨网通用；直连优先、TURN 兜底 |
| **relay** | `iceTransportPolicy=relay` | 仅 relay；`TransportPolicy::Relay` | 强制经 coturn，排障或严格中转 |

**原则**：两端策略必须一致。仅改浏览器 URL、不改 CAE（或相反）会导致一端发 relay、另一端丢弃，或一端只认 host、另一端无映射端口而失败。

## 2. 默认与「host 保持不变」

- **默认值**：Web 与 `CaeConfig.ini` 均为 `host`。
- **host + 固定端口 DNAT 可用**（STUN mapped port == `webrtc_port_range_*`）：与 Mode A 相同——注入公网 host、`a=ice-lite`、不挂 TURN。
- **host + DNAT 不可用**（蜂窝/CGNAT，mapped port ≠ 配置端口）：若已配置 TURN，CAE **自动将会话升级为 hybrid**（见 §5），否则跨网会失败。

## 3. 配置入口（优先级）

### 3.1 Web（`nexartc-cloudPhoneAccess-web/device`）

优先级（高 → 低）：

1. URL：`?ice_mode=host|p2p|hybrid|relay`（别名：`force_relay=1` → relay；`no_relay=1` / `p2p_only=1` → p2p）
2. 连接页「设置 → ICE 模式」下拉框（写入 `localStorage`：`cloudphone.device.iceMode`）
3. 默认 **host**

连接时 WebRTC 信令 **type=1** 请求体会带上：

```json
{ "type": 1, "ice_mode": "host", "...": "..." }
```

浏览器本地的 `RTCPeerConnection` iceServers / candidate 过滤与有效 ICE 模式一致（含 CAE Offer 升级后的会话覆盖）。

### 3.2 CAE

- 全局默认：`[webrtc] webrtc_ice_mode=host`（`CaeConfig.ini`）。
- **会话覆盖**：解析请求 JSON 的 `ice_mode`；合法值覆盖本会话；未传或非法则回退配置，再默认 host。
- TURN 主机/secret 仍来自配置；仅当本会话模式为 **hybrid/relay** 时才挂到 PeerConnection。
- Offer（type=2）携带 `ice_mode`（及可选 `ice_mode_reason`），供浏览器对齐。

日志关键字：

```text
WebRTC: session ice_mode=... client=... config=...
falling back to hybrid
host_dnat_unavailable
TURN REST credentials
injecting STUN-mapped public candidate
```

## 4. 端到端对齐建议

| 目标 | Web | CAE 配置建议 |
|------|-----|----------------|
| 现网 DNAT / 同网 | 默认或选 host | `webrtc_ice_mode=host` |
| 跨网且要直连优先 | 选 hybrid 或 `?ice_mode=hybrid` | 配置 TURN；或依赖 host 自动升级 |
| 强制中转 | `relay` | 同左；或 CAE `webrtc_ice_mode=relay` |
| 只测 P2P | `p2p` | 两端均为 p2p |

## 5. host 跨网失败与自动兜底

典型失败（本机蜂窝 `10.24.138.239` + 浏览器另一私网）：

```text
STUN mapped port mismatch local=50000 public=31379
ice_mode=host → 只发 10.24.138.239:50000 typ host
浏览器 iceState=checking，无 RTP
```

**自动兜底（Web `20260725B+` / 对应 CAE）**：

1. CAE 检测固定端口 DNAT 不可用且 TURN 已配置 → 本会话 `host` → `hybrid`。
2. Offer 带 `ice_mode=hybrid`、`ice_mode_reason=host_dnat_unavailable`。
3. Web 拉取 TURN，按 hybrid 建 PC；同时 CAE 可注入 STUN mapped `IP:port` 作为补充 host。

手动检查：

1. 路由器 DNAT：**UDP** 映射端口 = `webrtc_port_range_*`。
2. 硬刷新确认 `BUILD_VERSION=20260725B`，日志出现 escalate / TURN。
3. 无 TURN 配置时只能修 DNAT，或改选不可达。

## 6. 相关文档

- 部署与联调：[`nexartc-install-deploy-guide.md`](./nexartc-install-deploy-guide.md) §5.4.2
- Mode A / TURN：[`nexartc-turn-mode-a-implementation.md`](./nexartc-turn-mode-a-implementation.md) §2.2.1
- ICE UDP Mux 多客户端：[`nexartc-webrtc-ice-udp-mux-logical-separation.md`](./nexartc-webrtc-ice-udp-mux-logical-separation.md)

# nexartc 家宽 Edge TURN（本机 coturn）模式设计

> 日期：2026-07-26  
> 状态：**实现中（未标已支持）** — Hub/device/CAE/Edge 骨架已合入默认 `TURN_SITE=vps`；§10 端到端验收（强制 relay + Home READY）完成前，部署手册仍不得写成「已支持」  
> 产品上下文：NexaDesk / Mode A Signal Hub；VPS 带宽有限（现网约 5 Mbps）  
> 关联：  
> - [`nexartc-turn-mode-a-implementation.md`](./nexartc-turn-mode-a-implementation.md)（Mode A 拓扑）  
> - [`nexartc-webrtc-ice-modes.md`](./nexartc-webrtc-ice-modes.md)（ICE：host / p2p / hybrid / relay）  
> - [`nexartc-install-deploy-guide.md`](./nexartc-install-deploy-guide.md)  
> - [`turn-rest-api-signaling.md`](./turn-rest-api-signaling.md)（REST 凭据；§12.6.2 WAN 变更需重启 coturn）  
> 代码落点：`nexartc-cloudPhoneAccess-web/server/`（config / turn-site-config / turn-snapshot / home-turn-edge / session-router）、`device/src/{main,turn,wss}.ts`、`test/turn/home-edge/`、CAE `CaeHubTurnBind` + `WebRtcServerTransport` preflight/bind

---

## 1. 背景与问题

### 1.1 现网拓扑（保持不变的部分）

```
公网 VPS（固定 IP / 域名，带宽有限，如 5 Mbps）
├─ Signal Hub :443     信令 / CMS / device 静态页
└─ coturn（现网）      可选：媒体 TURN 中继（吃满上行时成为瓶颈）

家庭 / 机房
├─ 云手机 CAE          出站注册 Hub；媒体 host / hybrid / relay
└─ 用户浏览器          任意网络打开 Hub 上的 /device/
```

ICE **请求模式**已稳定（见 ICE 文档）：

| 请求 ice_mode | 含义 | 本设计态度 |
|---------------|------|------------|
| **host**（默认） | 优先仅 host / DNAT | **请求语义保持不变**；若最终有效模式 escalate 为 hybrid，则受 `turn_site` 约束（§2.1） |
| **p2p** | 仅 STUN，无 TURN | **保持不变** |
| **hybrid** | host/srflx 优先，TURN 兜底 | ICE 语义不变；TURN **落点**可配置 |
| **relay** | 强制仅 TURN | ICE 语义不变；TURN **落点**可配置 |

### 1.2 成本矛盾

- VPS **5 Mbps** 不足以长期承载双端 TURN 视频中继。  
- 信令锚点仍在 VPS；媒体中继希望尽量走 **家宽本机**。  
- 本机有真公网出口但 **动态 IP**；**不强制 DDNS**。  
- 仅切换浏览器 `iceServers`、CAE 仍走 VPS，**不能**宣称「媒体不经 VPS」。

### 1.3 目标

1. 新增可配置 TURN 落点：`turn_site=home`（家宽 Edge coturn）。  
2. 默认 `turn_site=vps`：**与今日行为兼容**。  
3. 无 DDNS：Edge 出站心跳 → Hub 登记 WAN IP → 签发 `turn:<ip>:…`。  
4. **浏览器与 CAE 同一会话快照**（site / secret / endpoint / generation）。  
5. Home 与 VPS **密钥隔离**，删除 URL 不足以防绕过。  
6. Edge 仅在 **READY** 后可签发；fail-closed。

### 1.4 非目标

- 不把 Hub/CMS 迁回家庭。  
- 不要求家宽 IP 永久固定。  
- P0 不强制 TURNS（纯 IP 证书麻烦）。  
- 不替代 CAE `50000` host DNAT。  
- 本文仍是设计稿，完整代码见 §10 闭环后标注。

---

## 2. 概念澄清：ICE 模式 vs TURN 落点

采用 **两轴正交**（不新增第五种 `ice_mode`）：

```
ice_mode（请求）  = host | p2p | hybrid | relay
effective_mode    = 会话最终 ICE 策略（可能由 host escalate 为 hybrid）
turn_site         = vps | home          ← 默认 vps
```

| effective_mode \ turn_site | `vps`（默认） | `home`（设计中） |
|----------------------------|---------------|------------------|
| host / p2p（有效） | 不使用 TURN | 不使用 TURN |
| hybrid | VPS coturn + `TURN_SECRET` | Home READY + `HOME_TURN_SECRET` |
| relay | 强制 VPS | 强制 Home READY |

### 2.1 请求模式 vs 有效模式（R6）

- **有效 host / 有效 p2p**：不读取 `turn_site`；可不取 TURN 票。  
- CAE 在固定端口 DNAT 不可用且已配置 TURN 时，可将请求 `host` **升级为有效 hybrid**。此时 **必须**读取 `turn_site`，并受 Edge READY / fallback 约束。  
- **禁止**在浏览器尚无票、CAE 已决定 escalate 的情况下直接建 PC。升级路径必须走 §5.3.4 **preflight → 取票 → bind → ACK** 协议。  
- 「Edge 离线时 host 仍可用」**仅**适用于有效 host（DNAT/同网本身可用）的场景；若已 escalate 为 hybrid 且 `TURN_SITE=home`、fallback=0，则与 hybrid/relay 一样 fail-closed（503 / 拒建连）。

### 2.2 产品命名

对外可称「家宽中继 / Edge TURN」；配置键：`TURN_SITE` / `turn_site`。

---

## 3. 总体架构

### 3.1 `turn_site=home` 拓扑

```
                    ┌─────────────────────────────────────┐
                    │ 公网 VPS（低带宽）                     │
                    │  Hub：信令 / CMS / 会话快照 / 签发     │
                    │  VPS coturn：仅 turn_site=vps 使用    │
                    └──────────────▲────────────────────────┘
                                   │ 低流量 WSS/HTTPS
        ┌──────────────────────────┼──────────────────────────┐
        │                          │                          │
 ┌──────┴──────┐            ┌──────┴──────┐           ┌──────┴──────┐
 │ Edge Agent  │            │ 云手机 CAE  │           │ 用户浏览器  │
 │ + coturn    │            │ LAN→TURN    │           │ WAN→TURN    │
 │ 本机        │            │             │           │             │
 └──────┬──────┘            └──────┬──────┘           └──────┬──────┘
        │                          │  turn:<LAN>             │ turn:<WAN>
        │                          │  (同站控制面)            │
        └──────────────────────────┴─────────────────────────┘
              relay candidate 对外仍广告 WAN；媒体不经 VPS
```

### 3.2 角色

| 角色 | 职责 |
|------|------|
| Hub | 鉴权 Edge；READY 状态机；按 site 选 **host+secret**；会话 `turnSiteSnapshot`；签发 |
| Edge coturn | 本机 TURN；`HOME_TURN_SECRET`；`external-ip=WAN/LAN` |
| Edge Agent | 心跳；原子写配置；**restart** coturn；上报 applied；配合公网探测 |
| CAE | 建 PC **前**应用快照；同站优先 `turn:<LAN_IP>` |
| 浏览器 | **先 join 再取票**；type=1 回传 snapshotToken；解析 503/`SNAPSHOT_STALE`；缓存失效 |

### 3.3 成本

| 路径 | VPS | 本机 |
|------|-----|------|
| 有效 host | ≈0 | ≈0 TURN |
| hybrid/relay + vps | 高 | 低 |
| hybrid/relay + home READY | ≈0 | 高（家宽） |

---

## 4. 动态公网 IP 与 Edge READY 状态机

### 4.1 原则

- 无 DDNS；签发使用 **合法公网 IPv4 字面量**。  
- **心跳在线 ≠ TURN Ready**（R3）。  
- 权威 WAN 见 §4.1.1；Agent `selfWanIp` 仅作交叉检查（不一致 → `[EXC]`，不进入 READY）。

### 4.1.1 权威 WAN IP 解析与校验（P0 必选）

Hub 在 Edge 心跳连接上解析「客户端公网 IP」时必须遵守：

1. **直连（无反代）**  
   - 使用 TLS/TCP 套接字的 peer 地址。  
   - 若为 IPv4-mapped IPv6（`::ffff:x.x.x.x`），**归一化为** `x.x.x.x`。

2. **经可信反代（Caddy / Nginx 等）**  
   - 仅当连接来自 **显式白名单**（`EDGE_TRUSTED_PROXIES`，CIDR 列表）时，才解析转发头。  
   - **部署要求**：反代必须 **覆盖/重写** 客户端传入的 `X-Forwarded-For`（追加真实 peer，或丢弃客户端伪造前缀后只写可信链），不得原样透传。  
   - **Hub 取值算法（防伪造）**：从右向左遍历 `X-Forwarded-For` 列表，跳过仍属于 `EDGE_TRUSTED_PROXIES` 的地址，取 **第一个非可信地址** 作为 client；若整链皆为可信代理且无 client → 失败。  
     （等价于：套接字 peer 必须是可信代理；再剥掉右侧可信跳，留下最靠近客户端的非代理 IP。）  
   - 若仅有 `X-Real-IP`：仅当 peer ∈ 白名单且该头为单一合法 IPv4 时采用；否则忽略。  
   - 非白名单来源：**忽略** 全部转发头，仍用套接字 peer；并打审计日志。  
   - 未配置白名单时，**禁止**信任任意 `X-Forwarded-For`。

3. **地址拒绝规则（归一化后）** — 命中任一条则心跳失败 / 不进入 SEEN→READY：  
   - 非 IPv4（纯 IPv6 **P0 不支持** Edge WAN）；  
   - 私网：`10.0.0.0/8`、`172.16.0.0/12`、`192.168.0.0/16`、`127.0.0.0/8`；  
   - 链路本地：`169.254.0.0/16`；  
   - CGNAT：`100.64.0.0/10`；  
   - 其他保留/文档/组播等（实现可用标准「公网单播」判定库）。  

4. **与 `selfWanIp`（统一：不一致不得 READY）**  
   - Agent 必须上报 `selfWanIp`（其出口探测值）。  
   - 与 Hub 权威 WAN **不一致** → 记 `[EXC]`，**本轮心跳失败，不得进入 / 保持 READY**（不以「仅告警、仍用 Hub 值」放行）。  
   - 权威值命中上款拒绝规则 → 同样失败。  
   - 一致且通过拒绝规则 → ACK 回写该 `wanIp`，方可继续 SEEN→READY。

### 4.2 状态机（P0 必选）

```text
OFFLINE
  → SEEN              # Hub 见到心跳 / 新 wanIp
  → APPLYING          # 撤销旧 generation；原子写 conf；restart coturn
  → LOCAL_READY       # Agent 本地自检（listening + 可选 loopback Allocate）
  → EXTERNAL_PROBING  # Hub（或指定探测点）对 WAN:3478 做 UDP/TCP Allocate
  → READY(generation) # 仅此状态可进入 turn-credentials
```

规则：

1. **只有 `READY` 的 generation** 可签发 Home 凭据。  
2. WAN IP 变化：先撤销旧 generation → `APPLYING`；**禁止**在 APPLYING/LOCAL_READY 窗口签发「新 IP + 旧 coturn」。  
3. coturn **不支持**普通 `systemctl reload` 热更新 `external-ip`；必须 **restart**（见 `turn-rest-api-signaling.md` §12.6.2）。  
4. restart / 写配置 / 公网探测失败 → fail-closed，保持 OFFLINE 或回到可探测前状态，返回 `503 HOME_TURN_OFFLINE`。  
5. Agent 上报 `appliedWanIp`（已写入并 restart 后的值）；须与 Hub 权威 `wanIp` 一致才进入 EXTERNAL_PROBING。  
6. 生产验收：探测应包含实际 relay 数据路径，不只 Allocate 成功。

### 4.3 凭据 TTL vs 端点新鲜度（R8）

| 概念 | 机制 |
|------|------|
| 凭据有效期 | `TURN_TTL` / `HOME_TURN_TTL`（REST username expiry） |
| 端点新鲜度 | `generation` + 短时 endpoint 缓存（**远短于** TTL，建议 ≤60s） |

- 缩短 TTL **不会**自动更新已建立 PeerConnection 内的旧 TURN URL。  
- 每个**新会话**必须确认当前 generation；或禁用/极短缓存 `iceServers`。  
- IP / site / generation 变化或 ICE 失败：清缓存；重建 PC 或受控 ICE restart。  
- 验收：「会话持续超过 TTL」与「TTL 后断网重连 / 重新 Allocate」。

---

## 5. 配置设计（向前兼容）

### 5.1 Hub（`/etc/nexartc/hub.env`）

| 变量 | 默认 | 说明 |
|------|------|------|
| `TURN_SITE` | `vps` | `vps` \| `home`；未设置 = vps |
| `TURN_SECRET` | 现网 | **仅 VPS coturn** |
| `HOME_TURN_SECRET` | （home 必填） | **仅 Home coturn**；P0 强制与 `TURN_SECRET` 不同（R1） |
| `TURN_HOST` / `TURN_HOST_IP` | 现网 | 仅 `turn_site=vps` |
| `TURN_PORT` | `3478` | 端口号约定（VPS/Home 控制口约定可相同） |
| `ENABLE_TURNS` | 现网 | **仅 VPS 站点**是否签发 `turns:` |
| `HOME_ENABLE_TURNS` | `0` | **仅 Home 站点**是否签发 `turns:`（P0 默认 0；与 `ENABLE_TURNS` 独立） |
| `TURNS_PORT` | `5349` | VPS TURNS 端口 |
| `HOME_TURNS_PORT` | `5349` | Home TURNS 端口（仅 `HOME_ENABLE_TURNS=1`） |
| `TURN_TTL` | 现网 | VPS 凭据 TTL |
| `HOME_TURN_TTL` | `600` | Home 凭据 TTL（与 endpoint 缓存策略解耦） |
| `EDGE_TOKEN` | （home 必填） | **独立、非空**；禁止回退 `AGENT_TOKEN`（R5） |
| `EDGE_ID` | 如 `home-turn-1` | token 绑定的固定 edgeId |
| `EDGE_TRUSTED_PROXIES` | 空 | 可信反代 CIDR 列表；空=不信任 XFF（§4.1.1） |
| `HOME_TURN_STALE_SEC` | `90` | 心跳超时 → 撤 READY |
| `HOME_TURN_ALLOW_VPS_FALLBACK` | `0` | 成本优先默认 0；fallback 时 **整站切换**（含 VPS 的 `ENABLE_TURNS`） |

站点配置对象（逻辑视图，签发时按最终 `site` 选取一整份，禁止混用字段）：

```ts
type TurnSiteConfig =
  | {
      site: 'vps';
      secret: typeof TURN_SECRET;
      hosts: string[];           // TURN_HOST / TURN_HOST_IP
      turnPort: number;
      turnsEnabled: boolean;     // ENABLE_TURNS
      turnsPort: number;         // TURNS_PORT
      ttl: number;               // TURN_TTL
    }
  | {
      site: 'home';
      secret: typeof HOME_TURN_SECRET;
      wanHost: string;           // READY edge.wanIp
      lanHost: string;           // edge.lanIp（CAE）
      turnPort: number;
      turnsEnabled: boolean;     // HOME_ENABLE_TURNS
      turnsPort: number;         // HOME_TURNS_PORT
      ttl: number;               // HOME_TURN_TTL
      edgeId: string;
      generation: number;
    };
```

### 5.2 Edge Agent / coturn（安全模板要点，R10）

```bash
# /etc/nexartc/home-turn.env（不进 Git）
HUB_URL=https://www.signalling-nexartc.cn
EDGE_TOKEN=<独立密钥>
EDGE_ID=home-turn-1
HOME_TURN_SECRET=<与 Hub HOME_TURN_SECRET 相同，≠ TURN_SECRET>
LAN_IP=192.168.124.102
RELAY_MIN_PORT=49152
RELAY_MAX_PORT=49251
HOME_ENABLE_TURNS=0
```

coturn 要求（模板 + systemd）：

- `static-auth-secret` **仅**经 EnvironmentFile / 启动参数注入，不进 Git。  
- `listening-ip` / `relay-ip=<LAN_IP>`；`external-ip=<WAN>/<LAN>` 由 Agent 维护。  
- `fingerprint`、`use-auth-secret`、`realm`、`no-cli`、`no-multicast-peers`。  
- `user-quota`、`total-quota`、每会话 `max-bps`、全机 `bps-capacity`（单位 bytes/s）。  
- **Peer ACL / 网络隔离**：禁止普通 TURN 用户把 Edge 当跳板访问路由器/IoT；若需触达 CAE 私网，最小 allowlist 或独立 VLAN/DMZ。  
- relay 端口池按浏览器+CAE 的 UDP/TCP allocation 数量估算，不能只按「100 口≈并发」拍脑袋。

### 5.3 会话快照与原子取票（P0 必选）

#### 5.3.1 快照结构

```ts
interface TurnSiteSnapshot {
  snapshotId: string;            // 会话级唯一；绑定 Hub sessionId
  sessionId: string;             // Hub /ws join 后的 sessionId
  site: 'vps' | 'home';
  edgeId?: string;
  generation: number;            // home READY 代数；vps 可用 0
  endpoints: {
    browserTurnHost: string;     // 通常 WAN IP
    caeTurnHost: string;         // 同站优先 LAN IP
    turnPort: number;
    turnsPort?: number;
    turnsEnabled: boolean;       // 来自该站点的 ENABLE_TURNS / HOME_ENABLE_TURNS
  };
  /** home=强制 hub_rest；vps=CAE 仍可 cae_local_hmac（现网方案 B），见 §5.3.3 */
  credentialSource: 'hub_rest' | 'cae_local_hmac';
  snapshotToken: string;         // 短时 HMAC/随机令牌，浏览器 type=1 回传
}
```

#### 5.3.2 与浏览器取票的原子顺序（禁止「先并行拉票再 join」）

现网 `device/src/main.ts` 在 Hub 会话建立前并行 `fetchTurnIceServers()`，`generation` 可能在取票与 CAE 建 PC 之间变化。P0 **必须**改为：

```text
1) 浏览器 WSS join Hub → 获得 sessionId
2) Hub 冻结 TurnSiteSnapshot(snapshotId, sessionId, generation, …)
   （此时记录 edge.generation；若随后 Edge 升代数，本快照仍绑定旧 generation，
    仅当 Edge 撤销该 generation 时快照作废）
3) 若请求 ice_mode ∈ {hybrid, relay}：
   浏览器凭 sessionId 取票 → type=1 bind(snapshotToken) → Hub 编排 CAE → ACK → 建 PC
   （细节同下「bind 路径」）
4) 若请求 ice_mode ∈ {host, p2p}：
   先走 §5.3.4 preflight；仅当 EFFECTIVE_MODE 仍为 host/p2p 时可跳过取票直接建 PC
```

**bind 路径**（hybrid / relay，或 host escalate 之后）：

```text
A) GET/POST /api/v1/turn-credentials?sessionId=…
   → iceServers + snapshotId/snapshotToken/generation（绑定已冻结 snapshot）
B) type=1 bind：{ phase: 'bind', ice_mode|effective_mode, snapshotId, snapshotToken, … }
C) Hub 校验 token 后向 CAE 下发同一 snapshot
   - home：含 CAE username/credential + caeTurnHost
   - vps：可只带端点元数据；CAE 继续方案 B
D) CAE ACK → 双方才可创建 PeerConnection
```

规则：

- **禁止**无 `sessionId` 的匿名拉票用于观看会话建 PC。  
- 浏览器持有的 `iceServers` 必须带 `snapshotId`；与 bind 不一致 → Hub 拒绝 / 要求重拉。  
- Edge `generation` 在冻结后被撤销 → `SNAPSHOT_STALE`，重走冻结 + 取票 + bind。  
- `turn_site=vps` 同样走 join→（按需取票）→bind 统一路径；`generation=0`。

#### 5.3.3 CAE 凭据来源（二选一已决：P0 强制 Hub 签发）

| turn_site | 浏览器 | CAE | CAE 是否持有 master secret |
|-----------|--------|-----|----------------------------|
| `vps`（现网） | 方案 A（Hub REST） | **保持方案 B**（本地 HMAC + `TURN_SECRET`） | 是（现网不变） |
| `home`（P0） | Hub REST（绑定 snapshot） | **强制 Hub 下发会话凭据**（`credentialSource: 'hub_rest'`） | **否**（不配置 `HOME_TURN_SECRET` / 不读 home master） |

说明：

- **删除**「Home 模式过渡期 CAE 本地 HMAC」描述；P0 **不**支持 `credentialSource: 'cae_local_hmac'`（home）。  
- Mode A 总册中「CAE 方案 B」仅适用于 **`turn_site=vps`**；home 路径以本文为准。  
- Hub → CAE 控制消息携带：`snapshotId`、`generation`、`caeTurnHost`、`username`、`credential`、`ttl`；CAE 建 PC 前应用并 ACK。  
- 线程安全：覆盖入口在 `CaeConnectionAgent` / `WebRtcServerTransport` 明确；禁止仅进程启动读 ini。

#### 5.3.4 host → hybrid 升级：preflight / 暂停协议（P0 必选）

问题：请求 `ice_mode=host` 时浏览器可先不取票；但 **是否 escalate 由 CAE 在收到 type=1 后决定**。若 CAE 已决定 hybrid，而浏览器无 `snapshotToken`/TURN 票，双方又禁止「无票先建 PC」，则时序死锁。

P0 协议（**type 编号仍为 1**；用载荷字段区分阶段，不新增信令 type）：

```text
1) join → Hub 冻结 snapshot（此时尚可不签发冰票；snapshot 已占 generation）
2) 浏览器发送 type=1 preflight
   { phase: 'preflight', ice_mode: 'host'|'p2p'|…, sessionId, snapshotId? }
   （无 snapshotToken / 无 iceServers 要求）
3) Hub 转交 CAE；CAE **只做策略判定，不创建 PeerConnection**，返回控制应答
   （可经 type=1 响应扩展字段，或 Hub 控制面消息）
   { effective_mode: 'host'|'p2p'|'hybrid', reason? }
4a) EFFECTIVE_MODE ∈ {host, p2p}：
    浏览器跳过取票；发送 type=1 bind（可无 token）；双方按有效 host/p2p 建 PC
4b) EFFECTIVE_MODE = hybrid：
    浏览器暂停建连 → 按既有 sessionId/snapshot 取票
    → type=1 bind({ phase:'bind', effective_mode:'hybrid', snapshotId, snapshotToken, … })
    → Hub 校验 token，向 CAE 下发会话凭据（home）或端点元数据（vps）
    → 等待 CAE ACK → 双方建 PC
5) 任一步失败（HOME_TURN_OFFLINE / SNAPSHOT_STALE / ACK 超时）：
    浏览器与 CAE 均不得留下半开 PC；可提示后重走 join 或仅重走 2–4
```

规则：

- preflight **禁止**创建 `RTCPeerConnection` / 发 Offer。  
- escalate 后的取票必须使用 **同一** join 冻结的 `snapshotId`（若 generation 已撤销则 `SNAPSHOT_STALE`，重新 join/冻结）。  
- 请求已是 `hybrid`/`relay` 可跳过 preflight，直接取票 + bind（§5.3.2）。  
- Hub 在 `turn_site=home` 路径上 **不是纯透明转发**：须识别 `phase`、校验 bind token、编排 CAE 凭据与 ACK（见 Mode A 兼容策略修订）。

### 5.4 浏览器 / device（R7，P0 必选）

- `ice_mode` UI/URL 四态不变。  
- **改连接时序**：废除「connect 开头并行拉票」用于建 PC。  
  - `hybrid`/`relay`：join → 取票 → type=1 bind。  
  - `host`/`p2p`：join → type=1 preflight →（若 escalate）取票 → bind。  
- `device/src/turn.ts`：**必须**解析结构化错误（`HOME_TURN_OFFLINE`、`SNAPSHOT_STALE`）。  
  - **relay**：立即停建连并明确提示。  
  - **hybrid**（含 host escalate）：可降级 STUN/host，必须记录有效模式与原因。  
  - **fallback=1**：仅接受 Hub 重新冻结的 **整站 VPS** 快照与凭据。  
- 响应字段：`turnSite`、`edgeId`、`generation`、`snapshotId`、`snapshotToken`；缓存策略见 §4.3（按 `snapshotId` 失效）。

### 5.5 密钥隔离（R1，P0）

- Home 临时凭据 = `HMAC(HOME_TURN_SECRET, username)`。  
- 该凭据在 VPS coturn 上 **必须鉴权失败**（自动化验收）。  
- Fallback：同时切换 host **与** secret，禁止「Home secret + VPS host」或相反。  
- 同一响应 **只发布一个最终 site**，不得 Home+VPS 双 URL 赌 ICE 优选（R9）。

---

## 6. Hub 协议与签发

### 6.1 Edge 通道

- `WSS /turn-edge` 或 `POST /api/v1/turn-edge/heartbeat`。  
- Bearer **`EDGE_TOKEN`**（绑定 `EDGE_ID`）；常量时间比较；限流 + 审计。  
- 重复注册 / takeover：同 `edgeId` 仅允许持有正确 token 的连接替换；记录 `[STAB]`。

心跳（Agent → Hub）示例字段：`edgeId`、`lanIp`、`selfWanIp`、`appliedWanIp`、`state`、`generation`、`relayPortMin/Max`、`ts`。

ACK：`wanIp`（权威）、`turnSite`、`expectedGeneration`、`ts`。

### 6.2 内存状态

```ts
interface HomeTurnEdge {
  edgeId: string;
  wanIp: string;
  lanIp: string;
  state: 'OFFLINE' | 'SEEN' | 'APPLYING' | 'LOCAL_READY' | 'EXTERNAL_PROBING' | 'READY';
  generation: number;
  lastSeen: number;
  appliedWanIp?: string;
  relayPortMin: number;
  relayPortMax: number;
  turnsEnabled: boolean;
}
```

### 6.3 签发（R9 + 快照绑定）

必须同时改造，而非只改 host 列表：

1. **site-aware `isTurnConfigured()`**  
   - vps：现网 `TURN_SECRET` + `TURN_HOST`（或 IP）。  
   - home：`HOME_TURN_SECRET` + Edge **READY**；**不**依赖 VPS `TURN_HOST`。  
2. 按最终 site 选取完整 **`TurnSiteConfig`**（secret / TTL / hosts / turns 开关）。  
3. Home 分支：**禁止**混入请求 Host、`TURN_HOST_IP`、硬编码 VPS IP。  
4. 观看会话签发 **必须**带有效 `sessionId`，且对应已冻结 `snapshotId`；响应含  
   `turnSite`、`edgeId`、`generation`、`snapshotId`、`snapshotToken`。  
5. 非 READY 且 fallback=0 → `503` + `code=HOME_TURN_OFFLINE`。  
6. 快照 generation 已撤销 → `409`/`503` + `code=SNAPSHOT_STALE`。

伪代码：

```ts
function issueTurnCredentials(sessionId: string): Creds {
  const snap = getFrozenSnapshot(sessionId);
  if (!snap) throw BadRequest('SESSION_SNAPSHOT_REQUIRED');
  if (snap.site === 'home' && !isGenerationStillReady(snap.generation)) {
    throw ServiceUnavailable('SNAPSHOT_STALE');
  }
  const cfg = siteConfigFromSnapshot(snap); // 完整站点对象，不混字段
  return buildWith({
    secret: cfg.secret,
    hosts: [cfg.site === 'home' ? cfg.wanHost : …],
    turnsEnabled: cfg.turnsEnabled,
    meta: {
      turnSite: snap.site,
      edgeId: snap.edgeId,
      generation: snap.generation,
      snapshotId: snap.snapshotId,
      snapshotToken: snap.snapshotToken,
    },
  });
}
```

CAE 凭据（仅 `site=home`）：Hub 在校验 type=1 的 `snapshotId`/`snapshotToken` 后，用**同一** `HOME_TURN_SECRET` 生成 CAE 专用 username/credential（可与浏览器不同 userId 前缀，但同 secret/同 generation/同 snapshot）。`site=vps` 不走此分支，CAE 仍用方案 B。

### 6.4 Admin

`GET /api/v1/turn-edge`：返回 state、generation、wanIp、lastSeen（勿暗示未 READY 可签发）。

---

## 7. 同站地址与 NAT hairpin（R4）

| 端 | TURN 控制地址 |
|----|----------------|
| 外网浏览器 | `turn:<WAN_IP>:3478` |
| 同站 CAE | **优先** `turn:<LAN_IP>:3478` |
| coturn | `external-ip=<WAN_IP>/<LAN_IP>`（relay 对外仍为 WAN） |

P0 必须明确并验收下列拓扑之一：

1. coturn 直接拥有公网地址；或  
2. 路由器对 **控制口 + relay 段** 确认 NAT loopback；或  
3. 受控的非对称路径（需单独设计，默认不采用）。

硬性要求：

- relay 端口 **公网/内网同号 1:1**；「映射整段」**不含端口转换**。  
- 验收：关闭 NAT loopback 后，CAE 仍能通过 **LAN TURN** Allocate；强制 relay 的 selected pair 数据路径符合所选拓扑。  
- 强制 relay 时 selected candidate pair **仅指向 Home**；VPS 无本会话媒体。

---

## 8. 端口与网络

| 端口 | 协议 | 用途 |
|------|------|------|
| 3478 | UDP+TCP | 控制 / Allocate |
| `min-port`–`max-port` | UDP | relay（整段 1:1） |
| 5349 | TCP | P0 可关 |

4G 无法打入 3478 → 本模式不可用，保持 `TURN_SITE=vps` 或强化 host DNAT。

---

## 9. 安全

| 项 | 要求 |
|----|------|
| `EDGE_TOKEN` | 独立、非空、绑定 `EDGE_ID`；不复用 `AGENT_TOKEN` |
| `HOME_TURN_SECRET` / `TURN_SECRET` | 分离；跨站凭据必须失败 |
| Master secret | `turn_site=home` 时 CAE **不得**持有 `HOME_TURN_SECRET`；仅用 Hub 下发会话凭据 |
| Open relay | 禁止；必须 use-auth-secret |
| 内网跳板 | ACL / VLAN；见 §5.2 |
| 限流审计 | 心跳与签发 |
| 客户端 | 不可擅自把生产 `turn_site` 切到 VPS |

---

## 10. P0 闭环（不可拆分顺序）

> 下列七项全部完成前，不得将 `TURN_SITE=home` 标为「已支持」。

1. **身份与密钥隔离**：`EDGE_TOKEN`、`HOME_TURN_SECRET`；站点配置对象（含 `ENABLE_TURNS` / `HOME_ENABLE_TURNS`）。  
2. **Edge READY 状态机**：权威 WAN 校验（§4.1.1）→ 原子配置 → **restart** → 本地检查 → 公网探测 → 发布 generation。  
3. **Hub 会话快照 / 原子签发**：join 后冻结 snapshot；credentials 绑定 `sessionId`/`snapshotId`；site-aware secret+host；Home 不混 VPS。  
4. **CAE 对齐**：P0 home **仅** Hub 下发会话凭据；preflight 不建 PC；bind 后 ACK；同站 LAN TURN。  
5. **Web 对齐**：join →（host/p2p 先 preflight）→ 按需取票 → type=1 bind；503 / `SNAPSHOT_STALE`。  
6. **coturn 安全部署**：1:1 映射、配额、容量、ACL、systemd restart。  
7. **端到端验收**：selected pair 属 Home；VPS 无媒体；§14 清单。

### P1（P0 之后）

- Admin UI；可运营 fallback 开关；Home `HOME_ENABLE_TURNS`+证书；多 Edge；mTLS；IPv6 WAN（若需要）。

---

## 11. 代码触点清单

| 模块 | 文件（预期） | 改动 |
|------|----------------|------|
| Hub config | `server/src/config.ts` | 站点对象字段：`HOME_*`、`EDGE_TRUSTED_PROXIES`、双 TURNS 开关 |
| Hub TURN | `server/src/turn-credentials.ts` | site-aware；禁混 VPS；`sessionId` 绑定；snapshot 元数据 |
| Hub Edge | `server/src/home-turn-edge.ts`（新） | WAN 校验、状态机、generation、探测 |
| Hub 路由 | `server/src/main.ts` | heartbeat、credentials、status |
| Hub 会话 | `session-router` / 信令 | 冻结 snapshot；识别 type=1 `phase`；校验 bind token；编排 CAE 凭据/ACK |
| device | `main.ts`、`turn.ts` | preflight/bind；escalate 后取票；503/STALE |
| CAE | `CaeConnectionAgent` / `WebRtcServerTransport` | preflight 只回 `effective_mode`；bind 后应用凭据+ACK |
| 本机 | `test/turn/home-edge/*` | Agent、conf 模板、systemd、探测脚本 |

**明确少动：** 有效 host 的候选注入、`a=ice-lite`、UDP mux 50000、CMS 用户模型、ICE 四态枚举。

---

## 12. 配置示例

### 12.1 现网兼容（默认）

```bash
TURN_SITE=vps
TURN_SECRET=<vps-secret>
TURN_HOST=www.signalling-nexartc.cn
TURN_HOST_IP=120.79.21.28
ENABLE_TURNS=1
# HOME_ENABLE_TURNS 可忽略；无需 EDGE_TOKEN / HOME_TURN_SECRET
```

### 12.2 家宽 Edge（设计目标）

```bash
TURN_SITE=home
TURN_SECRET=<vps-secret>              # 仅 fallback=1 或同机仍跑 VPS coturn 时使用
HOME_TURN_SECRET=<不同的 home-secret>
EDGE_TOKEN=<独立>
EDGE_ID=home-turn-1
EDGE_TRUSTED_PROXIES=                 # 若 Hub 前有反代，填反代 CIDR；否则留空
TURN_PORT=3478
ENABLE_TURNS=1                        # VPS 站点行为保持现网；fallback 时仍可 turns
HOME_ENABLE_TURNS=0                   # Home P0 默认关 TURNS
HOME_TURN_TTL=600
HOME_TURN_STALE_SEC=90
HOME_TURN_ALLOW_VPS_FALLBACK=0
```

```text
ice_mode=host     # 日常；仅当保持有效 host 时与 turn_site 无关
ice_mode=relay    # 验收双边 Home TURN
ice_mode=hybrid   # 跨网；escalate 后受 turn_site 约束
```

---

## 13. 风险与回滚

| 风险 | 缓解 |
|------|------|
| 假 READY | 状态机 + 公网探测 |
| 密钥混用 | 分 secret + 跨站鉴权失败用例 |
| 单边 TURN | P0 CAE 快照 |
| Hairpin | LAN 控制地址 + 拓扑验收 |
| Token 伪造 | 独立 EDGE_TOKEN |
| 旧 iceServers 缓存 | generation + 短缓存 |
| 上行打满 | quota / max-bps |
| 回滚 | `TURN_SITE=vps`；无需回退 ice_mode 枚举 |

---

## 14. 验收清单

### 14.1 兼容与主路径

- [ ] 默认 `TURN_SITE=vps`：与改前一致。  
- [ ] 有效 host：不依赖 Home READY。  
- [ ] `TURN_SITE=home` + READY：credentials 仅 Home；含 `turnSite/edgeId/generation/snapshotId`。  
- [ ] 强制 relay：浏览器与 CAE 日志同一 `snapshotId/generation`；selected pair 仅 Home；VPS 无本会话媒体。  
- [ ] Hub 模式：未 join 不能用匿名票建 PC；type=1 bind 缺/错 `snapshotToken` 被拒。  
- [ ] 请求 host 且 CAE escalate→hybrid：preflight 不建 PC；取票后 bind；双方 ACK 后再建 PC。  

### 14.2 评审追加

- [ ] Home 凭据访问 VPS coturn **鉴权失败**。  
- [ ] Edge 仅 SEEN/LOCAL_READY 时 **不签发** Home。  
- [ ] restart 或公网探测失败 → `503 HOME_TURN_OFFLINE`。  
- [ ] WAN 变更无「新 IP + 旧 coturn」签发窗口。  
- [ ] CAE 经 LAN TURN Allocate；关 NAT loopback 后符合所选拓扑。  
- [ ] relay 收 503 立即提示；hybrid 降级显示有效模式。  
- [ ] 旧 Web endpoint 缓存 + generation 切换：新会话不用旧 WAN。  
- [ ] 取票后 Edge 升代数 / 撤销：返回 `SNAPSHOT_STALE`，重走冻结流程。  
- [ ] 会话超过 TTL；TTL 后重连 / 重新 Allocate 符合设计。  
- [ ] Hub / Edge / coturn 各自重启：fail-closed → READY。  
- [ ] 端口耗尽、带宽上限、allocation 配额：有指标与错误日志。  
- [ ] 伪造 XFF（左侧插入假公网 IP）不能改变权威 WAN；缺失/私网/CGNAT 不能 READY。  
- [ ] `selfWanIp` 与 Hub 权威不一致：不能 READY。  
- [ ] `ENABLE_TURNS=1` 且 `HOME_ENABLE_TURNS=0`：Home 票无 turns；fallback 到 VPS 后票含现网 turns。  
- [ ] Home 路径 CAE 进程环境 **无** `HOME_TURN_SECRET`，仅使用 Hub 下发凭据。  

---

## 15. 文档与索引

| 文档 | 关系 |
|------|------|
| 本文 | Edge TURN 主设计（设计稿） |
| [`nexartc-webrtc-ice-modes.md`](./nexartc-webrtc-ice-modes.md) | ICE；turn_site 为**设计中** |
| [`nexartc-turn-mode-a-implementation.md`](./nexartc-turn-mode-a-implementation.md) | Mode A；落点规划 |
| [`nexartc-install-deploy-guide.md`](./nexartc-install-deploy-guide.md) | 实现后补步骤；此前不得写成已支持 |
| [`turn-rest-api-signaling.md`](./turn-rest-api-signaling.md) | REST；WAN 变更 restart |

---

## 16. 决策摘要

1. 新能力是 **`turn_site=home`**，不是新的 `ice_mode`。  
2. 默认 vps；有效 host 不读 turn_site；host→hybrid 须 preflight→取票→bind。  
3. P0：**分 secret、READY、snapshot、home 双边 Hub 凭据、Web/CAE 编排** 一体交付。  
4. 无 DDNS；权威 WAN：直连或右向左剥可信代理；`selfWanIp` 不一致不得 READY；**restart**。  
5. 同站 CAE 用 LAN TURN；`ENABLE_TURNS` / `HOME_ENABLE_TURNS` 分站点。  
6. 成本优先：Home 不混 VPS URL；fallback 整站切换（含 VPS TURNS 开关）。  

---

## 17. 评审意见与吸收说明（2026-07-26）

> 评审结论：**两轴方向保留；原 P0 不可实施。** 正文 §1–§16 已按下列意见改写。若实现与本节冲突，以本节门槛与 §10 闭环为准。在阻断项关闭前，状态保持「设计稿（待实现）」。

### 17.1 阻断项 → 正文落点

| ID | 要求摘要 | 已吸收位置 |
|----|----------|------------|
| R1 | Home/VPS 分 secret；签发选 host+secret；跨站鉴权失败 | §5.1、§5.5、§6.3、§14.2 |
| R2 | CAE 对齐并入 P0；会话 `turnSiteSnapshot` | §5.3、§10.4、§11 |
| R2b | 快照与取票原子绑定；type=1 bind 回传 token | §5.3.2、§6.3、§10.3/5、§14 |
| R2c | Home P0 强制 `hub_rest`；废除 home 本地 HMAC | §5.3.3；Mode A 表注 |
| R2d | host escalate：preflight → 取票 → bind → ACK | §2.1、§5.3.4、§10.4/5、§14 |
| R3 | READY 状态机；restart 非 reload；公网探测 | §4.2、§10.2 |
| R3b | WAN 权威：右向左剥可信代理；`selfWanIp` 不一致不得 READY | §4.1、§4.1.1 |
| R4 | 浏览器 WAN / CAE LAN；hairpin；1:1 映射 | §3.1、§7、§8 |
| R5 | 独立 `EDGE_TOKEN`，不复用 `AGENT_TOKEN` | §5.1、§6.1、§9 |
| R6 | 请求模式 vs 有效模式；escalate 后读 turn_site | §1.1、§2.1、§17.2 |
| R7 | device 解析 503；relay/hybrid 行为 | §5.4、§10.5、§11 |
| R8 | TTL 与 generation/缓存解耦 | §4.3、§14.2 |
| R9 | 全面改 credentials，禁混 VPS | §6.3、§11 |
| R9b | `ENABLE_TURNS` / `HOME_ENABLE_TURNS` 分站点 | §5.1、§12、§14.2 |
| R10 | coturn 安全/配额/ACL 模板 | §5.2、§10.6 |

### 17.2 行为矩阵（修订后）

| 场景 | 期望 |
|------|------|
| 默认 vps | 现网一致 |
| 请求 host 且保持有效 host | 不读 turn_site；preflight 后可无票建 PC |
| 请求 host 但 escalate→hybrid + home | preflight→取票→bind→ACK；受 READY/fallback |
| relay + home READY | 双边 Home；VPS 无媒体 |
| home 非 READY，fallback=0 | 503；relay 立即失败提示 |
| home 非 READY，fallback=1 | Hub 整站签发 VPS（secret+host） |

### 17.3 关联文档状态

在 §10 闭环完成前，下列内容只能写「设计中 / 尚未实现」，不得使用现在时暗示已可配置：

- `nexartc-webrtc-ice-modes.md` 中 turn_site 章节  
- `nexartc-turn-mode-a-implementation.md` TURN 落点行与 Home 信令编排说明  
- `nexartc-install-deploy-guide.md` §5.4.2  
- 仓库根 `CLAUDE.md` TURN placement 索引（标注设计稿）

### 17.4 复核补丁

**第二轮（四项）**：快照原子绑定；home=`hub_rest`；WAN 权威初稿；`HOME_ENABLE_TURNS`。

**第三轮（三项 + 排版）**：

1. **高** — host escalate preflight/bind 协议（§5.3.4）。  
2. **中** — XFF 右向左剥离；`selfWanIp` 不一致不得 READY（§4.1.1）。  
3. **中** — Mode A「透明转发」改为 VPS 现网透明 / Home 会话编排。  
4. **排版** — §17 顺序恢复为 17.1→17.2→17.3→17.4。  

---

## 18. P0 本地实现代码评审（2026-07-26）

> 评审范围：本地未提交的 Hub、Web device、CAE、Home Edge 模板与测试改动；按本文 §4、§5.3、§9、§10、§14 逐项对照。  
>
> 评审结论：**当前只能认定为 P0 骨架，不能认定「七阶段 All done」或 `TURN_SITE=home` 已支持。** 默认 `TURN_SITE=vps` 与关联文档的「实现中 / 未标已支持」状态继续保持。下列阻断项关闭并完成真实 Home relay E2E 前，不得升级状态。
>
> **2026-07-26 续**：CR1–CR9 代码侧修复进行中（fail-closed phase SM、session 凭据、ACK 闸门、READY 撤销/Hub probe、密钥隔离、Peer ACL 等）；**状态仍为「实现中」**。CR10 真实 Home relay E2E 未完成前不得标「已支持」。

### 18.1 代码评审发现

#### CR1（阻断）— CAE 原生构建失败

- `WebRtcServerTransport.cpp:2007` 调用 `CaeSignalAgent::SendControlJson()`，但该方法位于 `CaeSignalAgent.h:37-50` 的 `private` 区域。
- Root / Nonroot 原生构建均报：`'SendControlJson' is a private member of 'CaeSignalAgent'`。
- 影响：当前 CAE 产物无法生成，§10.4 与后续 Home relay E2E 均无可验收运行物。

#### CR2（严重）— type=1 phase 可绕过 snapshotToken 门禁

- Hub `server/src/session-router.ts:491-548` 仅在 `phase === 'bind'` 时校验 token，缺少 phase、未知 phase 或旧版 type=1 均继续原样转发。
- CAE `WebRtcServerTransport.cpp:1964-2010` 仅对 `preflight` 提前返回；其他 phase 会继续普通协商并创建 PC。
- 结果：发送 `{type:1, ice_mode:'relay'}` 或任意 `phase:'x'` 可绕过 snapshot/site/generation 绑定，直接反证 §14.1「bind 缺/错 token 被拒」。
- 关闭要求：Hub 对 type=1 实施 fail-closed 的会话状态机，只接受当前 session 允许的 `preflight` / `bind`；不得以浏览器自报 phase/effective_mode 代替 Hub 保存的 CAE 权威状态。

#### CR3（严重）— READY 可自证，且故障后可能持续签票

- Edge Agent `home-turn-edge-agent.mjs:116-145` 仅用 `ss -uln` 命中 UDP 端口便自报 `READY`，没有公网 Allocate 或 relay 数据验证。
- Hub `home-turn-edge.ts:244-291` 接受 Agent 自报 `EXTERNAL_PROBING` / `READY`，并允许直接跳到 `READY(generation)`；未实现 §4.2 要求的 Hub/指定探测点公网探测。
- 已 READY 的 Edge 上报 `SEEN` / `APPLYING` 或空 `appliedWanIp` 时，`home-turn-edge.ts:228-291` 会刷新 `lastHeartbeatAt`，却不撤销旧 READY。Agent/coturn restart 失败后可因持续心跳而无限保持旧 generation，不会触发 stale。
- 关闭要求：任何 apply/restart/本地检查失败先撤 generation；只有公网 Allocate + relay 数据探测成功才能发布新 generation，并持续检查 coturn 健康。

#### CR4（严重）— CAE TURN 凭据未按 session 隔离或清理

- `CaeHubTurnBindStore` 只有全局 `m_latest`（`CaeHubTurnBind.h:36`），每次 bind 覆盖前一个会话。
- `WebRtcServerTransport.cpp:2652-2653` 使用 `GetBest("")`，没有把当前 Hub sessionId 带入建 PC。
- `CaeHubTurnBindStore::Clear()` 无调用点；`ClearSession` / `ClearAllSessions` 只清 pipe。结构也没有 generation 与绝对 expiry 校验。
- 影响：并发会话可能串用凭据；旧 Home 凭据可能污染后续 VPS/direct 会话；TTL 后重连可能继续选择已过期凭据。
- 关闭要求：凭据按完整 sessionId 保存和消费，绑定 snapshotId/generation/expiry，并在 detach、WS 断开、站点切换与失败路径清理。

#### CR5（高）— ACK 未形成建 PC 闸门，且 CAE 无条件 ACK 成功

- Hub `session-router.ts:512-548` 发出 `turn_site_bind` 后立即转发创建 PC 的 binary bind，没有 pending ACK 与超时失败状态。
- Web `device/src/main.ts:546-616` 先发送 bind，再异步等待 ACK；未 await 结果，超时还返回成功，`ACK=false` 只记录日志。
- CAE `CaeSignalAgent.cpp:599-650` 不严格校验 session/site/snapshot/generation/host/expiry；即使 `hasCreds=false` 仍发送 `ok=true`。
- 缺少有效 Home 凭据时，`WebRtcServerTransport.cpp:2691-2710` 还可能回退 CAE 本地 VPS HMAC，形成浏览器 Home、CAE VPS 的单边 TURN。
- 关闭要求：ACK 必须关联当前 sessionId + snapshotId；Hub 收到有效 ACK 后才转发建 PC bind，失败/超时双方均不得留下 PC。

#### CR6（高）— 合法有效 host/p2p 被 token 校验阻断

- join 冻结失败时，`session-router.ts:427-456` 明确允许 host/p2p 继续，但所有显式 `phase=bind` 又在 `:492-510` 无条件要求 snapshot token。
- 复现场景：`TURN_SITE=home`、Edge offline、固定 DNAT 有效、preflight 返回 host；Web 发送无 token bind 后收到 `SESSION_SNAPSHOT_REQUIRED`，无法建 PC。
- 即使 Home 当时 READY，Web 也会给有效 host bind 附带 token，使 host 错误依赖 Edge generation。
- 关闭要求：Hub 保存 CAE preflight 的权威有效模式；仅 effective hybrid/relay 要求 ticket/token，effective host/p2p 允许同 session 的无票 bind。

#### CR7（高）— STALE / OFFLINE / 超时恢复未闭环

- `device/src/main.ts:507-527` 对 `SNAPSHOT_STALE` 只清缓存，不重新 join/freeze；hybrid 随后仍可能继续发送旧 bind。
- HOME offline 的 hybrid 降级只修改全局 override，`startWebRtcNegotiation()` 的局部 `effective` 仍为 hybrid，发送的 bind 与浏览器最终 ICE 配置不一致。
- preflight 超时直接采用请求模式；丢失 host→hybrid 结果时可能造成浏览器 host、CAE hybrid。
- Hub 拒绝后仅清 TURN cache，`webrtcBindSent` 保持 true，当前会话无法重试。
- Home join 冻结失败后没有错误快照：有 VPS 配置时 credentials 返回 `SESSION_SNAPSHOT_REQUIRED`，无 VPS 配置时入口返回 `TURN_NOT_CONFIGURED`，通常无法得到设计要求的 `HOME_TURN_OFFLINE`。
- 关闭要求：显式实现 rejoin/refreeze/refetch/rebind 的有限重试；所有失败均 fail-closed，并在一次状态转换中同步 effective mode、缓存与 bind guard。

#### CR8（高 / 安全）— 密钥与 coturn 网络隔离门禁未完成

- `server/src/config.ts` 只读取配置，未强制 `HOME_TURN_SECRET !== TURN_SECRET`，也未强制 `EDGE_TOKEN !== AGENT_TOKEN`。
- `test/turn/home-edge/turnserver.conf.template:17-21` 的 Peer ACL 全部被注释；持有合法 TURN 票的用户可把 Edge 当作访问路由器/IoT 等家庭内网的 UDP 跳板。
- `home-turn-edge.ts:163-166` 使用普通字符串比较 EDGE_TOKEN；`/turn-edge` 没有限流，错误 token 后连接也不关闭。
- `/api/v1/home-turn-edge` 无鉴权并设置 `Access-Control-Allow-Origin: *`，暴露 WAN/LAN、generation、状态与 relay 端口，不符合 §6.4 Admin 语义。
- 公网 IPv4 判定仍接受 `198.51.100.0/24`、`203.0.113.0/24`、`198.18.0.0/15` 等保留地址，不满足 §4.1.1 的公网单播拒绝规则。

#### CR9（中）— Home→VPS fallback 未以冻结 snapshot 为唯一权威

- `session-router.ts:512` 使用 `snap.site === 'home' || TURN_SITE === 'home'`。
- 当全局配置为 Home、但 join 已整站 fallback 并冻结为 VPS snapshot 时，仍走 Hub 下发凭据分支，与 snapshot 的 `credentialSource='cae_local_hmac'` 及 §5.3.3 不一致。
- 关闭要求：bind 后的所有 host/secret/TURNS/credentialSource 决策只读取已校验的 snapshot，不再读取全局请求站点补判。

#### CR10（阻断验收）— Phase 6/§10.7 没有真实 Home E2E

- `test_home_turn_unit.mjs:12-15` 在模块导入前固定 `TURN_SITE=vps`。
- 名为 `home snapshot when edge READY` 的用例（`:109-118`）没有冻结或签发 Home snapshot，只 force 一个测试 Edge 后直接验证通用 HMAC。
- `run_all_tests.sh` 仅追加该单元脚本；在没有 Home Edge、coturn、CAE 的情况下仍可全部通过。
- 尚未验证：双方 selected pair 属 Home、VPS 无本会话媒体、跨站凭据失败、CAE LAN relay、hairpin、generation 撤销、ACK 超时、Hub/Edge/coturn 重启与配额耗尽。

### 18.2 §10 七项门禁复核

| §10 项 | 本轮状态 | 未关闭依据 |
|--------|----------|------------|
| 1. 身份与密钥隔离 | **未完成** | 未强制 secret/token 不同；无跨站鉴权失败用例（CR8、CR10） |
| 2. Edge READY 状态机 | **未完成** | Agent 自报 READY；无公网 relay probe；故障不撤 generation（CR3） |
| 3. Hub snapshot / 原子签发 | **未完成** | type=1 可绕过 token；fallback 不完全以 snapshot 为权威（CR2、CR9） |
| 4. CAE 对齐 | **未完成** | 构建失败；全局凭据；无效 bind 仍 ACK；无 ACK 闸门（CR1、CR4、CR5） |
| 5. Web 对齐 | **未完成** | host 无票路径被拒；STALE/OFFLINE/超时未闭环（CR6、CR7） |
| 6. coturn 安全部署 | **未完成** | Peer ACL 未启用；公网/健康探测不足（CR3、CR8） |
| 7. 端到端验收 | **未完成** | 只有 VPS 模块单测，没有真实 Home 强制 relay E2E（CR10） |

### 18.3 本轮验证结果

| 命令 / 检查 | 结果 | 说明 |
|-------------|------|------|
| Web server `npm run build` | 通过 | TypeScript / server build 通过 |
| Web device `npm run build` | 通过 | TypeScript / Vite production build 通过 |
| `node --import tsx ../../test/turn/test_home_turn_unit.mjs` | 通过 | 覆盖不足，不能代表 Home E2E（CR10） |
| `node --check test/turn/home-edge/home-turn-edge-agent.mjs` | 通过 | 仅语法检查 |
| `git diff --check` | 通过 | 根仓与相关子仓无 whitespace error |
| CAE `externalNativeBuildRootDebug` / `externalNativeBuildNonrootDebug` | **失败** | `SendControlJson` private 访问错误（CR1） |

### 18.4 建议关闭顺序

1. 修复 CAE 构建；Hub type=1 改成按 session 的 fail-closed phase 状态机，先关闭 token 绕过。  
2. CAE 凭据改为 session/snapshot/generation/expiry 绑定，并覆盖全部清理路径。  
3. Hub 建立 pending bind/ACK 闸门；CAE 严格验证后才 ACK，禁止 Home 静默回退 VPS。  
4. 修复有效 host/p2p 无票路径，以及 Web STALE/OFFLINE/超时的重走流程。  
5. 实现 generation 撤销、公网 Allocate + relay 数据探测和持续 coturn 健康检查。  
6. 强制密钥隔离，启用最小 Peer ACL，补 Edge 鉴权/限流/Admin 保护与完整公网地址判定。  
7. 最后执行 §14 的真实强制 relay E2E；所有用例通过后，才允许将 §10 或关联文档状态改为「已支持」。  

---

## 19. 第二轮本地实现代码复核（2026-07-26）

> 复核背景：实现方声明 CR1–CR9 已关闭、CR10 仅剩真实 Home relay E2E。本轮重新读取最新未提交改动，并实际执行 Web 构建、单元测试、CAE 原生构建及针对性反例。  
>
> 当前结论：**“CR1–CR9 全部关闭”不成立。CR8、CR9 可在代码层关闭；CR1–CR3 未关闭；CR4–CR7 仅部分关闭；CR10 未关闭。** §18 保留为首轮基线，本节为更新后的当前评审状态。文档继续保持「实现中 / 未标已支持」。

### 19.1 CR1–CR10 二审状态

| CR | 二审状态 | 结论摘要 |
|----|----------|----------|
| CR1 | **未关闭** | `SendControlJson` 已公开，但 CAE 最终链接失败，仍无可用 native 产物 |
| CR2 | **未关闭** | 正常 type=1 已 fail-closed，但重复 `webrtc_json` 可利用 Hub/CAE 解析差异绕过 phase |
| CR3 | **未关闭** | Agent 仅报 LOCAL_READY 已完成；当前 probe 可被普通 TCP 服务误判，且 Node 20 Agent 缺 WebSocket |
| CR4 | **部分关闭** | session map/精确 `Get` 已完成；session key 仍由浏览器 payload 控制，清理仍有遗漏 |
| CR5 | **部分关闭** | Hub pending ACK gate 已完成；ACK 的 snapshot/expiry/generation 与 Home fail-closed 仍不严格 |
| CR6 | **部分关闭** | host/p2p 已不要求 token；但存在 snapshot 时仍读取站点并可能等待 Home ACK |
| CR7 | **部分关闭** | preflight timeout 已 fail-closed；OFFLINE、普通网络错误和 rejoin/refreeze 仍未闭环 |
| CR8 | **代码层关闭** | secret/token 隔离、timing-safe token、Admin 鉴权、Peer ACL、主要保留地址拒绝已落地 |
| CR9 | **关闭** | bind 的 Home/VPS 凭据分支已只依据 `snap.site` |
| CR10 | **未关闭** | 单测增强，但真实 Home 双端 relay、selected pair、VPS 无媒体等仍未验收 |

### 19.2 未关闭与部分关闭项

#### 19.2.1 CR1 — 可见性错误已修，但 CAE 仍链接失败

- `CaeSignalAgent.h:37-38` 已将 `SendControlJson()` 移到 public，§18 记录的 private 编译错误已关闭。
- `cae_service/CMakeLists.txt:1-11` 依赖 `aux_source_directory`，未显式加入新文件 `CaeHubTurnBind.cpp`；当前已有 CMake/Ninja graph 没有该对象。
- 实测以下命令均在最终链接阶段失败：
  - `./gradlew :app:externalNativeBuildRootDebug --no-daemon`
  - `./gradlew :app:externalNativeBuildRootDebug --rerun-tasks --no-daemon`
- 缺失符号为 `CaeHubTurnBindStore::{GetInstance,Set,Get,Clear,ClearAll}`。在标准本地构建命令通过前，CR1 不得关闭。

#### 19.2.2 CR2 — type=1 仍存在解析差异绕过

- `server/src/webrtc-frame.ts:30-46` 读取第一个 `webrtc_json`；若其不是 type=1，返回 `null`。
- `session-router.ts:543-546,681-686` 仅对成功解析出的 type=1 运行 phase 状态机；`null` 被当作普通 binary 转发。
- CAE `CaeAppCtrlCmdUtils.cpp:79-86` 将参数写入 map，重复 key 以后一个值覆盖前一个。
- 反例：

  ```text
  command=1005
  &webrtc_json=<type=2>
  &webrtc_json=<type=1, ice_mode=relay, 无 phase>
  ```

  Hub 对该帧的 `parseWebRtcControlFrame()` 结果为 `null`，CAE 却取最后一个 type=1；而 `WebRtcServerTransport.cpp:1963-2010` 仍只特殊返回 preflight，缺/未知 phase 会继续建 PC。
- 关闭要求：Hub 与 CAE 使用一致的参数解析规则，拒绝重复 `webrtc_json`；对无法分类的 WebRTC request fail-closed，CAE 侧也必须独立拒绝缺/未知 phase。

#### 19.2.3 CR3 — probe 不是 TURN/relay 探测，Agent 还有运行时阻断

- `home-turn-edge.ts:224-255` 在 TCP connect 成功后，即使 UDP STUN 静默、UDP error 或 `send` 抛错也执行 `done(true)`。
- 实测仅启动普通 TCP server（无 UDP、无 STUN/TURN、无 Allocate），结果仍为：

  ```json
  {"tcpOnlyNoTurn":true,"probeResult":true}
  ```

- 当前没有携带临时凭据的 TURN Allocate，也没有 relay 端口或数据路径验证；3478 被其他 TCP 服务占用、UDP/relay 映射失败时仍可发布 READY。
- `home-turn-edge-agent.mjs:184` 直接使用全局 `WebSocket`，但未 import 实现或声明最低 Node 版本。当前环境 Node `v20.20.2` 中 `typeof WebSocket === 'undefined'`；systemd 会在 WAN 探测后反复抛 `ReferenceError`，`node --check` 无法发现。
- Agent 只在 apply 时执行一次 `localReadyCheck()`；READY 后 coturn 崩溃仍会持续上报 LOCAL_READY，Hub 缺少持续健康撤销。
- 关闭要求：使用真实 TURN Allocate（生产还需 relay 数据探测）；补充持续健康检查；显式引入 WebSocket 客户端或强制并验证具备全局 WebSocket 的 Node 版本。

#### 19.2.4 CR4 — map 已按 session 隔离，但 session 不是信令层权威值

- 已完成：`CaeHubTurnBind.h:39` 使用完整 session map；`CaeHubTurnBind.cpp:11-31` 精确 Set/Get；`WebRtcServerTransport.cpp:2652-2654` 精确查询。
- 未完成：CAE 查找 key 来自浏览器 JSON 的 `sessionId`（`WebRtcServerTransport.cpp:1957-1960`），不是 NXS1 framing / pipe 的权威 session。Hub 解析了 `parsed.sessionId`，但没有校验或重写为外层 sessionId。
- 结果：B 会话可通过自己的合法 snapshot token 完成 Hub 校验，却在 binary payload 写入 A 的 sessionId，使 CAE 查询 A 的凭据。
- `CaeSignalAgent.cpp:808-822` 的 `DisarmPipe()` 只删 pipe，不清 TURN bind；该路径由 socket 析构、Reset、CloseOldClient 使用。
- 无效 replacement bind 与 CreatePC 检测到过期 entry 后也未清旧 entry。
- 关闭要求：sessionId 只能取自 Hub framing/pipe；Hub 必须校验或重写 payload；覆盖 Disarm、replacement 失败与过期拒绝清理。

#### 19.2.5 CR5 — ACK gate 已建立，但关联与有效性仍可 fail-open

- 已完成：Hub 对 Home 保存 pending bind，收到 CAE ACK 后才转发 binary；ACK timeout 不再放行。
- `turn-session-state.ts:106` 仅在收到的 snapshotId 非空时比较；空 snapshotId 会释放 pending bind。针对性调用结果为：

  ```text
  EMPTY_SNAPSHOT_ACK_ACCEPTED
  ```

- `session-router.ts:178` 与 `device/src/main.ts:670-674` 均使用 `msg.ok !== false`，缺少 `ok` 也被视为成功；Web 不核对 ACK 的 sessionId/snapshotId。
- CAE Home ACK 只要求 `expiryUnix > 0`，不要求 `expiryUnix > now`；generation 虽已解析，但不要求大于 0，也未与 request snapshot 关联。
- `WebRtcServerTransport.cpp:2653-2730` 只有成功查到且标记为 Home 的 map entry 才禁止 VPS fallback；payload sessionId 缺失/篡改时仍可进入本地 VPS HMAC/静态凭据分支。
- 关闭要求：Hub、Web、CAE 三处都要求显式 `ok === true` 和 sessionId/snapshotId 完全相等；CAE 在 ACK 前验证未过期、generation、当前 session，并在 Home 上下文缺 bind 时明确拒绝本地 VPS fallback。

#### 19.2.6 CR6 — host/p2p 无 token 已完成，但仍读取 turn_site

- `session-router.ts:597-601` 对无需 ticket 的 host/p2p 仍读取当前 session snapshot。
- 随后的 `:628-657` 若 snapshot 为 Home，仍下发 Home 凭据并等待 ACK；若 snapshot 为 VPS，也会下发 VPS 元数据。
- 因此“无 token bind”已修复，但“有效 host/p2p 不读取 turn_site、不依赖 Home Edge/ACK”仍不成立。
- 关闭要求：`needsTurnTicket(effective) === false` 时完全跳过 snapshot、turn_site_bind 与 ACK gate，直接按 Hub 已保存的权威 preflight 结果转发 bind。

#### 19.2.7 CR7 — 超时已 fail-closed，但 OFFLINE/恢复仍不一致

- 已完成：preflight timeout 返回失败并 teardown；`SNAPSHOT_STALE` 不再继续旧 bind；ACK timeout 返回失败。
- `device/src/main.ts:532-535` 对非结构化 HTTP/NETWORK 错误，hybrid 仍返回成功并继续建连，可能形成浏览器无 TURN、CAE 有 TURN。
- HOME offline 时 Web 将 hybrid 改成 host（`:518-530`），但：
  - 请求原本是 hybrid 时没有执行 host preflight，Hub 会返回 `PREFLIGHT_REQUIRED`；
  - 请求 host 已由 CAE escalate 为 hybrid 时，Hub 权威 effective mode 仍是 hybrid，缺 ticket 的“降回 host”仍会被拒绝。
- STALE 当前只 teardown + 提示 reconnect，没有实现文档要求的重新 join/freeze/refetch/rebind 有限流程。
- `server/src/main.ts:133-141` 仍在解析 sessionId 前调用 `isTurnConfigured()`；Home 没有 VPS 配置时会先返回 `TURN_NOT_CONFIGURED`，遮蔽后续正确的 `HOME_TURN_OFFLINE`。
- 关闭要求：普通网络/HTTP 错误同样 fail-closed；明确实现由 Hub 认可的 hybrid→host 降级状态转换，或直接失败；将 session-bound 错误判定移到通用配置预检之前。

### 19.3 本轮确认已关闭项

#### CR8 — 代码层关闭

- `assertHomeTurnConfigSafe()` 已在 Hub 启动时执行，强制 Home/VPS secret 与 Edge/Agent token 隔离。
- EDGE_TOKEN 已使用 `timingSafeEqual`，错误连接关闭并记录失败计数。
- Home Edge Admin API 已加入 CMS Admin / STREAM_TOKEN 鉴权并移除公开 CORS。
- coturn 模板已启用 deny-all + CAE LAN allowlist，主要保留/文档/benchmark IPv4 已拒绝。
- ACL 对真实双端 relay/hairpin 的可用性仍须纳入 CR10 E2E，不据此把整体 P0 标为完成。

#### CR9 — 关闭

- `session-router.ts:627-669` 的 CAE 凭据与元数据分支已只读取冻结后的 `snap.site`，不再使用全局 `TURN_SITE` 补判。
- Home→VPS fallback 因而保持 snapshot 的 VPS `credentialSource='cae_local_hmac'` 语义。

### 19.4 测试与构建复核

| 检查 | 二审结果 | 说明 |
|------|----------|------|
| Server `npm exec -- tsc --noEmit` | 通过 | TypeScript 类型检查通过 |
| Device `npm exec -- tsc --noEmit` | 通过 | TypeScript 类型检查通过 |
| Server `npm run build` | 通过 | tsc + Vite production build 通过 |
| Device `npm run build` | 通过 | tsc + Vite production build 通过 |
| `test_home_turn_unit.mjs` | 通过 | Home snapshot/revoke/地址等覆盖增强；未覆盖上述跨解析器、ACK、真实 relay 反例 |
| Edge Agent `node --check` | 通过 | 仅语法；Node 20 WebSocket 运行时失败 |
| CAE `externalNativeBuildRootDebug` | **失败** | 最终链接缺 `CaeHubTurnBindStore` 实现 |
| 纯 TCP 假 TURN probe | **错误通过** | `probeHomeTurnEdge()` 返回 true |
| 空 snapshotId ACK | **错误通过** | pending bind 被释放 |

补充：单测 `assertHomeTurnConfigSafe rejects equal secrets` 在模块已按 `TURN_SITE=vps` 导入后调用，源码注释也说明该调用是 no-op；它没有实际断言 equal-secret 启动失败。隔离进程以 `TURN_SITE=home` 实测时，当前代码能够正确拒绝相同 secret，因此 CR8 的代码结论不受影响，但测试名与覆盖应修正。

### 19.5 P0 状态结论

1. §10.2 Edge READY、公网探测仍未闭环（CR3）。  
2. §10.3 snapshot/原子签发仍存在 phase 解析绕过（CR2）。  
3. §10.4 CAE 仍无可用构建，session/ACK 绑定也未闭环（CR1、CR4、CR5）。  
4. §10.5 Web 的有效 host 与失败恢复仍不满足正文时序（CR6、CR7）。  
5. §10.7 真实 Home relay E2E 未执行（CR10）。  

因此，本轮不得将 §10 七项勾为完成，也不得将关联文档状态升级为「已支持」；当前继续保持 **「实现中」**。

### 19.6 二审后修复备注（同日续）

针对 §19.2 已落地的代码层修复（**仍不得标「已支持」**；CR10 真实 Home relay E2E 未跑）：

| CR | 修复摘要 |
|----|----------|
| CR1 | `CMakeLists.txt` 显式加入 `CaeHubTurnBind.cpp`；`externalNativeBuildRootDebug` 链接通过 |
| CR2 | Hub `classifyWebRtcControlFrame` 拒绝重复 `webrtc_json` / 非 type=1；CAE `ParseCommand` 拒绝重复 key；缺/未知 phase 拒建 PC |
| CR3 | `probeHomeTurnEdge` 改为 UDP STUN + 认证 Allocate；Agent 加载 `ws`；心跳持续 `localReadyCheck`；READY 周期 reprobe |
| CR4 | Hub 重写/校验 payload `sessionId`；CAE 用 pipe `LookupHubSessionId`；`DisarmPipe`/失败 replacement/过期清 bind |
| CR5 | ACK 要求 `ok===true` 且 snapshotId 精确匹配；CAE expiry>now、generation>0；Web 校验 session/snapshot |
| CR6 | host/p2p 完全跳过 snapshot / `turn_site_bind` / ACK |
| CR7 | hybrid/网络错误 fail-closed；sessionId 路径先于 `isTurnConfigured()` |
| CR8/9 | 保持关闭；子进程覆盖 equal-secret；单测覆盖 TCP 假 TURN、空 snapshot ACK、重复 webrtc_json |

**残留**：CR10 真实双端 Home relay / selected pair / VPS 无媒体；CR3 生产级 relay 数据路径探测；CR7 自动 rejoin/refreeze 有限状态机。

---

## 20. 第三轮代码复核与修复闭环（2026-07-26）

> 复核背景：实现方按 §19 二审项修复 CR1–CR7 后再次提交本地未提交改动。本轮除重新执行 Web、CAE 构建外，还使用协议反例与本仓真实 coturn 验证 type=3/4/5、标准 TURN 401、allocation 回收、READY 撤销及 effective mode 权威绑定。  
>
> 状态口径：本节的“关闭”仅表示**本地代码与自动化反例层面关闭**；CR10 真实 Home 双端 relay E2E 尚未完成，因此文档总状态继续保持 **「实现中（未标已支持）」**，§10 仍不得整体勾选完成。

### 20.1 三审发现（修复前）

1. **CR2：合法 WebRTC 信令被误拒绝**  
   `classifyWebRtcControlFrame()` 将所有非 type=1 的 `command=1005` 判为错误，但浏览器 Answer/Candidate/Close 分别是 type=3/4/5。结果是 CAE 发出 Offer 后，Hub 会拒绝 Answer，正常 Hub WebRTC 无法建连。原单测把“非 type1 全拒绝”写成期望，形成假绿。
2. **CR3：标准 coturn 401 解码错误**  
   STUN `ERROR-CODE` 被按 `class*256+number` 解析；标准 401 `[0,0,4,1]` 因而得到 1025，真实 coturn 永远不能通过认证 Allocate probe。
3. **CR3：probe allocation 泄漏及 READY 撤销不完整**  
   Binding、challenge、Allocate 使用不同 UDP socket，且成功后未发送同五元组 `Refresh(LIFETIME=0)`；30 秒 reprobe 会与 coturn 600 秒默认 allocation、`user-quota=12` 冲突。已有 READY 收到 `SELF_WAN_MISMATCH` / `WAN_REJECTED` 时又在撤 generation 前返回，仍可短时继续签票。
4. **CR2/CR6：Hub effective mode 与 bind payload 脱钩**  
   Hub 按保存的 host/p2p 判断“不需要票”，却将浏览器自报 `ice_mode` 原样交给 CAE。可构造 `preflight=p2p → bind=relay`，绕过 ticket 后让 CAE 尝试本地 VPS TURN。
5. **CR4/CR5：CAE session 与 Home TURN fail-closed 仍有缺口**  
   非 Hub socket 可回退 payload `sessionId` 查询 Hub bind map；`turn_site_bind` 未确认当前 active pipe；Hub hybrid/relay 缺 bind 时仍可能走本地 VPS。严格 Home 仅有 Hub 凭据时，Hub TURN 也未计入关闭 UDP mux 的条件，可能 ACK 成功但 CAE 零 relay candidate。
6. **兼容性：phase 强制误伤 direct/VPS**  
   CAE 对所有 type=1 无条件要求 `phase`，现有直连页 / SDK 的无 phase type=1 因而不能建 PC。
7. **CR7：STALE 只有 teardown，没有有限 rejoin/refreeze**。

### 20.2 本轮修复落点

| CR | 修复结果 |
|----|----------|
| CR1 | `CaeHubTurnBind.cpp` 已进入 CMake；Root / Nonroot native 最终链接均通过 |
| CR2 | Hub 仅让 type=1 进入 phase 状态机；type=3/4/5 透明转发；type=2、未知、缺失/非整数 type 与重复 `webrtc_json` fail-closed；CAE 重复 key 继续独立拒绝 |
| CR2/6 | `validateBindMode()` 强制 `ice_mode === effective_mode === Hub effectiveMode`；host/p2p 必须已有 preflight 结果，hybrid/relay 仅可用一致字段建立初始权威模式，关闭无票 mode-switch 绕过 |
| CR3 | 标准 `ERROR-CODE=class*100+number`；Binding→401→认证 Allocate→Refresh(0) 全程复用单 UDP socket；Refresh 失败不发布 READY，并限制 LOCAL_READY 重试频率 |
| CR3 | 增加 probe epoch，revoke/WAN 变化后旧异步 probe 不得恢复 READY；`SELF_WAN_MISMATCH` / `WAN_REJECTED` 立即撤 generation 并关闭 Edge WS，Agent 重连后重新取得权威 WAN |
| CR4 | Hub 校验/重写外层 session；ACK/preflight result 必须属于当前 agent 的 attached session；CAE bind map 只使用 `LookupHubSessionId()` 的 Hub pipe session，不再用浏览器 payload 查询 |
| CR4/5 | `turn_site_bind` 必须命中 active pipe，检查与 Set/Clear 在 session 锁内完成；Hub TURN 模式缺 bind、Home 凭据过期/不完整、VPS 元数据无本地配置均拒绝建 PC，禁止静默 VPS fallback |
| CR5 | Home ACK 要求显式 expiry、`expiry>now`、`generation>0`、非空 snapshot/凭据；VPS 元数据要求精确 `turnSite=vps + credentialSource=cae_local_hmac`；Hub-issued Home TURN 也会关闭 per-PC UDP mux |
| CR6 | Hub 与 CAE 的有效 host/p2p 都不读取/校验遗留 TURN bind，不依赖 snapshot、Edge READY 或 ACK |
| CR7 | Web 对 `SNAPSHOT_STALE` 执行一次有界 `teardown → reconnect/join → refreeze → refetch → preflight/bind`；旧异步 HTTP/ACK 结果按 session/generation 丢弃，重复 STALE 后 fail-closed，不无限重连 |
| 兼容性 | CAE 仅对 Hub pipe 强制 `phase=preflight|bind`；旧 direct/VPS 无 phase type=1 保持原 CreatePC 行为 |
| Edge 部署 | Agent 明确支持 Node.js ≥20，加入 `ws` lockfile；部署步骤使用 `npm ci --omit=dev` |

### 20.3 新增反例门禁

`test_home_turn_unit.mjs` 新增或修正以下断言：

- type=3/4/5 必须为透明转发，type=2/未知/缺失/非整数/非对象必须拒绝；
- 重复 `webrtc_json` 必须拒绝；
- `preflight=p2p/host → bind=relay`、`ice_mode != effective_mode` 必须拒绝；
- 标准 TURN 401 必须进入认证 Allocate；四个请求必须使用同一 UDP 源端口；
- Refresh 必须携带 `LIFETIME=0` 与 `MESSAGE-INTEGRITY`；Refresh 失败时 probe 必须返回 false；
- 已有 READY 遇到 `SELF_WAN_MISMATCH` 或权威 WAN 拒绝时，state 必须变为 OFFLINE、generation 必须清零；
- 空 snapshot ACK、缺失 `ok===true`、equal-secret 子进程等既有门禁继续通过。

### 20.4 构建与协议验证

| 检查 | 结果 | 说明 |
|------|------|------|
| Server `npm run build` | **通过** | TypeScript + Vite SSR bundle |
| Device `npm run build` | **通过** | TypeScript + Vite production bundle；build 标识 `20260726B` |
| `test_home_turn_unit.mjs` | **ALL PASS** | 包含 §20.3 反例 |
| 本仓真实 coturn `probeHomeTurnEdge()` | **通过** | 标准 401、认证 Allocate、同五元组 Refresh(0)，日志 `released=yes` |
| TCP-only 假 TURN | **正确失败** | 无 UDP STUN/TURN 响应不得 READY |
| Edge Agent `node --check` | **通过** | `ws` 依赖已 lock |
| CAE `externalNativeBuildRootDebug` | **BUILD SUCCESSFUL** | `CaeHubTurnBind` 编译并最终链接 |
| CAE `externalNativeBuildNonrootDebug` | **BUILD SUCCESSFUL** | 同上 |

### 20.5 CR1–CR10 当前状态

| CR | 三审修复后状态 | 说明 |
|----|----------------|------|
| CR1 | **代码层关闭** | Root / Nonroot native 链接通过 |
| CR2 | **代码层关闭** | 类型矩阵、重复参数、phase 与 effective mode 均 fail-closed |
| CR3 | **代码层关闭** | 真实 coturn Allocate/回收及 READY 撤销通过；实际 relay 数据路径并入 CR10 |
| CR4 | **代码层关闭** | Hub outer framing / active pipe 为唯一 session 权威，清理路径闭合 |
| CR5 | **代码层关闭** | 严格 ACK、Home 无 VPS fallback、Hub TURN UDP mux 闭合 |
| CR6 | **代码层关闭** | 有效 host/p2p 不读取 turn_site；mode-switch 绕过关闭 |
| CR7 | **代码层关闭** | 全部错误 fail-closed；STALE 有一次有界 rejoin/refreeze |
| CR8 | **代码层关闭** | 密钥、token、Admin、ACL 与地址拒绝保持关闭 |
| CR9 | **代码层关闭** | site 决策仍只取冻结 snapshot |
| CR10 | **未关闭** | 尚缺真实 Home 硬件双端强制 relay、selected pair、VPS 无媒体及 hairpin/ACL 验收 |

因此，§19.6 中“非 type=1 全拒绝”的表述由本节纠正为“**type=1 编排；type=3/4/5 透明转发；其他客户端方向类型 fail-closed**”。当前可以认为 CR1–CR9 的本地代码项已关闭，但在 CR10 完成前，本文与关联部署文档仍必须保持 **「实现中 / 未标已支持」**。

---

## 21. 第四轮终审与竞态修复闭环（2026-07-26）

> 复核范围：在 §20 修复基础上，继续只检查 Home TURN Edge 的 Hub/device、Edge probe、CAE bind 三条关联时序，不扩展到其他媒体问题。本轮重点使用旧 socket、错 session/snapshot、Edge 撤代、UDP 丢包及 CAE detach/reattach 反例检查 TOCTOU 和默认 VPS 兼容性。
>
> 状态口径：以下“关闭”仍只表示本地代码、协议反例和构建层面关闭。CR10 真实 Home 硬件双端 relay / selected pair / VPS 无媒体 / hairpin 与 ACL 验收尚未完成，本文总状态继续保持 **「实现中（未标已支持）」**。

### 21.1 终审发现

1. **CR4/CR5：CAE 原始 `ok=true` 不等于 Hub 已释放 bind。** 错 snapshot、无 pending、ACK 超时或重复 ACK 时，`takePendingBindOnAck()` 返回空，但 Hub 仍可能向浏览器报告成功；同时带未知 sessionId 的旧 framed 消息会在 Agent 只有一个新会话时落入兼容回退。
2. **CR5：bind 校验与 ACK 释放之间存在 generation TOCTOU。** Home snapshot 在 bind token 校验通过后、等待 CAE ACK 期间被 Edge 撤销，旧代码仍可释放 pending bind 并创建 PC。
3. **CR7：旧 WSS 排队事件和重复取票可拆掉新会话。** 关闭/替换 socket 后，旧 `onmessage` 仍可把 `SNAPSHOT_STALE` 或 ACK 投到全局新 session waiter；Hub hybrid/relay 又在 join 和 negotiation 各执行一次 `force=true` 取票，成功与失败结果可互相竞争。
4. **CR4：Agent replacement / heartbeat timeout 清理不完整。** replaced Agent 的旧 binary 未统一核对当前 Agent WS；heartbeat sweep 直接删除 registry，导致 attached browser 不关闭，snapshot 与 turn-session state 无法进入 client cleanup。
5. **CR3：READY 可被非法 Agent 状态或 endpoint 漂移保活。** 未知/Hub-owned state、`LOCAL_READY` 配错误但非空 `appliedWanIp`，以及 LAN endpoint 改变，均可能保留旧 generation。
6. **CR3：外部 probe 只验证了请求，不认证响应。** Allocate/Refresh 成功响应未核对 UDP 来源、`MESSAGE-INTEGRITY` 和 `FINGERPRINT`；每阶段仅发一个 UDP 包，单包丢失会误撤 READY，Refresh 丢包还可能留下 allocation。
7. **CR4/CR5：CAE detach 与 bind store 非原子。** 旧 pipe 摘链后再清 store，可擦掉同 session 新 attach 已写入的 bind；仅凭 pipe 内 sessionId 又会把已关闭/摘链 Hub pipe 当 active，甚至误降级到 direct 兼容路径。Home 消息也未强制 `credentialSource=hub_rest` 和完整数值 schema。
8. **部署门禁：** Edge README 未明确把两个 unit 安装到 `/etc/systemd/system/`，也缺少 `systemctl daemon-reload`，首次部署可能无法 enable unit。

### 21.2 修复落点

| 范围 | 修复结果 |
|------|----------|
| Hub session 路由 | framed 消息携带 sessionId 时必须精确命中，只有真正无 framing 的单会话旧流量才允许兼容回退；Agent binary 还必须来自 registry 当前 WS |
| Hub ACK | ACK 必须关联当前 Agent、session 和冻结 snapshot；Home 仅在 exact pending bind 实际发送给 CAE 后回 `ok=true`；VPS 保留 metadata ACK 兼容，不误套 Home pending gate；负 ACK 同样按 snapshot 作用域清理 |
| Home generation | Home ACK 释放点同步再次执行 `assertSnapshotIssuable()`；若 generation 已撤销，取消 pending、返回 `SNAPSHOT_STALE`，不转发 binary，由 device 执行一次有界 rejoin/refreeze |
| Device WSS | 每个 handler 捕获创建时 socket identity；关闭时先失效并解除 handler，旧排队事件不得进入新会话；session/snapshot 不匹配的 ACK 直接忽略，不 drain 当前 waiter |
| Device 取票 | 移除 join 与 negotiation 的双 `force=true` 请求；Hub hybrid/relay 每次 negotiation 只执行一次 session-bound fetch；build 标识更新为 `20260726C` |
| Agent 生命周期 | heartbeat sweep 走统一 `forceDisconnectAgent()`，关闭 attached browser 并触发 snapshot/state cleanup；replaced Agent 的旧 binary fail-closed |
| Edge 状态机 | Agent 仅可报告 `SEEN/APPLYING/LOCAL_READY`；错误 state、错误 applied WAN、WAN/LAN endpoint 变化均先撤 generation 和 probe epoch；endpoint 改变后清 probe freshness，再重新进入外部探测 |
| Edge probe | 全部 STUN/TURN 请求带有效 FINGERPRINT；响应必须来自目标 IP/port；Binding/401 要求有效 FINGERPRINT，认证 Allocate/Refresh 还要求长期凭据 MESSAGE-INTEGRITY；同 txId 按 500ms 起始 RTO 有界指数重传 |
| CAE active identity | `CaeHubPipeSocket` 使用 atomic closed，并绑定不可变 full sessionId + 单调 pipe epoch；active 必须同时匹配 map pointer/session/epoch 且未关闭，inactive Hub pipe 不得降级成 direct |
| CAE 原子性 | bind Set/Get/Clear 与 pipe active 检查统一为 `session → bindStore` 锁序；Disarm/ClearSession/ClearAll 在同一 session 临界区清 store；SendControlJson/PostDisconnect 保持锁外 |
| CAE 建 PC | type=1 初收、长协商结束、CreatePC 起点和最终三表 commit 多点复核同一 pipe identity；TURN bind 以 active+store 原子读取；Home/VPS 均要求 request snapshot 与 bind snapshot 精确相等 |
| CAE schema | Home 强制 `credentialSource=hub_rest`，port/TTL/expiry/generation 必须为范围内整数且未过期；VPS 继续精确 `cae_local_hmac`，旧 direct/VPS 路径与 per-PC UDP mux 规则保持兼容 |
| Edge 部署 | README 明确 env 权限、unit 安装到 `/etc/systemd/system/`、`daemon-reload` 及 enable 顺序 |

### 21.3 新增反例与最终门禁

`test_home_turn_unit.mjs` 与针对性伪 Agent/Client 测试新增或确认：

- stale framed session 不得回退到唯一新 client；replaced Agent 旧 binary 不得转发；
- Home premature/wrong ACK 为失败且 binary 保持 0，exact ACK 后才由 0→1；默认 VPS metadata ACK 仍为 true 且 PC binary 正常转发；
- Home ACK 前撤 generation 后，binary 保持 0、pending 被清、浏览器收到 `SNAPSHOT_STALE`；
- WSS close 后手工触发已排队旧 handler，`lateControlDeliveredAfterClose=0`；
- Agent heartbeat sweep 后 attached client 被关闭并进入 session cleanup；
- 未知 Agent state、错误 applied WAN、LAN endpoint 改变均撤 generation；
- 无 MESSAGE-INTEGRITY、坏 FINGERPRINT、错误 UDP 源端口响应均不能通过 probe；丢首个 UDP 请求后以同 transaction 重传并成功；
- 标准 401、认证 Allocate、同源端口 Refresh(0) 与 allocation 回收继续通过。

| 检查 | 最终结果 | 说明 |
|------|----------|------|
| Server `npm run build` | **通过** | TypeScript + Vite SSR bundle |
| Device `npm run build` | **通过** | TypeScript + Vite production bundle；build `20260726C` |
| `test_home_turn_unit.mjs` | **ALL PASS** | 含状态、session/ACK、WSS 旧事件、MI/FP/源端口/重传反例 |
| Home/VPS 伪 Agent/Client 全链 | **通过** | Home exact ACK gate 与默认 VPS 兼容均通过 |
| 本仓真实 coturn probe | **通过** | 标准 401 + MI/FINGERPRINT + Allocate + 同五元组 Refresh(0)，`released=yes` / `realCoturnProbe=true` |
| Edge Agent `node --check` | **通过** | Node.js ≥20 + `ws` lockfile 路径 |
| CAE Root native | **BUILD SUCCESSFUL** | active identity、bind store 与 schema 代码最终链接 |
| CAE Nonroot native | **BUILD SUCCESSFUL** | 同上 |
| Web / CAE / coturn tracked diff check | **通过** | 无空白错误；未清理工作区内用户的其他未提交文件 |

### 21.4 CR 状态结论

- **CR1–CR2、CR6、CR8–CR9：** 维持 §20 的本地代码层关闭结论。
- **CR3：** Edge 状态、endpoint generation、响应认证、allocation 回收与 UDP 重传在本地代码/真实 coturn probe 层关闭；真实 relay 数据路径仍计入 CR10。
- **CR4：** Hub exact routing、Agent 生命周期、CAE pipe identity/epoch 与 session→store 原子清理在本地代码层关闭。
- **CR5：** ACK exact association、实际转发成功语义、ACK 前 generation 复核及 CAE snapshot/schema 门禁在本地代码层关闭。
- **CR7：** 旧 WSS 事件隔离、单次取票、一次有界 STALE recovery 在本地代码层关闭。
- **CR10：未关闭。** 仍需真实 Home 硬件双端强制 relay，保存双方 selected pair，确认 VPS 无本会话媒体，并完成 NAT hairpin / Peer ACL / relay 端口与配额验收。

因此，本轮不得把 §10 七项整体勾选完成，也不得把 Home Edge 部署路径标为生产“已支持”；主文档与关联文档继续保持 **「实现中 / 未标已支持」**。

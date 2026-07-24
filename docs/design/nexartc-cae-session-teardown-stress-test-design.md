# CAE 会话 Teardown / 多用户并发稳定性自动化测试设计

> 针对实网事故：WebRTC 多会话并发断开时 CAE abort（`Pure virtual function called!` / `Track is not open`），  
> 以及后续修复（同步 close、`m_rtpTeardownPause`、`IncidentLogArchiver`、Supervisor 配置）。  
> 目标：用可重复的自动化用例覆盖 **2–3 用户并发上下线** 与 **频繁撕连压测**，防止回归。

相关文档：

| 文档 | 关系 |
|------|------|
| [`cae-supervisor-remote-admin-design.md`](./cae-supervisor-remote-admin-design.md) §10–§11 | 自愈看门狗 + 事故快照 |
| [`nexartc-logging-design.md`](./nexartc-logging-design.md) | `[STAB]` / incidents 分析 |
| [`nexartc-turn-mode-a-implementation.md`](./nexartc-turn-mode-a-implementation.md) | Mode A Hub + device 信令 |
| `deps/datachannel-native/.../cae-receiver/` | 现有 C++ WebRTC 接收端样例 |
| `test/auto_browser_stability.js` / `test/turn/*.mjs` | 现有 Node/Puppeteer 测试 |

---

## 1. 问题与测试目标

### 1.1 已修复故障模式（被测对象）

根因摘要：

1. **最后一客户端异步 `closePeer` + 提前释放 RTP pause** → `sendFrame` 撞已关闭 Track → abort。  
2. **多客户端 ICE UDP mux**：A 关闭时 B 仍在 `sendFrame` → pure virtual。  
3. abort 后 Supervisor 若配置读错则自愈慢；事故日志被轮转覆盖。

修复后必须守住的不变量：

| ID | 不变量 |
|----|--------|
| I1 | 任意并发上下线过程中 **CAE Worker 不 abort**（无 `Pure virtual` / `Track is not open` 进 stdout） |
| I2 | 全程或恢复后 **Hub `deviceId` 仍在线**（`GET /api/v1/agents`） |
| I3 | 仍在线的 viewer **视频不长期中断**（decode/RTP 有增长；允许 tear 期间短暂 pause） |
| I4 | 最后一客户端离开后，媒体可停，但 **Agent 注册保持**（除非刻意测 Supervisor 重启） |
| I5 | 若发生自愈重启，`logs/incidents/` 出现对应快照（可选门禁） |

### 1.2 测试目标分层

```text
L1 功能用例（正确性）     — 少次数、断言状态机
L2 并发场景（竞态覆盖）   — 2–3 用户编排剧本
L3 撕连压测（稳定性）     — 高频 connect/disconnect，时长/轮次可配
L4 门禁（CI / 夜间）      — 失败即红；产出报告 + 可选 pull incidents
```

---

## 2. 实现选型：Node.js vs C/C++

### 2.1 结论（推荐）

| 角色 | 推荐技术 | 理由 |
|------|----------|------|
| **主推：编排 + 压测客户端** | **Node.js（`wrtc` / `werift` + `ws`）** | 易并发多实例、剧本化、与现有 `test/turn/*.mjs` 一致；贴近 **Web 客户端信令路径**（Hub `/ws` + SDP/ICE）；CI 友好 |
| **可选：浏览器真源回归** | **Puppeteer + 真实 Chrome**（少量并发） | 覆盖真实 `RTCPeerConnection` / H.264 decode；资源重，不适合 3 路×高频撕连 |
| **可选：原生栈对齐** | **C++（扩展 `cae-receiver` / libdatachannel）** | 与 CAE 同库，更容易复现 **mux / Track teardown** 类 native abort；适合专项复现，不适合快速写剧本 |

**默认落地路径：Node.js headless WebRTC 客户端（模拟 web）+ 轻量编排器。**  
C++ 作为 **Phase-2 专项复现工具**，不阻塞主门禁。

### 2.2 对比

| 维度 | Node.js（推荐主路径） | C/C++（cae-receiver 系） | Puppeteer Chrome |
|------|----------------------|---------------------------|------------------|
| 模拟 Web 信令 | 高（可走 Hub `/ws`） | 中（现样例偏直连 CAE WSS） | 最高（真 device 页） |
| 多实例并发成本 | 低（进程/Worker） | 中（进程） | 高（浏览器） |
| 复现 Pure virtual | 中高（真实 ICE/RTP） | **最高**（同 libdatachannel） | 中 |
| 开发速度 | 快 | 慢 | 中 |
| CI / 夜间压测 | 优 | 可 | 差（易 flaky） |
| 与现有仓库契合 | `test/*.mjs` 已有 | `cae-receiver` 已有 | `auto_browser_*.js` 已有 |

### 2.3 Node.js 技术选型细化

推荐栈（择一即可，优先可编译部署到开发机）：

1. **`werift`（纯 JS WebRTC）+ `ws`**  
   - 优点：无原生编译负担；适合信令/多会话剧本。  
   - 风险：媒体编解码能力弱于 Chrome；对「有画面」断言可用 RTP/stats 代替硬解。
2. **`wrtc`（node-webrtc）+ `ws`**  
   - 优点：更接近浏览器 RTP。  
   - 风险：需原生模块，ARM/CI 环境要锁版本。
3. **Puppeteer 仅用于 L1 冒烟（1–2 客户端）**，不进 L3 主压测。

Mode A 连接路径（与 device 页对齐）：

```text
Client_i
  → WSS Hub /ws  (token + join deviceId)
  → 转发 Offer/Answer/Candidate 至 CAE Agent
  → RTCPeerConnection (ice_mode=host|hybrid)
  → DataChannel control + 收视频轨
  → 主动 close / 杀进程 / 断 WSS（模拟异常下线）
```

### 2.4 C++ 专项路径（可选）

扩展 `cae-receiver`：

- 增加 Hub Mode A join（不只直连 CAE）。  
- 支持 `N` 个进程并行 + 脚本 `stress_teardown.sh` 编排 kill/restart。  
- 用途：当 Node 压测 **偶发** 复现 abort 时，用同库客户端钉死。

---

## 3. 测试架构

```text
┌─────────────────────────────────────────────────────────────┐
│  Orchestrator (Node.js)                                      │
│  - 读 env: HUB / STREAM_TOKEN / DEVICE_ID / ICE_MODE         │
│  - 启 N 个 SimulatedWebClient                                │
│  - 跑 Scenario / Stress 剧本                                 │
│  - 轮询 Probe: Hub agents / CAE liveness / log signatures    │
└───────────────┬─────────────────────────────┬───────────────┘
                │                             │
        ┌───────▼────────┐           ┌────────▼────────┐
        │ Client A/B/C   │           │ Probe (adb/ssh) │
        │ Hub+/ws+WebRTC │           │ Hub API / adb   │
        └───────┬────────┘           └────────┬────────┘
                │                             │
                ▼                             ▼
        ┌───────────────┐            ┌──────────────────┐
        │ Signal Hub    │◄──────────►│ CAE + Supervisor │
        └───────────────┘            └──────────────────┘
```

### 3.1 建议目录

```text
test/session_stress/
  README.md
  package.json                 # 独立或挂到现有 test 依赖
  lib/
    hub_client.mjs             # /ws join + 信令转发
    webrtc_client.mjs          # PC + DC + stats
    cae_probe.mjs              # adb liveness / log grep / incidents
    hub_probe.mjs              # GET /api/v1/agents
    report.mjs                 # junit/json 报告
  scenarios/
    F01_single_connect.mjs
    F02_graceful_disconnect.mjs
    ...
    C01_two_online_stable.mjs
    C02_one_online_one_flap.mjs
    C03_all_offline_together.mjs
    C04_three_staggered.mjs
    S01_flap_storm.mjs
  run_functional.mjs
  run_concurrent.mjs
  run_stress.mjs
  run_all.sh
```

环境变量（与现有 deploy env 对齐）：

```bash
HUB_BASE=https://120.79.21.28
STREAM_TOKEN=...
DEVICE_ID=device-mi9-001
ICE_MODE=host                  # host|hybrid|relay
ADB_SERIAL=9898d727            # 可选；无 adb 则只做 Hub 探针
STRESS_ROUNDS=200
STRESS_CLIENTS=3
INSECURE=1                     # 调试证书
```

---

## 4. 功能用例（L1）

每个用例：准备 → 动作 → 断言 → 清理。失败时 dump：Hub agents、CAE `[STAB] DisconnectClient`、最新 `incidents/`。

| ID | 名称 | 步骤 | 通过标准 |
|----|------|------|----------|
| F01 | 单用户上线 | 1 client join → ICE connected → 收 ≥1s 视频 stats | Hub `sessionCount≥1`；client `connected`；无 abort |
| F02 | 优雅下线 | F01 后 `pc.close()` + 关 WSS | Hub `sessionCount→0`；Agent 仍 registered；CAE 存活 |
| F03 | 异常断 WSS | F01 后直接 `ws.terminate()` | CAE 清理会话；Agent 仍在；无 abort |
| F04 | 异常杀 PC | F01 后丢弃 PC 不 close（模拟 tab crash） | 同 F03；允许超时清理 |
| F05 | 重连 | F02 后再上线 | 二次 `OnReady`；视频恢复 |
| F06 | STOP 命令 | DC 发 STOP / 协议 stop | 与 F02 等价清理 |
| F07 | ice_mode=hybrid | 同 F01（可选 TURN） | ICE 连通；路径可 relay |
| F08 | 触控冒烟 | connected 后发少量 touch | 无 DC 错误；CAE 无注入 flood crash |
| F09 | 注册保持 | 全程监控 Agent | `agent_registered` 不因单客户端离开变 false |
| F10 | 事故快照（负向可选） | 人为 `pkill` Worker | Supervisor 拉起；`incidents/` 新增目录 |

**功能门禁建议：** CI 跑 F01–F06（host）；夜间加 F07–F08。

---

## 5. 并发场景用例（L2）— 对标 teardown crash

> 核心：多 `conn_id` 同时存在时，交错 `DisconnectClient` / `sendFrame`。

### 5.1 剧本总表

| ID | 名称 | 编排（T 为秒） | 针对风险 |
|----|------|----------------|----------|
| C01 | 双用户同时在线稳态 | T0: A↑ B↑；T0–60 保持 | 双路 fan-out 稳定性 |
| C02 | 一在线一撕连 | T0: A↑ B↑；T10–T60: B 每 2s 下线上线；A 一直看 | **A 收流时 B teardown**（历史 abort） |
| C03 | 同时下线 | T0: A↑ B↑；T20: A↓ B↓ 同秒 | 最后客户端同步 close |
| C04 | 三用户交错 | T0: A↑B↑C↑；交错单下线/上线 | 3 路 mux + drain |
| C05 | 最后一人离开 | T0: A↑B↑；T15: A↓；T20: B↓ | 最后 peer 同步 close 路径 |
| C06 | 新用户踩旧 teardown | T0: A↑；T5: A↓ 同时 B↑ | close 与 create 竞态 |
| C07 | 双用户同时下线后立刻双上线 | T0: A↑B↑；T10: A↓B↓；T11: A↑B↑ | 端口/mux 复用 |
| C08 | 保持 1 用户 + 风暴第 2 用户 | A 常驻；B 0.5–1Hz flap × 120s | 高频单侧撕连 |
| C09 | 三用户同时下线 | A↑B↑C↑ → 同秒全↓ | CloseAll / 多 drain |
| C10 | Hub detach 风暴 | 只断 `/ws` 不关 PC（半开） | Agent/pipe 清理 vs PC |

图示（C02）：

```text
A  ████████████████████████████████████  (online)
B  ██░░██░░██░░██░░██░░██░░██░░██░░██   (flap)
   ↑ connect     ↑ disconnect
Probe: CAE pid 恒定；无 Pure virtual；A video_delta>0（除短 pause）
```

### 5.2 并发断言（每个 C* 共用）

1. **进程**：`adb` 上 `com.nexartc.cae.Server` pid 在场景期间不变；若变则记 `restarted`（失败，除非场景允许自愈）。  
2. **日志签名（负向）**：场景中 **不得** 新增  
   - `Pure virtual function called`  
   - `Track is not open`  
   - `terminating with uncaught exception`  
3. **正向日志（抽检）**：断开侧出现  
   `[STAB] WebRTC DisconnectClient done ... (sync close under pause`  
4. **Hub**：`deviceId` 始终在 `agents`；`sessionCount` 与在线 client 数大致一致（允许 2–3s 滞后）。  
5. **残留会话**：场景结束全部 client 清理后，`sessionCount==0`，60s 内 Agent 仍在。

---

## 6. 频繁上下线压测（L3）

### 6.1 参数

| 参数 | 默认 | 说明 |
|------|------|------|
| `STRESS_CLIENTS` | 2 或 3 | 并发模拟用户数 |
| `STRESS_ROUNDS` | 200 | 每 client 的 connect 次数（或全局回合） |
| `ONLINE_MS` | 1000–5000 随机 | 在线停留 |
| `OFFLINE_MS` | 200–2000 随机 | 离线间隔 |
| `PATTERN` | `stagger` / `sync` / `pair_flap` | 见下 |
| `DURATION_SEC` | 可选，与 rounds 二选一 | 时长模式 |

### 6.2 压测模式

| Pattern | 行为 |
|---------|------|
| `stagger` | 各 client 独立随机 flap（最常见） |
| `sync_all` | 全体同时上 → 同时下（打最后一员路径） |
| `pair_flap` | Client0 常驻；其余高频 flap（打 C02/C08） |
| `rolling` | 始终保持恰好 1 人在线，轮转身份 |

### 6.3 通过 / 失败

**通过：**

- 全程无 abort 签名；Worker pid 不变（或重启次数 = 0）。  
- Hub Agent 掉线累计时间 < 5s（允许瞬断重连）。  
- Orchestrator 退出码 0；报告 `fail_rounds=0`。

**失败即停（可选 `--fail-fast`）：**

- 探测到 abort 签名 / pid 消失且 30s 未恢复。  
- 自动 `adb pull …/logs/incidents` + 最新 `cae_server_*.log` 尾。

### 6.4 报告格式

```json
{
  "suite": "S01_flap_storm",
  "started_at": "...",
  "clients": 3,
  "rounds": 200,
  "pattern": "pair_flap",
  "cae_pid_start": 18564,
  "cae_pid_end": 18564,
  "abort_signatures": 0,
  "hub_offline_events": 0,
  "client_errors": [],
  "result": "PASS"
}
```

---

## 7. Probe 设计（稳定性判定核心）

### 7.1 Hub Probe

```text
GET /api/v1/agents  (Authorization: Bearer STREAM_TOKEN)
每 2s：记录 deviceId 是否存在、sessionCount、lastPing 增量
```

### 7.2 CAE Probe（有 adb 时）

```bash
# 进程
pidof / ps | com.nexartc.cae.Server

# liveness
cat /data/local/tmp/cae/run/liveness.json

# abort 签名（增量）
grep -E 'Pure virtual|Track is not open|terminating with uncaught' \
  /data/local/tmp/cae/run/cae_stdout.log

# 可选：DisconnectClient 同步 close
grep 'sync close under pause' $(ls -t cae_server_*.log | head -1)
```

无 adb 时：仅 Hub + 客户端 ICE/stats；降低门禁强度（标注 `probe=hub_only`）。

### 7.3 与 IncidentLogArchiver 联动

- L3 默认 **不应** 产生 incident（无重启）。  
- 若 `incidents/` 数量在压测中增加 → **判 FAIL**（说明 Worker 死过并被 Supervisor 拉起）。  
- 负向用例 F10 单独允许增加。

---

## 8. 执行计划与门禁

### 8.1 日常开发

```bash
# 功能冒烟（~3–5 min）
node test/session_stress/run_functional.mjs

# 并发剧本（~10–15 min）
node test/session_stress/run_concurrent.mjs --only=C02,C03,C05
```

### 8.2 夜间 / 发版前

```bash
STRESS_CLIENTS=3 STRESS_ROUNDS=500 PATTERN=pair_flap \
  node test/session_stress/run_stress.mjs
```

### 8.3 门禁矩阵

| 阶段 | 必跑 | 建议 |
|------|------|------|
| PR | F01 F02 F05；C02 缩短版（30s） | — |
| merge 前 | C01–C05 | C06–C07 |
| 夜间 | S01 500 rounds × 3 clients | Puppeteer 单路 10min |
| 复现 native abort | C++ cae-receiver × pair_flap | — |

---

## 9. 分阶段落地

### Phase 0（本文）— 设计 ✅

### Phase 1 — Node 编排骨架（优先）

1. `hub_client.mjs` + `webrtc_client.mjs`（Mode A join + PC）。  
2. `cae_probe.mjs` / `hub_probe.mjs`。  
3. 落地 F01/F02/F05 + C02/C03/C05。  
4. `run_stress.mjs`（`pair_flap`）。  
5. 接入 `test/turn/run_all_tests.sh` 可选 target：`session_stress`。

### Phase 2 — 强化

1. 三用户 C04/C09；随机种子可复现（`--seed=`）。  
2. 报告上传 / junit。  
3. Puppeteer 包装 1 路真浏览器对照。  
4. C++ `cae-receiver` Hub 模式 + `stress_teardown.sh`。

### Phase 3 — 产品化

1. admin / CI nightly job。  
2. 失败自动附带 `incidents/` artifact。  
3. 与码率/场景切换用例矩阵合并（可选）。

---

## 10. 用例与代码映射（实现时填写）

| 用例 | 脚本 | 主要断言 |
|------|------|----------|
| F01–F06 | `scenarios/F*.mjs` | ICE / Hub / 无 abort |
| C02 | `scenarios/C02_one_online_one_flap.mjs` | A 视频 + B flap + pid |
| C03 | `scenarios/C03_all_offline_together.mjs` | 同步下线无 abort |
| S01 | `run_stress.mjs` | rounds / signatures |

---

## 11. 风险与限制

1. **家庭 NAT / host ICE**：压测机须能达 CAE `UDP 50000` 或改 `ICE_MODE=hybrid`。  
2. **单设备算力**：3 路编码可能掉帧；稳定性用例关注 **不崩**，不强制满帧。  
3. **werift 非 Chrome**：功能上等价信令/会话；编解码差异不作为「画质」门禁。  
4. **Supervisor cooldown 45s**：压测中若误杀 Worker，恢复窗口要计入超时。  
5. **勿在压测中 force-stop 整包**：会带走 Supervisor，超出本套件范围。

---

## 12. 变更记录

| 日期 | 说明 |
|------|------|
| 2026-07-24 | 首版：针对会话 teardown crash 的功能/并发/压测方案；推荐 Node.js 主路径 + C++ 专项；目录与门禁矩阵 |

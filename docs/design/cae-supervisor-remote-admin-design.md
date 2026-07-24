# CAE 监控进程（CEA）与远程运维设计

> 目标：当 CAE 异常、卡死、掉线或被异常杀死时，由一个常驻的监控进程接管本地控制面，先清理旧 CAE，再拉起新 CAE，并通过 `www.signalling-nexartc.cn/admin.html` 继续接收远程配置与重启指令。  
> CAE crash / 稳定性日志路径见 [`nexartc-logging-design.md`](./nexartc-logging-design.md)（`cae_crash.log`、`[EXC] CRASH`、`[STAB] heartbeat`）。  
> **异常重启前必须先做日志备份**：见本文 **§11 Incident Snapshot**（与 logging-design §2.6 对齐）。  
> **多用户会话 teardown / 撕连压测**：见 [`nexartc-cae-session-teardown-stress-test-design.md`](./nexartc-cae-session-teardown-stress-test-design.md)。

## 1. 设计结论

可以做，而且**推荐做成 CAE 之外的独立进程**，不要把监控逻辑放在 CAE 同进程内。

原因很直接：
- CAE 真异常时，同进程里的 watchdog 也会一起死。
- `AbstractCaeServerService` 当前返回 `START_NOT_STICKY`，系统不会帮你自动拉起。
- Hub / admin.html 是公网可达的控制面，最适合做远程运维入口。

## 2. 组件拆分

### 2.1 角色定义

- **CAE Worker**：现有媒体引擎进程，负责采集、编码、信令、WebRTC。
- **CEA Supervisor**：新增常驻监控进程，负责健康检查、拉起/杀死 CAE、回报状态、接收远程指令。
- **Signal Hub**：`www.signalling-nexartc.cn` 上的 `admin.html` + `/api/ws` 管理通道。

### 2.2 进程关系

```text
admin.html -> Hub admin-ws -> CEA Supervisor -> CAE Worker
                              ^                |
                              |                v
                           状态/心跳      公网 IP / 运行态
```

### 2.3 关键约束

- **只走出站连接**：CEA Supervisor 通过 WSS 主动连 Hub，避免给家庭网络加入站端口。
- **监控进程必须独立存活**：建议独立 Android service / root daemon / app_process。
- **CAE 异常先清理再重启**：避免残留 socket、锁文件、端口占用。
- **控制面与媒体面解耦**：admin.html 只能管控，不直接碰媒体流。

## 3. 生命周期

### 3.1 启动

1. CEA Supervisor 自启动（开机、崩溃恢复或手工启动）。
2. Supervisor 读取本地配置与设备标识。
3. Supervisor 向 Hub 注册，建立长期心跳。
4. Supervisor 拉起 CAE Worker。
5. CAE Worker 再向 Hub /agent 注册并进入正常服务态。

### 3.2 异常自愈

触发条件建议包括：
- CAE 进程退出或僵死。
- CAE 心跳超时。
- 本地端口/Socket 不可用。
- Hub 侧检测到 agent heartbeat 超时。

动作建议：
1. Supervisor 标记当前 CAE 为 `DEGRADED`。
2. **先归档事故快照**（§11 Incident Snapshot：轮转日志尾、crash、stdout、liveness、meta）。
3. 先优雅停止 CAE，超时后强杀。
4. 清理旧 socket、临时文件、残留子进程（**不得**清掉 `logs/incidents/`）。
5. 重新启动 CAE Worker。
6. 恢复向 Hub 的注册与心跳。

### 3.3 远程重启

admin.html 触发的“重启”不直接等于 kill 整机，而是：
- `restart_cae`：只重启 CAE Worker。
- `restart_supervisor`：重启 CEA Supervisor，自带重连与拉起 CAE。
- `stop_cae`：停 CAE，但保留 Supervisor 在线。
- `stop_supervisor`：停 Supervisor 与 CAE。

## 4. 与 admin.html 的通信

建议新增一个独立的 Hub 管理通道，例如：
- `wss://www.signalling-nexartc.cn/api/ws/supervisor`

协议建议沿用现有 JSON 风格：

### 4.1 Supervisor -> Hub

```json
{
  "type": "register_supervisor",
  "deviceId": "device-mi9-001",
  "caePid": 1234,
  "version": "24.12.0",
  "capabilities": ["restart_cae", "query_public_ip"]
}
```

```json
{
  "type": "status",
  "deviceId": "device-mi9-001",
  "caeState": "RUNNING",
  "publicIp": "120.79.21.28",
  "publicPort": 50000,
  "lastExitReason": "watchdog_timeout"
}
```

### 4.2 Hub -> Supervisor

```json
{ "type": "start_cae", "requestId": "..." }
{ "type": "restart_cae", "requestId": "..." }
{ "type": "stop_cae", "requestId": "..." }
{ "type": "restart_supervisor", "requestId": "..." }
{ "type": "query_public_ip", "requestId": "..." }
```

Hub 侧 `admin.html` 继续只操作现有 `admin-ws.ts`，由 Hub 转发到对应 Supervisor。

## 5. 公网 IP 获取

### 5.1 推荐方式

CEA Supervisor 直接复用 CAE 已有的 STUN / TURN 探测逻辑，或单独实现一个轻量 `PublicIpResolver`：
- 向 coturn 做 STUN Binding。
- 读取 reflexive address 作为 CAE 公网 IP。
- 结果短缓存，网络切换或 CAE 重启后刷新。

### 5.2 上报内容

建议至少上报：
- `publicIp`
- `publicPort`（若固定映射端口存在）
- `resolver`（stun / coturn / fallback）
- `resolvedAt`

### 5.3 失败策略

- 解析失败不阻断 CAE 重启。
- 解析结果为空时，Hub 端显示 `unknown`。
- 若端口映射不稳定，不要强行发布 host candidate。

## 6. 安全设计

- Supervisor 注册必须带独立 token，不要复用浏览器 `STREAM_TOKEN`。
- Hub 管理密码仍然只用于 `admin.html` 登录。
- 所有远控命令都要带 `requestId`，并返回 `ack/ok/error`。
- 对 `kill_cae` / `restart_supervisor` 做权限隔离与审计日志。

## 7. 落地建议

### 7.1 最小可行版本

先做三件事：
1. Supervisor 常驻 + 心跳。
2. CAE 进程健康检查 + 自动重启。
3. Hub admin.html 远程下发 `restart_cae` / `query_public_ip`。

### 7.2 后续增强

- 设备列表与在线状态展示。
- CAE 异常告警推送。
- 公网 IP / NAT 类型 / 最近一次重启原因可视化。
- 与现有 `/api/v1/agents`、`/api/v1/health` 做统一面板。

## 8. 结论

这个方案是可行的，且比“让 CAE 自己负责自救”更稳：
- CAE 负责业务。
- CEA Supervisor 负责活着和重启。
- Hub / admin.html 负责远程编排。

这样即使 CAE 崩了，远程运维链路仍然在，管理员还能继续启动、重启、取公网 IP。

## 9. 当前落地骨架

截至 2026-07-21，已先落命令面和 Android 骨架：
- `nexartc-cloudPhoneAccess-web/server/src/adb.ts`：新增 `start_supervisor` / `stop_supervisor` / `restart_supervisor` / `query_public_ip` 的 ADB 命令封装。
- `nexartc-cloudPhoneAccess-web/server/src/admin-ws.ts`：把上述 Supervisor 动作接入 `admin.html` 的 WebSocket 命令面。
- `nexartc-cloud-phone-access-engine/app/src/shared/java/com/nexartc/cae/supervisor/CaeSupervisorRuntime.java`：常驻监控骨架，负责拉起/停止 CAE。
- `nexartc-cloud-phone-access-engine/app/src/shared/java/com/nexartc/cae/service/CaeSupervisorServiceBase.java`：Supervisor Service 基类。
- `nexartc-cloud-phone-access-engine/app/src/main/java/com/nexartc/cae/ui/CaeMainActivity.java`：常规启动/停止 CAE 时同步起停 Supervisor。

限制（已被 §10 补强替代，保留作历史）：
- `query_public_ip` 目前仍返回 `unknown`，真实 STUN / coturn 探测还没接到 Android 侧 IPC。
- 监控骨架先沿用现有 CAE 运行态与服务启停路径，后续可再升级为更强的独立守护进程。

## 10. 补强架构（对抗 CAE 长期不可用）

> 对照实网事故：Hub TCP 闪断、Agent 半开假在线、Worker 僵死/退出后无人拉起、`webrtc_local_ip` DHCP 漂移、域名信令不可达。  
> **目标不是“永不掉线”，而是：掉线可自愈、远程可拉起、控制面与媒体面命运解耦。**

### 10.1 进程解耦

| 组件 | 进程 | 生命周期 |
|------|------|----------|
| CAE Worker | root: `app_process … com.nexartc.cae.Server`；nonroot: 主进程内引擎线程 | 可杀可重启 |
| CEA Supervisor | 独立 Android 进程 `com.nexartc.cloudapp:cae_supervisor`（`START_STICKY` + 开机自启） | 与 Worker 不同进程，Worker 崩不影响 Supervisor |
| Hub | VPS | 提供 agent 在线态 + Supervisor 待办命令 |

约束：
- Supervisor **不得**与 Worker 共用 `AbstractCaeServerService` 的静态 `sLastKnownRunning`。
- `am force-stop` / 整包卸载仍会带走同 UID 进程——这属于环境级故障，不在自愈承诺内。
- root 设备：`BOOT_COMPLETED` → 拉起 Supervisor → Supervisor 再拉起 Worker。

### 10.2 CAE → Supervisor 三层探活（零周期 su）

> 背景（2026-07-24）：Supervisor 若每 5s `su -c test -d /proc/$pid` 探 root Worker，Magisk 会刷 Toast「CAE已被授予超级用户权限」。周期探活**禁止**走 su；`su` 仅用于启停/pkill 等低频生命周期。

路径：`/data/local/tmp/cae/run/liveness.json`（Worker 写，mode 允许 app UID 读/删以便 Launcher invalidate）。

#### L1 — 应用层心跳（主探活）

Worker（`CaeLiveness`）每 **5s** 写一次（墙钟毫秒）：

```json
{
  "ts_ms": 1784637698372,
  "pid": 17076,
  "device_id": "device-mi9-001",
  "agent_registered": true,
  "local_ip": "192.168.124.108",
  "listen_port_app": 7000,
  "webrtc_port": 50000
}
```

Supervisor（`:cae_supervisor`）**只信文件年龄**（默认 `supervisor_liveness_timeout_sec=30`）：
- `present && now - ts_ms ≤ timeout` → 健康。
- 文件缺失或超时 → `restartCae(liveness_stale|worker_not_running)`。
- **不**再周期读 `/proc` / `su ps` 判 pid。

僵死覆盖：进程仍在但心跳停写 → 超时后同样重启。

#### L2 — Launcher Process 句柄（秒级死亡感知）

`CaeServerService`（主进程）经 Magisk `su` 拉起 Worker，持有 `Process` 句柄；`AbstractCaeServerService` 健康检查用 `Process.isAlive()`（**无 su**）。

管道/Worker 退出时：Launcher **立即 invalidate** `liveness.json`（unlink 或写 `ts_ms=0` tombstone），避免 Supervisor 在超时窗口内误判仍健康；再走 in-service relaunch。

说明：`Process` 观察的是 `su -c ...` 管道进程，与当前 `exec` 进 Worker 的生命周期绑定；若未来 daemonize 脱钩须同步改 L2。

#### L3 — 旁证（保留）

TCP `listen_port_app`、WebRTC UDP（advisory）、Hub agent poll / pending、`agent_unregistered_too_long`、wlan IP 变化——抓住心跳仍写但业务已挂或 Agent 掉线。

#### `su` 策略

| 操作 | 周期探活 | 启停/pkill/删 liveness（故障时） | Magisk launch |
|------|----------|----------------------------------|---------------|
| 允许 | 否 | 是 | 是（仅拉起） |

运维双保险：Magisk 对该应用关闭 Toast / 静默授权（非代码唯一手段）。

### 10.3 Hub → Supervisor（agent offline）

双通道（实现以 HTTP 为主，避免再引入 WS 客户端依赖）：

1. **推送入队**：Hub `sweepStaleAgents` 超时踢掉 agent 时，入队  
   `{ type: "restart_cae", deviceId, reason: "agent_heartbeat_timeout" }`。
2. **Supervisor 拉取**：`GET /api/v1/supervisor/pending?deviceId=&token=` → 领取后 `POST .../ack`。
3. **并行探测**：Supervisor 每 ~20s 轮询 `GET /api/v1/agents`；若本地 liveness 称已注册但 Hub 连续两次不见该 device → `restart_cae(hub_agent_offline)`。

### 10.4 信令多 endpoint（IP 优先）

配置：

```ini
[signal]
signal_agent_url=wss://120.79.21.28/agent
signal_agent_urls=wss://120.79.21.28/agent,wss://www.signalling-nexartc.cn/agent
signal_disable_tls_verify=1
```

规则：
- `GetSignalAgentUrls()` = `signal_agent_urls` CSV ∪ `signal_agent_url`（去重，**IP 字面量 URL 排前**）。
- `CaeSignalAgent` 单次连接失败后切换下一个 URL，再指数退避。
- Supervisor Hub HTTP base 由第一个 `wss://` 推导为 `https://host`，可用 `supervisor_hub_http_base` 覆盖。

### 10.5 扩展健康项

| 检查项 | 动作 |
|--------|------|
| liveness 超时 / 缺失（L1）或 Launcher Process 退出（L2） | kill + start CAE |
| `listen_port_app` / WebRTC UDP 端口无人监听且进程在 | restart |
| `wlan0` IPv4 变化 | 清空无效 `webrtc_local_ip` 覆盖；必要时 restart（host ICE） |
| Hub agent 离线（pending 或 poll） | restart_cae |
| `agent_registered=false` 持续过久且信号 URL 已配置 | restart_cae |

### 10.6 明确不能彻底消灭的场景

断网、关机、`force-stop`、su/Magisk 失效、Hub 整机不可达、运营商拦截全部 endpoint——只能等条件恢复后再自愈。

### 10.6.1 MIUI / 自启动拒绝（实网 2026-07-24）

日志特征：

```text
lowmemorykiller: Kill 'com.nexartc.cloudapp:cae_supervisor' ... low on memory
AutoStartManagerService: MIUILOG- Reject RestartService packageName :com.nexartc.cloudapp
```

`START_STICKY` 被 MIUI 自启动管理拦住后，看门狗无法复活。缓解：

1. 设备上开启「自启动 / 无限制电池 / 锁定任务」。
2. 代码侧：`CaeSupervisorWatchdogReceiver`（AlarmManager ~60s）二次拉起 Supervisor。
3. **UI 不得在 `MainActivity.onDestroy` 停 Supervisor**（曾导致打开 App 再退出后永久失守）。
4. 用户点「停止」只停 Worker，并写 `user_stopped` 旗标；Supervisor 继续存活但不自动 relaunch。

### 10.7 落地清单（本轮实施）

- [x] 设计补强（本节）
- [x] Supervisor 独立进程 + `START_STICKY` + BootReceiver（root）
- [x] Worker liveness 写入 + Supervisor ≤30s 看门狗
- [x] Hub pending 命令 + Supervisor 轮询 `/api/v1/agents` / pending
- [x] `signal_agent_urls` 多 endpoint 回退
- [x] wlan0 / 端口 / agent 注册态健康检查
- [x] **异常重启前日志备份（§11）** — `IncidentLogArchiver` 已落地（2026-07-24）

---

## 11. 异常重启 / 拉起前日志备份（Incident Snapshot）

> 背景：实网中 `Pure virtual` / `Track is not open` 等 abort 后，Supervisor 会杀进程并拉起新 Worker。  
> 轮转日志与 `cae_stdout.log` 会被新会话覆盖；若未在 **kill 之前** 冻结现场，根因证据丢失。  
> 路径约定与排障入口见 [`nexartc-logging-design.md`](./nexartc-logging-design.md) §2.5 / §2.6。

### 11.1 目标

1. **先备份、后清理/拉起**：任何会毁掉现场的重启动作，必须先落盘一份不可变快照。
2. **可归因**：每份快照带 `reason`、pid、liveness、时间戳，可与 Hub 掉线时间对齐。
3. **可拉取**：`adb pull` 单目录即可取走最近 N 次事故，无需猜轮转文件。
4. **有界**：磁盘占用可控（个数 + 总字节上限），避免把手机打满。

### 11.2 触发与不触发

| 触发点 | 调用方 | 说明 |
|--------|--------|------|
| `restartCae(reason)` | `CaeSupervisorRuntime` | **主路径**：liveness_stale / worker_process_missing / hub_agent_offline / listen_port_* / wlan0_ip_changed / hub_pending / agent_unregistered_too_long |
| `AbstractCaeServerService` health check 发现引擎已死并准备 `stopEngine`+`startEngine` | `CaeServerService` | 进程级立即重启；同样要先归档 |
| admin / Hub `restart_cae` | 同上 `restartCae` | 远程重启也归档（便于对照“人为 vs 自愈”） |

| 不触发 | 原因 |
|--------|------|
| 冷启动且无旧 Worker / 无旧日志 | 无现场可保 |
| `Skip restart (cooldown)` | 未执行 kill |
| 仅 `ensureCaeStarted` 且 liveness 健康 skip | 未破坏现场 |
| 配置热读（`reloadConfigLight`） | 无进程动作 |

**时序硬约束（必须遵守）：**

```text
detect unhealthy
  → archiveIncident(reason)     # 同步、有超时上限（建议 ≤3s）
  → invalidate liveness + pkill / stopService
  → ensureCaeStarted / startEngine
```

禁止：先 `pkill` 再归档（stdout / 打开的 fd 可能已空；竞态下会拷到半截新进程日志）。

### 11.3 目录与命名

根目录（与现有日志同树，便于权限与 pull）：

```text
/data/local/tmp/cae/logs/incidents/
```

单次事故目录名：

```text
YYYYMMDD_HHMMSS_<reason_slug>_<pid>/
```

- `reason_slug`：把 `reason` 中非 `[A-Za-z0-9._-]` 换成 `_`，截断至 48 字符。  
  例：`liveness_stale_ageMs=45000` → `liveness_stale_ageMs_45000`
- `pid`：归档时刻 `liveness.json` 中的 pid；缺失则用 `0`。
- 墙钟用设备本地时区（与 CAE 文件日志时间一致）。

示例：

```text
/data/local/tmp/cae/logs/incidents/20260724_085657_Track_is_not_open_4683/
/data/local/tmp/cae/logs/incidents/20260724_090147_listen_port_app_missing_7000_0/
```

### 11.4 单次快照内容

| 文件 | 来源 | 必选 | 说明 |
|------|------|------|------|
| `meta.json` | Supervisor 生成 | 是 | 见下方 schema |
| `liveness.json` | `/data/local/tmp/cae/run/liveness.json` | 尽量 | 僵死判定依据；缺失则写 `liveness.missing` |
| `cae_crash.log` | `…/logs/cae_crash.log`（及兼容 `cae_signal.log`） | 尽量 | native abort / SIG* |
| `cae_stdout.log` | `…/run/cae_stdout.log` | 尽量 | RootProcessSupervisor 捕获的 stdout/stderr 尾（含 `Pure virtual`） |
| `cae_server_latest.log` | 当前轮转文件整份 copy | 是* | `ls -t cae_server_*.log \| head -1` |
| `cae_server_prev.log` | 次新轮转文件（若存在） | 否 | 覆盖跨轮转边界的尾部 |
| `logcat_cae_tail.txt` | `logcat -d -s CAE RootProcessSupervisor CaeSupervisor CaeRootService` 最近约 400 行 | 否 | 文件日志未 flush 时的补强 |
| `reason.txt` | 纯文本一行 `reason` | 是 | 便于 `ls`/`grep` 不解析 JSON |

\* 若主轮转文件缺失，至少保证 `meta.json` + 已存在的 crash/stdout。

`meta.json` schema：

```json
{
  "schema": 1,
  "archived_at_ms": 1784854607933,
  "archived_at_local": "2026-07-24T08:56:57+08:00",
  "reason": "liveness_stale ageMs=45210",
  "reason_slug": "liveness_stale_ageMs_45210",
  "trigger": "supervisor_restart_cae",
  "device_id": "device-mi9-001",
  "worker_pid": 4683,
  "supervisor_pid": 9140,
  "liveness_age_ms": 45210,
  "agent_registered_local": true,
  "hub_expect_agent": true,
  "apk_version_name": "…",
  "libcae_native_mtime": "…",
  "files_copied": ["liveness.json", "cae_crash.log", "cae_stdout.log", "cae_server_latest.log"],
  "archive_elapsed_ms": 420,
  "notes": ""
}
```

### 11.5 实现要点

**推荐落点（Java，Supervisor 进程）：**

- 新建 `com.nexartc.cae.supervisor.IncidentLogArchiver`
- `CaeSupervisorRuntime.restartCae()` 在 `pkill` **之前**调用 `IncidentLogArchiver.archive(reason, trigger)`
- `AbstractCaeServerService` health-restart 路径同样调用（`trigger=server_service_health`）
- root 场景：归档命令通过现有 `runSu(...)` 执行，保证可读 `/data/local/tmp/cae/**`

**归档命令（逻辑伪代码，可用 shell 内聚）：**

```sh
INC_ROOT=/data/local/tmp/cae/logs/incidents
LOG_DIR=/data/local/tmp/cae/logs
RUN_DIR=/data/local/tmp/cae/run
# mkdir INC_ROOT/<name>
# cp liveness / crash / stdout
# cp "$(ls -t $LOG_DIR/cae_server_*.log | head -1)" cae_server_latest.log
# optional: second latest → cae_server_prev.log
# logcat -d … > logcat_cae_tail.txt
# write meta.json + reason.txt
# prune old incidents
```

约束：
- **同步调用**，整体 **硬超时 ≤3s**（超时则带着已拷文件继续重启，并在 `meta.notes` 记 `archive_timeout`）。
- **禁止**在归档路径上做大范围 `tar` 全盘 logs（耗时长、易拖死自愈）。
- 拷贝用 `cp`/`cat`；不要 `mv` 现行轮转文件（避免打断仍存活的僵死进程写盘）。
- 归档成功打一行 Supervisor 日志：  
  `[STAB] incident archived path=… reason=…`  
  失败：`[EXC] incident archive failed …`（不阻断重启）。

### 11.6 保留策略

配置（写入 `CaeConfig.ini`，Supervisor `IniView` 读取）：

```ini
# 异常重启前日志备份
supervisor_incident_keep_count=10
supervisor_incident_max_total_mb=64
supervisor_incident_enabled=1
```

剪枝规则（每次归档成功后执行）：
1. 按目录名时间序，超过 `keep_count` 的最旧目录删除。
2. 若总大小仍超过 `max_total_mb`，继续删最旧直至满足。
3. 正在写入的目录不删；剪枝失败只打日志。

### 11.7 与现有日志通道的关系

| 通道 | 日常 | 事故后 |
|------|------|--------|
| `cae_server_*.log` | 环形轮转，可被新会话覆盖 | 事故前副本在 `incidents/…/cae_server_latest.log` |
| `cae_crash.log` | 追加；可能被多次 crash 混写 | 事故时刻整文件副本 |
| `cae_stdout.log` | 进程级 stdout 尾 | 事故时刻副本（捕捉 `Pure virtual`） |
| `liveness.json` | 被新 Worker 覆写 | 事故时刻副本 |
| logcat | 环缓易丢 | 可选 tail 补强，不作为唯一真源 |

Native crash handler（`[EXC] CRASH` → `cae_crash.log`）**仍然保留**；Incident Snapshot 是其上层“重启前冻结”，两者互补而非替代。

### 11.8 分析流程

```bash
SERIAL=<adb_serial>
adb -s "$SERIAL" shell "su -c 'ls -lt /data/local/tmp/cae/logs/incidents | head -20'"
adb -s "$SERIAL" pull /data/local/tmp/cae/logs/incidents/ ./cae_incidents/

# 单次事故
DIR=./cae_incidents/<name>
cat "$DIR/meta.json"
grep -E '\[EXC\]|\[STAB\]|Pure virtual|Track is not|FATAL|SIG' \
  "$DIR/cae_server_latest.log" "$DIR/cae_crash.log" "$DIR/cae_stdout.log" 2>/dev/null | tail -80
```

对齐 Hub：用 `meta.archived_at_local` / `archived_at_ms` 去 `serve_https_*.log` 搜前后 2 分钟的 `agent unregister` / `DEVICE_OFFLINE`。

### 11.9 落地清单（本节）

- [x] `IncidentLogArchiver` + `restartCae` / Service health 挂钩
- [x] `meta.json` + 目录命名 + 剪枝
- [x] ini 开关与配额（`supervisor_incident_*`）
- [x] 文档与 install 速查命令（logging-design / install-guide）
- [x] 实机验证：Supervisor `restartCae` 前产生 `logs/incidents/<stamp>_*/`（含 `meta.json` / 轮转日志 / crash / liveness）

实现落点：
- `nexartc-cloud-phone-access-engine/.../supervisor/IncidentLogArchiver.java`
- `CaeSupervisorRuntime.restartCae()` → archive → kill → start
- `AbstractCaeServerService` health check → archive → stop/startEngine


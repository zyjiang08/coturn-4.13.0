# CAE 监控进程（CEA）与远程运维设计

> 目标：当 CAE 异常、卡死、掉线或被异常杀死时，由一个常驻的监控进程接管本地控制面，先清理旧 CAE，再拉起新 CAE，并通过 `www.signalling-nexartc.cn/admin.html` 继续接收远程配置与重启指令。  
> CAE crash / 稳定性日志路径见 [`nexartc-logging-design.md`](./nexartc-logging-design.md)（`cae_crash.log`、`[EXC] CRASH`、`[STAB] heartbeat`）。

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
2. 先优雅停止 CAE，超时后强杀。
3. 清理旧 socket、临时文件、残留子进程。
4. 重新启动 CAE Worker。
5. 恢复向 Hub 的注册与心跳。

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

### 10.2 CAE → Supervisor 应用层 liveness（≤30s）

路径：`/data/local/tmp/cae/run/liveness.json`（root 可写，跨进程可读）。

Worker 每 **5s** 写一次（墙钟毫秒）：

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

Supervisor 判定：
- `now - ts_ms > 30s` 或文件缺失且本应在跑 → **先杀后启** Worker。
- 仅 PID 存活但 liveness 停滞 = 僵死，同样重启。

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
| liveness 超时 / Worker 进程消失 | kill + start CAE |
| `listen_port_app` / WebRTC UDP 端口无人监听且进程在 | restart |
| `wlan0` IPv4 变化 | 清空无效 `webrtc_local_ip` 覆盖；必要时 restart（host ICE） |
| Hub agent 离线（pending 或 poll） | restart_cae |
| `agent_registered=false` 持续过久且信号 URL 已配置 | restart_cae |

### 10.6 明确不能彻底消灭的场景

断网、关机、`force-stop`、su/Magisk 失效、Hub 整机不可达、运营商拦截全部 endpoint——只能等条件恢复后再自愈。

### 10.7 落地清单（本轮实施）

- [x] 设计补强（本节）
- [x] Supervisor 独立进程 + `START_STICKY` + BootReceiver（root）
- [x] Worker liveness 写入 + Supervisor ≤30s 看门狗
- [x] Hub pending 命令 + Supervisor 轮询 `/api/v1/agents` / pending
- [x] `signal_agent_urls` 多 endpoint 回退
- [x] wlan0 / 端口 / agent 注册态健康检查


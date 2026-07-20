# nexartc Mode A — VPS 部署与验证记录

> 日期：2026-07-12  
> 依据：[`nexartc-turn-mode-a-implementation.md`](./nexartc-turn-mode-a-implementation.md) §0 / Phase 0b / P0  
> **完整安装步骤（编译 / 产物 / 配置 / 测试 URL / 日志）：** [`nexartc-install-deploy-guide.md`](./nexartc-install-deploy-guide.md)  
> 目标域名：`www.signalling-nexartc.cn`（A → `120.79.21.28`）  
> 云手机：家庭内网 MI 9（adb `9898d727`），**未**迁到 VPS

---

## 1. 结论摘要

| 检查项 | 结果 |
|--------|------|
| VPS 上 coturn `use-auth-secret` + 公网 Allocate | ✅ `turnutils_uclient` 公网 10/10 中继 |
| Hub `GET /api/v1/health` | ✅ |
| Hub `GET /api/v1/turn-credentials`（Bearer） | ✅ 无/错 token → 401；正确签发 iceServers |
| Hub 托管 `device/` 静态页 | ✅ HTTP 200，`WebRTC Cloud Phone` |
| 浏览器 Trickle ICE `typ relay`（强制 relay） | ✅ 收集到 3 个 relay 候选（relay IP=`120.79.21.28`） |
| 域名 HTTPS `https://www.signalling-nexartc.cn/` | ✅ 服务端正常（VPS 本机 + 用户侧可访问）；证书 SAN 匹配 |
| 域名 HTTPS（部分外网 ISP） | ⚠️ 个别网络仍可能在 TLS 握手阶段 RST（与运营商/备案策略有关）；可用 IP 或 `:9443` 兜底 |
| CAE 真机写入 TURN 方案 B 配置并启动 | ✅ APK/root 路径；`webrtc_ice_mode=hybrid` |
| Phase 3 `/agent` + `/ws` 房间路由 | ✅ `test/turn/test_hub_phase3.mjs` / `deploy_vps.sh` |
| hybrid ICE + relay 出画（MI 9 联调） | ⚠️ 可 connected；relay 路径 **高丢包/卡顿**（见实现文档 §16） |
| p2p-only 对照（`?ice_mode=p2p`） | ✅ 同公网 srflx **~19s ICE failed** → 当前网络 **必须 TURN** |

**P0（Hub + coturn + device 迁 VPS）已通过。**

**正式入口（推荐）：**

- 页面（当前 VPS 测试 token 见 §8.4）：
  `https://120.79.21.28/device/?turn_token=<STREAM_TOKEN>&device_id=device-mi9-001`
- ICE 模式：`&ice_mode=hybrid`（默认，host → p2p → relay）| `&ice_mode=relay` | `&ice_mode=p2p`（仅诊断）
- 调试强制 relay：在 URL 末尾加 `&force_relay=1` 或 `&ice_mode=relay`；日常优先验证 host 端口映射，再看 p2p
- Health：`https://120.79.21.28/api/v1/health`
- Agent 列表：`GET /api/v1/agents`（Bearer `STREAM_TOKEN`）

**device 页请优先用 IP 加载**（bundle 最新）；`www.signalling-nexartc.cn/device/` 在部分 curl/缓存场景下可能超时或旧 JS。Hub WSS 可用域名或 IP。

**兜底（个别网络域名 HTTPS 不通时）：**

- `https://120.79.21.28/device/` 或 `https://120.79.21.28:9443/device/`（浏览器可能提示证书与 IP 不匹配；`turn_token` 参数同样有效）

TURN/STUN UDP 走域名 `www.signalling-nexartc.cn:3478` **不受** HTTPS SNI 拦截影响（已实测）。

密钥用途与获取方式见 **§8**。

---

## 2. 部署拓扑（本次实测）

```
公网 VPS 120.79.21.28 (Aliyun ECS, Ubuntu 20.04)
├─ nexartc-hub.service     Node 20  → :443 + :9443  (TLS = signalling-nexartc.cn 证书)
├─ nexartc-coturn.service  apt coturn 4.5.1.1
│     UDP/TCP 3478, TLS 5349
│     external-ip=120.79.21.28/172.18.141.122
│     use-auth-secret + realm=nexartc.com
└─ 静态页 /opt/nexartc/hub/device/

家庭内网 MI 9 (CAE)
└─ webrtc_ice_mode=hybrid
   webrtc_turn_host=120.79.21.28
   webrtc_turn_secret=<与 Hub/coturn 相同>
   signal_agent_url=wss://120.79.21.28/agent
   device_id=device-mi9-001
   agent_token=<AGENT_TOKEN>
   signal_disable_tls_verify=1
```

系统单元与示例配置在仓库：`test/turn/vps/`。一键脚本：`test/turn/deploy_vps.sh`。

---

## 3. 安装与部署步骤（可复现）

### 3.1 开发机准备

```bash
# 1) SSH（注意本机若有 ssh 函数包装，请用 /usr/bin/ssh）
/usr/bin/ssh -i ~/.ssh/id_ed25519 root@signalling-nexartc.cn 'hostname'

# 2) 构建 Hub + device
cd nexartc-cloudPhoneAccess-web && ./build_all.sh

# 3) 确认域名证书
ls cert/unpacked/Nginx/fullchain.crt cert/unpacked/Nginx/private.key
# SAN 须含 signalling-nexartc.cn / www.signalling-nexartc.cn
```

### 3.2 一键部署到 VPS

```bash
./test/turn/deploy_vps.sh
# 仅复测：
./test/turn/deploy_vps.sh --verify-only
```

脚本会：

1. 生成（或复用）`/tmp/nexartc-vps-deploy/env` 中的 `TURN_SECRET` / `STREAM_TOKEN`（**勿提交 Git**）
2. 安装 Node 20 二进制到 `/usr/local`（系统 apt 自带 Node 10 过旧）
3. 同步 `/opt/nexartc/hub/{device,server}`、证书、`/etc/nexartc/turnserver.conf`
4. 启用 `nexartc-hub.service` + `nexartc-coturn.service`（并 disable 旧 `coturn.service`）
5. 跑 P0 验收（health / credentials / uclient）

手工等价路径见下方 §3.3。

### 3.3 手工部署要点

**环境文件** `/etc/nexartc/hub.env`（chmod 600）：

```bash
TURN_SECRET=...
STREAM_TOKEN=...
TURN_HOST=www.signalling-nexartc.cn
TURN_PORT=3478
TURNS_PORT=5349
ENABLE_TURNS=1
HUB_PORTS=443:cae-server,9443:cae-server
WEB_ROOT=/opt/nexartc/hub
LOG_DIR=/opt/nexartc/hub/server/logs
```

**证书**：

```bash
# Hub basename cae-server
/opt/nexartc/hub/server/certs/cae-server.{crt,key}
# coturn TLS 5349
/etc/nexartc/certs/turn.{crt,key}
```

**coturn**（`test/turn/vps/turnserver.conf`）：

- `use-auth-secret`；secret 由 systemd `EnvironmentFile` 注入 `--static-auth-secret`
- `external-ip=120.79.21.28/172.18.141.122`（阿里云 NAT 必需）
- **不要**开 `--allow-loopback-peers`
- Ubuntu 20.04 自带 coturn **无** `--no-daemon`；前台运行时省略 `--daemon` 即可

**防火墙 / 安全组**（阿里云控制台）至少放行：

| 端口 | 协议 | 用途 |
|------|------|------|
| 443 / 9443 | TCP | Hub HTTPS/WSS |
| 3478 | UDP+TCP | STUN/TURN |
| 5349 | TCP | TURNS |
| 49152–65535 | UDP | TURN relay |

本机 `iptables` 为 ACCEPT；连通性主要取决于云安全组。

### 3.4 云手机 CAE（家庭内网）

```bash
# 部署含 CaeTurnCredentials 的产物
cd nexartc-cloud-phone-access-engine/output/server
./deploy.sh <serial>
adb -s <serial> install -r CaeServer-root-debug.apk

# 写入 [webrtc]（host 优先，secret 与 VPS TURN_SECRET 一致）
webrtc_ice_mode=hybrid
webrtc_local_ip=192.168.124.103
webrtc_public_ip=                # 运行时通过 coturn/STUN 发现公网映射 IP
webrtc_port_range_begin=50000
webrtc_port_range_end=50000
webrtc_turn_host=www.signalling-nexartc.cn
webrtc_turn_port=3478
webrtc_turn_secret=<TURN_SECRET>
webrtc_turn_ttl=3600
webrtc_turn_user_id=device-mi9-001
webrtc_stun_server=stun:www.signalling-nexartc.cn:3478
# `listen_port_h5=50000` 与 [webrtc] 的 UDP mux 端口统一；relay 仅作最后兜底

# 配置路径（APK 模式）
/data/user/0/com.nexartc.cloudapp/files/cae/config/CaeConfig.ini
# 启动
adb shell su -c 'am start-foreground-service -n com.nexartc.cloudapp/com.nexartc.cae.service.CaeServerService'
# 日志关键字
adb logcat -s CAE-Server | grep -iE 'turn_host|TURN REST|WebRTC config'
```

说明：纯 `/data/local/tmp/cae` + `app_process` 在本机曾因缺少 `ImeBridgeDispatcher` 导致 `JNI_ERR`；**优先用 root APK 服务路径**。

### 3.5 device 页面 UI（Hub 托管时的默认值）

| 阶段 | 设计要求的 UI | 2026-07-12 之前（错误） | 修复后 |
|------|---------------|-------------------------|--------|
| **P0（当前）** | 页面在 Hub 域名；TURN 同源拉取；**WSS 仍填云手机 CAE 可达地址**（Phase 3 前） | 硬编码默认 `www.nexartc.com:50000` | Hub 域名下 **Host 留空**、Port 默认 `50000`，并提示「非本页域名」 |
| **Phase 3（目标）** | 去掉 CAE IP 输入，改为 **deviceId** + `wss://<hub>/ws` | — | 待实施 |

**为何不能默认 `www.signalling-nexartc.cn`？**  
Hub 域名只托管 **HTTPS 静态页 + TURN 签发**；云手机 WebRTC **信令 WSS 仍在 CAE**（`listen_port_h5=50000`），Phase 3 前浏览器必须能 **直连 CAE 的 WSS**（家庭公网 IP + 端口映射，或同 LAN IP）。把 Hub 域名填进 WSS 会连到 Hub 代理，而 VPS 上 `proxy_config` 的 upstream 默认 `127.0.0.1:50000`，**无法到达家庭内网 CAE**。

**为何不能默认 `www.nexartc.com`？**  
这是旧版直连 CAE 的遗留常量，与当前 VPS `signalling-nexartc.cn` 部署无关。

**联调示例（Phase 3 前，CAE 在同 LAN）：**

```
https://www.signalling-nexartc.cn/device/?turn_token=6dc053f3124cda23a4279e8f34e3ea0e&cae_ip=192.168.x.x&cae_port=50000
```

刷新后若仍看到 `www.nexartc.com`，清浏览器该站点 **localStorage**（键 `cloudphone.device.wssHost`）或强刷。

---

## 4. 验证命令与实测结果

### 4.1 Hub HTTP（经 IP，规避备案拦截）

```bash
source /tmp/nexartc-vps-deploy/env   # 或从 VPS 拉取 /etc/nexartc/hub.env
HUB=https://120.79.21.28

curl -sk "$HUB/api/v1/health"
# {"ok":true}

curl -sk -o /dev/null -w '%{http_code}\n' "$HUB/device/"
# 200

curl -sk -o /dev/null -w '%{http_code}\n' "$HUB/api/v1/turn-credentials?clientId=x"
# 401

curl -sk -H "Authorization: Bearer $STREAM_TOKEN" \
  "$HUB/api/v1/turn-credentials?clientId=vps-check" | jq .
# iceServers 含 stun/turn/turns，username=expiry:clientId
```

### 4.2 coturn 公网中继

```bash
# 开发机已编译 coturn-4.13.0/build-local
export LD_LIBRARY_PATH=/tmp/libevent-local/usr/lib/x86_64-linux-gnu:$LD_LIBRARY_PATH
BIN=coturn-4.13.0/build-local/bin

# VPS 上临时 peer（验证用）
ssh root@signalling-nexartc.cn 'turnutils_peer -L 0.0.0.0 -p 3480 &'

$BIN/turnutils_uclient -u tester -W "$TURN_SECRET" \
  -e 120.79.21.28 -r 3480 -n 10 -m 1 -c 120.79.21.28
# → tot_send_msgs=10, tot_recv_msgs=10, lost 0
# → Received relay addr: 120.79.21.28:<port>
```

Hub 签发的 `username`/`credential` 同样可直接 `-u/-w` 打通（REST 算法一致）。

### 4.3 浏览器 `typ relay`

无头 Chrome（puppeteer）用 Hub 返回的 `iceServers` + `iceTransportPolicy:'relay'`，实测 **3** 条 `typ relay`，地址均为 `120.79.21.28`。

页面手工验证：

```
https://www.signalling-nexartc.cn/device/?turn_token=6dc053f3124cda23a4279e8f34e3ea0e&force_relay=1
```

在页面日志 / `chrome://webrtc-internals` 中确认 relay。

### 4.4 域名 HTTPS 说明

- 多数网络：`https://www.signalling-nexartc.cn/` 可正常访问（用户侧已确认）
- 个别外网 ISP：TLS 握手阶段仍可能 RST，此时用 §1 兜底 IP 地址
- HTTP `:80` 在未备案场景下可能返回阿里云拦截页（与 Hub 443 无关）

---

## 5. 运维速查

```bash
# 状态
ssh root@signalling-nexartc.cn 'systemctl status nexartc-hub nexartc-coturn --no-pager'

# 日志
journalctl -u nexartc-hub -f
journalctl -u nexartc-coturn -f

# 重载配置
# 改 /etc/nexartc/hub.env 或 turnserver.conf 后：
systemctl restart nexartc-coturn nexartc-hub
```

密钥轮换见 **§8.5**（`TURN_SECRET` 与 CAE 配置需同步）。

---

## 8. 密钥说明：`STREAM_TOKEN` 与 `TURN_SECRET`

Mode A 里有两套密钥，职责不同，**不可混用**。

### 8.1 对比

| | **STREAM_TOKEN** | **TURN_SECRET** |
|---|------------------|-----------------|
| **是什么** | Hub HTTP API 的 Bearer 访问令牌 | coturn REST API 的 HMAC 根密钥 |
| **谁持有** | 浏览器（经 `?turn_token=`）、测试脚本、未来登录态 | **仅服务端**：Hub、coturn、CAE（ini） |
| **保护什么** | `GET /api/v1/turn-credentials` 不被任意公网用户滥用 | TURN 中继鉴权；防止伪造 `username`/`credential` |
| **是否下发到 JS** | 是（页面 URL / localStorage） | **否**（禁止出现在 `device/` 前端代码或 Git） |
| **配置位置** | Hub 环境变量 `STREAM_TOKEN` | Hub `TURN_SECRET` + coturn `--static-auth-secret` + CAE `webrtc_turn_secret` |
| **泄露后果** | 他人可调用 Hub 签发**短时** TURN 凭据（受 TTL 限制） | 他人可在 TTL 内**直接**向 coturn 申请中继，危害更大 |
| **算法** | 字符串相等比较（`Authorization: Bearer …`） | `credential = Base64(HMAC-SHA1(TURN_SECRET, expiry:userId))` |

### 8.2 为什么需要 `STREAM_TOKEN`？

浏览器连接云手机前，需要先向 Hub 拉取 **短时** `iceServers`（含 TURN `username`/`credential`）。该接口若完全公开，任意人均可向你的 coturn 申请中继，占用带宽与端口。

因此 Hub 在 Phase 1 采用最小鉴权（设计文档 **P1-a**）：

1. 请求头：`Authorization: Bearer <STREAM_TOKEN>`
2. 或页面 URL：`?turn_token=<STREAM_TOKEN>`（`device/src/turn.ts` 会写入 `localStorage`，下次可省略）
3. 未配置 `STREAM_TOKEN` 时 Hub **开放**该接口（仅适合本机联调，**生产必须设置**）

`STREAM_TOKEN` **不会**代替 TURN 登录 coturn；Hub 收到合法 Bearer 后，用 **`TURN_SECRET`** 在服务端算出短时 `username`/`credential` 再返回给浏览器（方案 A）。

### 8.3 数据流（简化）

```
浏览器  --Bearer STREAM_TOKEN-->  Hub /api/v1/turn-credentials
                                        |
                                        | HMAC(TURN_SECRET, expiry:userId)
                                        v
浏览器  <-- iceServers + username/credential --+
                                                |
浏览器 / CAE  --username+credential-->  coturn Allocate
         (CAE 本地用同一 TURN_SECRET 自行 HMAC，方案 B)
```

### 8.4 如何获取 & 当前 VPS 测试用 token

**从 VPS 读取（权威来源）：**

```bash
/usr/bin/ssh -i ~/.ssh/id_ed25519 root@signalling-nexartc.cn \
  'grep ^STREAM_TOKEN= /etc/nexartc/hub.env'
```

**从本机部署缓存读取**（若执行过 `test/turn/deploy_vps.sh`）：

```bash
grep ^STREAM_TOKEN= /tmp/nexartc-vps-deploy/env
```

**当前实例（2026-07-12 部署，`www.signalling-nexartc.cn`）测试用：**

| 项 | 值 |
|----|-----|
| `STREAM_TOKEN` | `6dc053f3124cda23a4279e8f34e3ea0e` |

**可直接打开的测试页：**

```
https://www.signalling-nexartc.cn/device/?turn_token=6dc053f3124cda23a4279e8f34e3ea0e
```

**API 自测：**

```bash
curl -s -H "Authorization: Bearer 6dc053f3124cda23a4279e8f34e3ea0e" \
  "https://www.signalling-nexartc.cn/api/v1/turn-credentials?clientId=test" | jq .
```

无 token → `401 missing bearer token`；错误 token → `401 invalid token`。

> **安全提示：** 上述 token 已写入本文档便于联调。对外正式运营前建议轮换（§8.5），且 **切勿** 将 `TURN_SECRET` 写入文档或前端。

### 8.5 轮换 / 新建

```bash
# 生成新 STREAM_TOKEN（Hub API 用）
openssl rand -hex 16

# 生成新 TURN_SECRET（Hub + coturn + CAE 必须一致）
openssl rand -hex 24
```

修改 `/etc/nexartc/hub.env` 后：

```bash
systemctl restart nexartc-coturn nexartc-hub
```

若改了 `TURN_SECRET`，还需同步 CAE `CaeConfig.ini` 的 `webrtc_turn_secret` 并重启 CAE。

---

## 6. 与设计文档阶段对照

| 阶段 | 状态 |
|------|------|
| L0/L1/L2 本地（§14） | ✅ 已完成（2026-07-12 开发机） |
| P0 Hub+coturn+device → VPS | ✅ 功能通过 |
| P1 家庭 CAE 出站 `/agent` | ✅ Phase 3 已实施 |
| hybrid ICE + 真机 WebRTC | ⚠️ 建联 OK；relay 媒体质量待优化 |
| p2p-only 网络诊断 | ✅ 已验证不可行（同 NAT hairpin） |

详细联调记录、排障 Checklist、演进路线图见 **[`nexartc-turn-mode-a-implementation.md` §16](./nexartc-turn-mode-a-implementation.md#16-联调会话记录排障手册与演进方向2026-07-12--2026-07-13)**。

---

## 7. 后续必做

1. **P0 媒体质量**：✅ Hub WSS A/V 抑制 + 降低 BWE 初值 + 固定 50000 端口（见实现文档 §16.6）。  
2. **Agent 保活**：✅ Hub 模式会话结束保持 Signal Agent；断线自动重连已有。Admin lastPing 告警待做。  
3. **发布流程**：✅ `deploy_vps.sh` 默认跑 `build_all.sh`（`--skip-build` 跳过）。  
4. 阿里云安全组确认 UDP 中继端口段长期开放；磁盘占用已 ~90%，建议清理 `/root` 大包。  
5. 正式运营前轮换 `STREAM_TOKEN` / `AGENT_TOKEN` / `TURN_SECRET`（§8.5）；勿提交 Git。  
6. ICP 备案完成后统一 `www.signalling-nexartc.cn/device/` 入口，避免 IP/域名双轨 stale bundle。  
7. Phase 5：JWT 登录替代固定 `STREAM_TOKEN`。  
8. **STUN 公网 IP 发现**：✅ `PublicIpResolver`（见 `cae-stun-public-ip-discovery.md`）；设备侧清空 `webrtc_public_ip` 并确保路由器 DNAT `50000/UDP`。

---

## 9. 快速排障（联调常见）

| 现象 | 检查 | 处理 |
|------|------|------|
| `Hub join 失败: agent offline` | `curl -sk -H "Authorization: Bearer $STREAM_TOKEN" https://120.79.21.28/api/v1/agents` | 启动 CAE：`adb shell am start-foreground-service -n com.nexartc.cloudapp/com.nexartc.cae.service.CaeServerService` |
| 面板仍是 `mode=hybrid` 但想测 p2p | URL 是否含 `ice_mode=p2p`；是否硬刷新 | 用 IP 打开 device 页 |
| ICE connected 但 `decoded=0`、lost 很大 | Stats `ICE path`；CAE logcat `sendFrame`/`PLI` | 见实现文档 §16.6 P0 |
| p2p 模式 ~19s failed | 预期行为（同公网 srflx hairpin 失败） | 生产用 hybrid |

```bash
# 一键 P0/P3 验收
./test/turn/deploy_vps.sh --verify-only

# 重新部署 device 页（改前端后）
cd nexartc-cloudPhoneAccess-web && ./build_all.sh --skip-server
cd ../.. && ./test/turn/deploy_vps.sh
```

# nexartc Mode A — 安装部署手册

> 日期：2026-07-20  
> 适用范围：家庭内网 CAE（Android）+ 公网 VPS（Signal Hub / coturn / device 页）  
> 关联文档：  
> - [`nexartc-turn-mode-a-implementation.md`](./nexartc-turn-mode-a-implementation.md)（设计与 ICE 策略）  
> - [`nexartc-turn-mode-a-vps-deployment.md`](./nexartc-turn-mode-a-vps-deployment.md)（VPS 验收记录）  
> - [`cae-stun-public-ip-discovery.md`](./cae-stun-public-ip-discovery.md)（STUN 动态公网 IP）

本文说明 **编译 → 产物路径 → 安装 → 配置 → 测试 URL → 日志定位** 的完整闭环。当前生产拓扑以 **VPS `120.79.21.28` / `www.signalling-nexartc.cn`** 为准。

---

## 0. 拓扑与模块一览

```
开发机 (本仓库)
├─ nexartc-cloudPhoneAccess-web/     Hub(server) + device 页
├─ nexartc-cloud-phone-access-engine/  CAE server (Android APK)
├─ coturn-4.13.0/                    TURN 源码（本地测 + 工具）
└─ test/turn/deploy_vps.sh           VPS 一键部署

公网 VPS 120.79.21.28
├─ nexartc-hub.service      Node Hub  :443 / :9443
│     WEB_ROOT=/opt/nexartc/hub   → device/ + server/
├─ nexartc-coturn.service   TURN/STUN UDP/TCP 3478, TLS 5349
└─ /etc/nexartc/hub.env     密钥（勿提交 Git）

家庭内网 Android（CAE）
└─ com.nexartc.cloudapp (root APK)
      WSS → wss://120.79.21.28/agent
      ICE  → host / hybrid / relay（见 webrtc_ice_mode）
```

| 模块 | 仓库路径 | 运行位置 | 默认日志 |
|------|----------|----------|----------|
| coturn | `coturn-4.13.0/` + `test/turn/vps/` | VPS | `verbose` → journald |
| Signal Hub | `nexartc-cloudPhoneAccess-web/server/` | VPS | `/opt/nexartc/hub/server/logs/` |
| Web device | `nexartc-cloudPhoneAccess-web/device/` | VPS 静态页 | 浏览器 Console |
| CAE | `nexartc-cloud-phone-access-engine/` | Android 真机 | logcat + 文件日志 |

---

## 1. 开发机前置条件

```bash
# 仓库根目录
cd /home/harry/swork/cloud_phone_p2p

# SSH（注意：若 shell 包装了 ssh，请用绝对路径）
/usr/bin/ssh -i ~/.ssh/id_ed25519 root@signalling-nexartc.cn 'hostname'

# Android
export ANDROID_SDK_ROOT=~/Android/Sdk   # 按本机路径调整
export ANDROID_NDK_HOME=$ANDROID_SDK_ROOT/ndk/<version>
export JAVA_HOME=...                    # JDK 11+

# Node 18+（构建 web）；Rust/Cargo（若编 QUIC，Mode A WebRTC 可不依赖）
adb version
node -v
```

证书（Hub / TURNS）：

```bash
ls cert/unpacked/Nginx/fullchain.crt cert/unpacked/Nginx/private.key
# SAN 须含 signalling-nexartc.cn / www.signalling-nexartc.cn
```

密钥本地缓存（由 `deploy_vps.sh` 生成，**勿提交**）：

```text
/tmp/nexartc-vps-deploy/env
```

| 变量 | 用途 |
|------|------|
| `TURN_SECRET` | coturn `static-auth-secret`；与 CAE `webrtc_turn_secret`、Hub HMAC **同一密钥** |
| `STREAM_TOKEN` | Hub API Bearer；浏览器 URL `?turn_token=` |
| `AGENT_TOKEN` | CAE `/agent` 注册口令；`CaeConfig.ini` → `agent_token` |

---

## 2. coturn（TURN/STUN）— 编译 / 安装 / 部署

### 2.1 本地编译（开发机工具 + 联调）

脚本：`test/turn/build_coturn_local.sh`

```bash
cd /home/harry/swork/cloud_phone_p2p
./test/turn/build_coturn_local.sh
```

**产物：**

```text
coturn-4.13.0/build-local/bin/turnserver
coturn-4.13.0/build-local/bin/turnutils_uclient
coturn-4.13.0/build-local/bin/turnutils_peer
```

运行时若使用解包的 libevent：

```bash
export LD_LIBRARY_PATH=/tmp/libevent-local/usr/lib/x86_64-linux-gnu:$LD_LIBRARY_PATH
# 或 source test/turn/env_local.sh
```

### 2.2 VPS 上安装（生产推荐 apt）

当前 VPS 使用 **系统 apt coturn**（非必须自编译），由 systemd 托管：

| 项 | 路径 / 值 |
|----|-----------|
| 单元 | `test/turn/vps/nexartc-coturn.service` → `/etc/systemd/system/` |
| 配置 | `test/turn/vps/turnserver.conf` → `/etc/nexartc/turnserver.conf` |
| 密钥 | `/etc/nexartc/hub.env` 中 `TURN_SECRET`（`--static-auth-secret`） |
| 证书 | `/etc/nexartc/certs/turn.{crt,key}`（与域名证书同源） |
| 监听 | UDP/TCP **3478**；TLS **5349** |
| 中继端口 | `49152–65535` |
| `external-ip` | `120.79.21.28/<内网 NIC IP>` |
| `realm` | `nexartc.com` |
| 日志 | **`verbose` 默认开启**；`--log-file=stdout` → `journalctl -u nexartc-coturn` |

**一键部署（含 coturn 配置同步）：**

```bash
./test/turn/deploy_vps.sh
```

**手工重启 / 看日志：**

```bash
/usr/bin/ssh -i ~/.ssh/id_ed25519 root@signalling-nexartc.cn '
  systemctl restart nexartc-coturn
  systemctl status nexartc-coturn --no-pager
  journalctl -u nexartc-coturn -n 80 --no-pager
'
```

**公网 Allocate 自检（开发机）：**

```bash
source /tmp/nexartc-vps-deploy/env
# 先向 Hub 取临时凭据，再 turnutils_uclient（见 deploy_vps.sh verify 段）
./test/turn/deploy_vps.sh --verify-only
```

### 2.3 从源码部署到 VPS（可选）

若要用自编译 `turnserver` 替换 apt 包：

1. 在 VPS 或交叉环境编译 `coturn-4.13.0`  
2. 将二进制放到例如 `/usr/local/bin/turnserver`  
3. 修改 `nexartc-coturn.service` 的 `ExecStart` 路径  
4. 保持 `turnserver.conf` + `TURN_SECRET` 不变，重启服务  

日常联调 **不必** 走本路径；以 apt + `deploy_vps.sh` 为准。

---

## 3. Signal Hub（信令服务器）— 编译 / 安装 / 部署

Hub 源码：`nexartc-cloudPhoneAccess-web/server/`（TypeScript → `out/server/main.js`）。

### 3.1 编译

```bash
cd /home/harry/swork/cloud_phone_p2p/nexartc-cloudPhoneAccess-web
./build_all.sh                 # 同时构建 device + server → out/
# 或仅 server：
./build_all.sh --skip-device
```

**产物：**

```text
nexartc-cloudPhoneAccess-web/out/
├── device/                 # Web 客户端静态页
├── admin.html
└── server/
    ├── main.js             # Hub 入口
    ├── package.json
    ├── node_modules/       # 运行时依赖（ws 等）
    └── logs/               # 本地运行时日志目录模板
```

### 3.2 部署到 VPS

**推荐一键：**

```bash
cd /home/harry/swork/cloud_phone_p2p
./test/turn/deploy_vps.sh              # 默认先 build_all.sh 再 rsync
./test/turn/deploy_vps.sh --skip-build # 使用已有 out/
```

部署后 VPS 布局：

```text
/opt/nexartc/hub/
├── device/          ← WEB_ROOT 下的静态页
└── server/main.js   ← nexartc-hub.service 工作目录

/etc/nexartc/hub.env           # 密钥 + HUB_PORTS + LOG_DIR
/etc/systemd/system/nexartc-hub.service
```

关键环境变量见 `test/turn/vps/hub.env.example`：

```bash
SIGNAL_MODE=both
HUB_PORTS=443:cae-server,9443:cae-server
WEB_ROOT=/opt/nexartc/hub
LOG_DIR=/opt/nexartc/hub/server/logs
TURN_HOST=www.signalling-nexartc.cn
TURN_PORT=3478
...
```

**服务管理：**

```bash
systemctl restart nexartc-hub
systemctl status nexartc-hub --no-pager
journalctl -u nexartc-hub -f --no-pager
```

**健康检查：**

```bash
curl -sk https://120.79.21.28/api/v1/health
# 期望：{"ok":true,...}
```

> **注意：** 部分网络对 `https://www.signalling-nexartc.cn/` 存在备案/SNI 拦截。联调请优先用 **IP**：`https://120.79.21.28/`。Hub 与 device 实际已部署在同一 VPS，域名解析到该 IP。

### 3.3 Hub 日志（默认开启）

| 通道 | 位置 | 说明 |
|------|------|------|
| 应用文件日志 | `$LOG_DIR` → `/opt/nexartc/hub/server/logs/` | 轮转：`serve_https_*.log`（默认约 4×2MB） |
| systemd | `journalctl -u nexartc-hub` | 启动、WSS join/register、API |
| 关键关键词 | `[agent] registered` / `[session] joined` / `[audit]` | Phase 3 路由 |

```bash
# VPS 上
tail -f /opt/nexartc/hub/server/logs/serve_https_1.log
journalctl -u nexartc-hub -n 100 --no-pager | grep -E 'agent|session|turn|error'
```

---

## 4. Web 客户端（device 页）— 编译 / 安装 / 部署

源码：`nexartc-cloudPhoneAccess-web/device/`（Vite + TypeScript）。

### 4.1 编译

```bash
cd nexartc-cloudPhoneAccess-web
./build_all.sh
# 或
cd device && npm install && npm run build
# 产物在 device/dist/，build_all.sh 会同步到 out/device/
```

**产物地址：**

```text
nexartc-cloudPhoneAccess-web/out/device/
├── index.html
└── assets/index-<hash>.js    # 每次构建 hash 变化，用于确认是否刷到最新包
```

### 4.2 部署服务器地址

| 入口 | URL | 说明 |
|------|-----|------|
| **推荐（联调）** | `https://120.79.21.28/device/` | 与 Hub 同机；bundle 最新 |
| 域名 | `https://www.signalling-nexartc.cn/device/` | 同机；部分网络 HTTPS 可能被拦 |
| 兜底端口 | `https://120.79.21.28:9443/device/` | 证书可能提示与 IP 不匹配 |

部署命令与 Hub 相同：`./test/turn/deploy_vps.sh`（rsync `out/device/` → `/opt/nexartc/hub/device/`）。

**确认已部署的 JS bundle：**

```bash
curl -sk https://120.79.21.28/device/ | grep -oE 'index-[A-Za-z0-9_-]+\.js'
# 与本地 out/device/assets/ 下文件名对比
```

浏览器请 **Ctrl+Shift+R** 硬刷新，避免缓存旧 JS。

### 4.3 Web 客户端日志（默认开启）

device 页默认在页面日志面板 + DevTools Console 输出，前缀示例：

| 日志关键字 | 含义 |
|------------|------|
| `ICE 模式: host\|hybrid\|relay\|p2p` | 当前 ICE 策略 |
| `ICE 配置: mode=... TURN=ON\|OFF` | PeerConnection iceServers |
| `connectionState=connected` | PC 已连通 |
| `ICE path: ... (host\|p2p\|relay)` | **选中** ICE 路径（最关键） |
| `TURN 凭据已获取` | hybrid/relay 拉到 Hub 临时凭据 |

无需额外开关；打开页面即有完整信令 / ICE / Stats 日志（Stats 约每 5s）。

---

## 5. CAE Server — 编译 / 产物 / 安装 / 配置

工程：`nexartc-cloud-phone-access-engine/`。Mode A 联调使用 **root flavor Debug APK**。

### 5.1 环境

```bash
export ANDROID_SDK_ROOT=...
export ANDROID_NDK_HOME=...
export JAVA_HOME=...
adb devices   # 确认真机在线
```

可选：首次需要 stub / 依赖时：

```bash
cd nexartc-cloud-phone-access-engine
./scripts/build_stubs.sh    # 若缺 libtquic 等 stub
```

### 5.2 编译

**推荐（APK，含 JNI）：**

```bash
cd nexartc-cloud-phone-access-engine
./gradlew assembleRootDebug --no-daemon
```

**产物地址：**

```text
# Gradle 原始输出
app/build/outputs/apk/root/debug/app-root-debug.apk

# 惯例拷贝（安装脚本常用）
output/server/CaeServer-root-debug.apk

# 亦可：
cp -f app/build/outputs/apk/root/debug/app-root-debug.apk \
      output/server/CaeServer-root-debug.apk
```

其它构建入口（完整 native/Java 打包）：

```bash
./scripts/build_cae.sh              # 全量
./scripts/build_cae.sh --native-only
./scripts/build_cae.sh --release
```

输出目录同样在 `output/server/`（`libcae_native.so`、`deploy.sh`、`start_cae.sh` 等）。Mode A 真机联调以 **root APK** 为主。

### 5.3 安装到真机

```bash
SERIAL=<adb_serial>   # 例：9898d727
APK=nexartc-cloud-phone-access-engine/output/server/CaeServer-root-debug.apk

adb -s "$SERIAL" install -r "$APK"
adb -s "$SERIAL" shell am force-stop com.nexartc.cloudapp
adb -s "$SERIAL" shell am start-foreground-service \
  -n com.nexartc.cloudapp/com.nexartc.cae.service.CaeServerService
```

亦可用 `output/server/deploy.sh`（app_process 路径，视现场脚本而定）。

### 5.4 配置

**运行时配置文件（优先）：**

```text
/data/user/0/com.nexartc.cloudapp/files/cae/config/CaeConfig.ini
```

兼容/历史路径：

```text
/data/local/tmp/cae/config/CaeConfig.ini
```

APK 内置模板：

```text
app/src/main/assets/config/CaeConfig.ini
```

Settings UI 可能写入 SharedPreferences：

```text
/data/user/0/com.nexartc.cloudapp/shared_prefs/cae_settings.xml
```

> SharedPreferences 中的字段会在启动时 patch 进 ini；改 `webrtc_public_ip` / `webrtc_local_ip` 时两边都要核对。

#### 5.4.1 Mode A 最小配置示例

```ini
[webrtc]
webrtc_local_ip=192.168.x.x          ; 必须等于手机 wlan0 真实 IP；错 IP 会导致 host 失败
webrtc_public_ip=                     ; 留空，由 STUN 动态发现
webrtc_stun_server=stun:120.79.21.28:3478
webrtc_enable_udp_mux=1
webrtc_port_range_begin=50000
webrtc_port_range_end=50000
webrtc_turn_host=120.79.21.28
webrtc_turn_port=3478
webrtc_turn_secret=<TURN_SECRET>      ; 与 VPS /etc/nexartc/hub.env 相同
webrtc_turn_ttl=3600
webrtc_turn_user_id=device-mi9-001
webrtc_ice_mode=host                  ; host | hybrid | p2p | relay（默认 host）
webrtc_force_relay=0
webrtc_low_latency=1

[signal]
signal_agent_url=wss://120.79.21.28/agent
device_id=device-mi9-001
agent_token=<AGENT_TOKEN>
signal_disable_tls_verify=1           ; IP 直连调试用；证书匹配域名时可改 0
```

查看手机当前局域网 IP：

```bash
adb shell "ip -4 -o addr show wlan0 | awk '{print \$4}' | cut -d/ -f1"
```

写入示例：

```bash
adb shell "su -c 'sed -i \"s/^webrtc_local_ip=.*/webrtc_local_ip=192.168.124.101/\" \
  /data/user/0/com.nexartc.cloudapp/files/cae/config/CaeConfig.ini'"
```

#### 5.4.2 ICE 模式说明

| `webrtc_ice_mode` | CAE 行为 | 浏览器默认 / URL |
|-------------------|----------|------------------|
| **host**（默认） | 仅发 host；Offer 带 `a=ice-lite`；不挂 TURN | 默认；可 `&ice_mode=host` |
| **hybrid** | host/srflx + TURN fallback | `&ice_mode=hybrid` |
| **p2p** | 不上报 relay（调试） | `&ice_mode=p2p` |
| **relay** | 仅 relay | `&ice_mode=relay` 或 `&force_relay=1` |

路由器若要测 **公网 host**：需 DNAT **UDP 50000** → 手机；映射端口必须与 `webrtc_port_range_*` 一致，否则不注入公网 host（见 STUN 设计文档）。

### 5.5 CAE 日志（默认开启）

CAE **默认即详细日志**，无需额外 verbose 开关。

| 通道 | 位置 | 说明 |
|------|------|------|
| logcat | tag `CAE` | 实时：`adb logcat -s CAE` |
| 文件日志 | `/data/local/tmp/cae/logs/cae_server_*.log` | 轮转文件，容量较大 |
| 应用私有 | `/data/user/0/com.nexartc.cloudapp/files/cae/logs/` | 视启动路径而定 |

**常用过滤：**

```bash
SERIAL=<adb_serial>

# 实时
adb -s "$SERIAL" logcat -s CAE | grep -iE \
  'ice_mode|ice-lite|WebRTC config|candidate|PeerConnection|registered|FAILED|OnReady'

# 文件（最新一轮会话）
adb -s "$SERIAL" shell "su -c '
  LOG=\$(ls -t /data/local/tmp/cae/logs/cae_server_*.log | head -1)
  grep -E \"ice_mode|candidate sent|state=2\\(Connected\\)|typ relay|TURN REST|OnReady\" \$LOG | tail -80
'"
```

| 日志关键字 | 含义 |
|------------|------|
| `WebRTC config: ice_mode=host local_ip='...'` | 策略与注入局域网 IP |
| `discovered public IP via STUN ... -> x.x.x.x:50000` | 公网 host 可用 |
| `offer annotated with a=ice-lite` | host 模式角色（浏览器 controlling） |
| `candidate sent ... typ host` | CAE 发出的 host（确认 IP 是否正确） |
| `TURN REST credentials` | 本会话启用了 TURN（hybrid/relay） |
| `state=2(Connected)` | ICE/DTLS 已通 |
| `All 9 DataChannels open` / `OnReady` | 业务就绪，开始推流 |
| `track not open ... pc_state=1` | 仍 Connecting，媒体未通路 |
| `CaeSignalAgent: registered with Hub` | Agent 在线 |

确认 Agent：

```bash
source /tmp/nexartc-vps-deploy/env
curl -sk -H "Authorization: Bearer $STREAM_TOKEN" \
  https://120.79.21.28/api/v1/agents
# 期望含 deviceId、hasSession
```

---

## 6. 一键推荐流程（从零到可测）

```bash
cd /home/harry/swork/cloud_phone_p2p

# 1) 构建并部署 Hub + device + coturn 配置到 VPS
./test/turn/deploy_vps.sh

# 2) 编译并安装 CAE
cd nexartc-cloud-phone-access-engine
./gradlew assembleRootDebug --no-daemon
cp -f app/build/outputs/apk/root/debug/app-root-debug.apk output/server/CaeServer-root-debug.apk
SERIAL=$(adb devices | awk 'NR>1 && $2=="device"{print $1; exit}')
adb -s "$SERIAL" install -r output/server/CaeServer-root-debug.apk

# 3) 写入配置（示例）后重启服务
PHONE_IP=$(adb -s "$SERIAL" shell "ip -4 -o addr show wlan0 | awk '{print \$4}' | cut -d/ -f1" | tr -d '\r')
source /tmp/nexartc-vps-deploy/env
adb -s "$SERIAL" shell "su -c '
  CFG=/data/user/0/com.nexartc.cloudapp/files/cae/config/CaeConfig.ini
  sed -i \"s/^webrtc_local_ip=.*/webrtc_local_ip=$PHONE_IP/\" \$CFG
  sed -i \"s/^webrtc_ice_mode=.*/webrtc_ice_mode=host/\" \$CFG
  sed -i \"s/^webrtc_turn_secret=.*/webrtc_turn_secret=$TURN_SECRET/\" \$CFG
  sed -i \"s/^agent_token=.*/agent_token=$AGENT_TOKEN/\" \$CFG
  sed -i \"s|^signal_agent_url=.*|signal_agent_url=wss://120.79.21.28/agent|\" \$CFG
'"
adb -s "$SERIAL" shell am force-stop com.nexartc.cloudapp
adb -s "$SERIAL" shell am start-foreground-service \
  -n com.nexartc.cloudapp/com.nexartc.cae.service.CaeServerService

# 4) 生成测试 URL（见下一节）
```

---

## 7. 测试方法与测试 URL

### 7.1 如何生成测试 URL

```bash
source /tmp/nexartc-vps-deploy/env
# STREAM_TOKEN 即浏览器 turn_token

DEVICE_ID=device-mi9-001   # 须与 CAE CaeConfig.ini device_id 一致

# 基础（默认 ice_mode=host）
echo "https://120.79.21.28/device/?turn_token=${STREAM_TOKEN}&device_id=${DEVICE_ID}"

# 显式模式
echo "https://120.79.21.28/device/?turn_token=${STREAM_TOKEN}&device_id=${DEVICE_ID}&ice_mode=host"
echo "https://120.79.21.28/device/?turn_token=${STREAM_TOKEN}&device_id=${DEVICE_ID}&ice_mode=hybrid"
echo "https://120.79.21.28/device/?turn_token=${STREAM_TOKEN}&device_id=${DEVICE_ID}&ice_mode=relay"
echo "https://120.79.21.28/device/?turn_token=${STREAM_TOKEN}&device_id=${DEVICE_ID}&ice_mode=p2p"
```

| 查询参数 | 作用 |
|----------|------|
| `turn_token` | = `STREAM_TOKEN`；拉 TURN 凭据（host 模式会跳过拉取，但仍建议带上便于切 hybrid） |
| `device_id` | Hub 房间绑定的 CAE `device_id` |
| `ice_mode` | `host` / `hybrid` / `relay` / `p2p`（省略则 **host**） |
| `force_relay=1` | 等价 `ice_mode=relay` |
| `no_relay=1` | 等价 `ice_mode=p2p` |

当前环境示例（token 以 `/tmp/nexartc-vps-deploy/env` 为准，会轮换）：

```text
https://120.79.21.28/device/?turn_token=<STREAM_TOKEN>&device_id=device-mi9-001
```

### 7.2 功能测试清单

| # | 步骤 | 期望 |
|---|------|------|
| 1 | `GET /api/v1/health` | `ok: true` |
| 2 | `GET /api/v1/agents`（Bearer） | 出现 `device-mi9-001` |
| 3 | 打开测试 URL，硬刷新 | 页面日志：`ICE 模式: host`，bundle 为最新 hash |
| 4 | 点连接 | Hub：`client join`；CAE：`registered` 后 `CreatePeerConnection` |
| 5 | ICE | CAE：`candidate sent ... typ host` 且 IP=手机真实局域网 IP；`state=2(Connected)` |
| 6 | 浏览器 Stats | `ICE path: ... (host)`；有视频帧 |
| 7 | （可选）`ice_mode=relay` | 两端均为 relay；`TURN REST` 日志出现 |
| 8 | （可选）`deploy_vps.sh --verify-only` | H1–H5 + uclient + Phase3 OK |

### 7.3 自动化 / 脚本

```bash
./test/turn/deploy_vps.sh --verify-only   # Hub + coturn + Phase3
./test/turn/test_hub_api.sh               # Hub API
./test/turn/test_hub_phase3.mjs           # /agent + /ws 管道
./test/turn/test_cae_public_ip_resolver.sh
./test/turn/run_all_tests.sh              # 本地套件（视环境）
```

---

## 8. 各模块日志速查（默认全开）

### 8.1 总表

| 模块 | 默认级别 | 查看命令 |
|------|----------|----------|
| **coturn** | `verbose`（conf 已开） | `journalctl -u nexartc-coturn -f` |
| **Hub** | INFO + 文件轮转 | `journalctl -u nexartc-hub -f`；`tail -f /opt/nexartc/hub/server/logs/serve_https_1.log` |
| **device 页** | 页面 + Console 全开 | 浏览器 F12 / 页内日志面板；搜 `ICE path` |
| **CAE** | 全量 CAE tag + 文件 | `adb logcat -s CAE`；`/data/local/tmp/cae/logs/cae_server_*.log` |

### 8.2 典型故障对照

| 现象 | 优先看 | 常见原因 |
|------|--------|----------|
| Agent 不在线 | CAE `SignalAgent`；Hub `[agent]` | `signal_agent_url` / `agent_token` / TLS |
| 一直 Connecting | CAE `candidate sent` 的 IP | `webrtc_local_ip` ≠ wlan0 |
| 无视频但 Connected | CAE `sendFrame` / 浏览器 decoded | 码率/解码；非 ICE 问题 |
| 只有 relay 才通 | 浏览器 `ICE path` | 同 NAT hairpin；host 需局域网或 DNAT 50000 |
| 页面旧逻辑 | `index-*.js` hash | 未 `deploy_vps.sh` 或未硬刷新；域名被拦应用 IP |
| TURN 401 | Hub credentials API | `turn_token` ≠ `STREAM_TOKEN` |

### 8.3 建议的一次排障命令包

```bash
source /tmp/nexartc-vps-deploy/env
SERIAL=$(adb devices | awk 'NR>1 && $2=="device"{print $1; exit}')

echo "=== Hub agents ==="
curl -sk -H "Authorization: Bearer $STREAM_TOKEN" https://120.79.21.28/api/v1/agents; echo

echo "=== device bundle ==="
curl -sk https://120.79.21.28/device/ | grep -oE 'index-[A-Za-z0-9_-]+\.js'

echo "=== CAE recent WebRTC ==="
adb -s "$SERIAL" shell "su -c '
  LOG=\$(ls -t /data/local/tmp/cae/logs/cae_server_*.log | head -1)
  grep -E \"ice_mode|local_ip|candidate sent|Connected|FAILED|OnReady|TURN REST\" \$LOG | tail -40
'"

echo "=== coturn / hub (VPS) ==="
/usr/bin/ssh -i ~/.ssh/id_ed25519 root@signalling-nexartc.cn \
  'journalctl -u nexartc-hub -u nexartc-coturn --since "10 min ago" --no-pager | tail -40'
```

---

## 9. 产物与路径速查

| 组件 | 编译产物（开发机） | 部署目标 |
|------|-------------------|----------|
| device 页 | `nexartc-cloudPhoneAccess-web/out/device/` | VPS `/opt/nexartc/hub/device/` |
| Hub | `nexartc-cloudPhoneAccess-web/out/server/main.js` | VPS `/opt/nexartc/hub/server/` |
| coturn 工具 | `coturn-4.13.0/build-local/bin/` | 开发机测试；VPS 用 apt + conf |
| coturn 配置 | `test/turn/vps/turnserver.conf` | VPS `/etc/nexartc/turnserver.conf` |
| CAE APK | `.../output/server/CaeServer-root-debug.apk` | `adb install` → 真机 |
| CAE 配置 | assets 模板 + 真机 ini | `/data/user/0/.../files/cae/config/CaeConfig.ini` |
| 密钥 | `/tmp/nexartc-vps-deploy/env` | VPS `/etc/nexartc/hub.env` |

---

## 10. 相关脚本索引

| 脚本 | 作用 |
|------|------|
| `test/turn/deploy_vps.sh` | 构建 web + 部署 Hub/device/coturn 配置 + P0/P3 验证 |
| `test/turn/build_coturn_local.sh` | 本地编译 coturn 与 turnutils |
| `test/turn/vps/*.service` | systemd 单元模板 |
| `nexartc-cloudPhoneAccess-web/build_all.sh` | device + Hub → `out/` |
| `nexartc-cloud-phone-access-engine/gradlew assembleRootDebug` | CAE root APK |
| `nexartc-cloud-phone-access-engine/scripts/build_cae.sh` | CAE 传统打包 |

---

## 11. 变更记录

| 日期 | 说明 |
|------|------|
| 2026-07-20 | 首版安装部署手册：coturn / Hub / device / CAE；默认 host ICE；日志默认全开说明 |

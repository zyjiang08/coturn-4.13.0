# TURN REST API 与信令服务器集成设计

本文档描述如何用信令服务器动态生成 TURN 凭据，供 WebRTC 客户端向 coturn 发起请求并完成鉴权。该方案对应 coturn 的 **TURN REST API**（也称 secret-based timed authentication）。

## 1. 整体架构

```mermaid
sequenceDiagram
    participant Browser as 浏览器/WebRTC客户端
    participant Signal as 信令服务器
    participant Turn as coturn

    Browser->>Signal: 请求加入房间 / 获取 ICE 配置
    Note over Signal: 用 shared secret 生成<br/>临时 username + password
    Signal-->>Browser: iceServers (含 TURN 凭据)
    Browser->>Turn: STUN/TURN Allocate (401 挑战)
    Turn-->>Browser: 401 + realm + nonce
    Browser->>Turn: 带 MESSAGE-INTEGRITY 的认证请求
    Note over Turn: 解析 username 中的过期时间<br/>HMAC 校验 password
    Turn-->>Browser: Allocate 成功，分配 relay 地址
```

### 角色分工

| 组件 | 职责 |
|------|------|
| 信令服务器 | 持有 **shared secret**（绝不下发给客户端），按需生成临时 `username` / `credential` |
| coturn | 持有相同 secret，校验凭据是否合法、是否过期 |
| 客户端 | 只拿到临时凭据，按 RFC 5389 长期凭据机制完成 STUN/TURN 认证 |

## 2. 凭据生成算法

coturn 配置注释（`docker/coturn/turnserver.conf`）：

```
usercombo -> "timestamp:userid"
turn user -> usercombo
turn password -> base64(hmac(secret key, usercombo))
```

### 2.1 Username 格式

```
temporary_username = "<过期时间戳>" + ":" + "<用户ID>"
```

- **分隔符**默认是 `:`，可用 `rest-api-separator` 修改。
- **时间戳含义（重要）**：coturn 实现里这是**过期时间**（`now + TTL`），不是创建时间。
- **生产上建议使用 `expiry:userId`**：`get_rest_api_timestamp()` 虽然兼容 `userId:expiry`，但只有当 `userId` 含非数字字符时才不会歧义；纯数字 `userId` 会被误判成前置时间戳。
- **用户 ID 可选**：没有业务 ID 时，可以只用时间戳，例如 `"1735689600"`；但这种写法更适合测试，生产上不利于审计和按用户配额。

uclient 测试代码（`src/apps/uclient/mainuclient.c`）：

```c
const unsigned long exp_time = 3600 * 24; /* one day */
snprintf(new_uname, sizeof(new_uname), "%lu%c%s",
         (unsigned long)time(NULL) + exp_time, rest_api_separator, (char *)g_uname);
```

### 2.2 Password 生成

```
password = base64( HMAC-SHA1(key=shared_secret, message=temporary_username) )
```

- HMAC 算法：**SHA-1**（`SHATYPE_DEFAULT`）。
- 输出做 **Base64** 编码（不是 hex）。

### 2.3 信令服务器示例（Node.js）

```javascript
const crypto = require('crypto');

function generateTurnCredentials(userId, secret, ttlSeconds = 3600) {
  const expiry = Math.floor(Date.now() / 1000) + ttlSeconds;
  const username = userId ? `${expiry}:${userId}` : `${expiry}`;

  const hmac = crypto
    .createHmac('sha1', secret)
    .update(username)
    .digest();

  const credential = hmac.toString('base64');

  return { username, credential, ttl: ttlSeconds, expiry };
}

function getIceServers(turnHost, turnPort = 3478) {
  const { username, credential } = generateTurnCredentials('user-123', 'your-shared-secret', 3600);

  return {
    iceServers: [
      { urls: 'stun:' + turnHost + ':' + turnPort },
      {
        urls: 'turn:' + turnHost + ':' + turnPort,
        username,
        credential,
      },
      {
        urls: 'turns:' + turnHost + ':5349',
        username,
        credential,
      },
    ],
  };
}
```

### 2.4 Python 示例

```python
import hmac
import hashlib
import base64
import time

def generate_turn_credentials(user_id: str, secret: str, ttl: int = 3600) -> dict:
    expiry = int(time.time()) + ttl
    username = f"{expiry}:{user_id}" if user_id else str(expiry)

    digest = hmac.new(
        secret.encode('utf-8'),
        username.encode('utf-8'),
        hashlib.sha1
    ).digest()
    credential = base64.b64encode(digest).decode('utf-8')

    return {"username": username, "credential": credential, "ttl": ttl, "expiry": expiry}
```

## 3. coturn 服务端校验流程

客户端发来带 `USERNAME` 的 STUN/TURN 请求后，核心逻辑在 `src/apps/relay/userdb.c` 的 `get_user_key()`：

1. 从 username 解析时间戳（`get_rest_api_timestamp`）。
2. 判断 `timestamp >= now`（`!turn_time_before(ts, ctime)`），过期则拒绝。
3. 加载所有 shared secret（支持密钥轮换），逐个计算：
   - `HMAC-SHA1(key=secret, message=username)` → Base64 → 作为 password。
4. 按 RFC 5389 长期凭据机制校验 `MESSAGE-INTEGRITY`：
   - `key = MD5(username : realm : password)`。
   - 对比请求中的完整性校验值。
5. 认证成功后，把派生出的 `hmackey` / `pwd` 缓存到当前 session；后续同一会话中的请求通常直接用缓存校验，不会再次解析 username 时间戳。

时间戳解析（`get_rest_api_timestamp`）兼容两种格式，但生产建议只使用第一种：

- `1735689600:userId` — 时间戳在前（推荐）。
- `userId:1735689600` — 若冒号前不全是数字，则时间戳在后。

启用 `--use-auth-secret` 时，coturn 自动设置长期凭据模式（`src/apps/relay/mainrelay.c`）：

```c
turn_params.use_auth_secret_with_timestamp = 1;
turn_params.ct = TURN_CREDENTIALS_LONG_TERM;
use_lt_credentials = 1;
```

## 4. coturn 配置方法

### 4.1 最小配置（静态 secret）

`/etc/turnserver.conf`：

```ini
listening-port=3478
tls-listening-port=5349

use-auth-secret
static-auth-secret=your-very-long-random-secret

realm=yourcompany.com

listening-ip=0.0.0.0
relay-ip=<本机用于中继的IP>

# 若 coturn 部署在 NAT 后，还需要：
# external-ip=<公网IP>
# 或 external-ip=<公网IP>/<内网IP>

min-port=49152
max-port=65535

fingerprint
no-multicast-peers
no-cli
```

命令行等价写法：

```bash
turnserver \
  --use-auth-secret \
  --static-auth-secret=your-very-long-random-secret \
  --realm=yourcompany.com \
  -L 0.0.0.0 \
  -E <relay-ip> \
  --min-port=49152 --max-port=65535 \
  -f
```

`use-auth-secret` / `static-auth-secret` 会自动启用长期凭据模式；不必再显式写 `lt-cred-mech`。如果 coturn 部署在 NAT 后，再补 `--external-ip=<public-ip[/private-ip]>`，否则客户端拿到的 `XOR-RELAYED-ADDRESS` 可能不可达。

### 4.2 生产配置（数据库管理 secret，支持轮换）

```ini
use-auth-secret
userdb=/var/db/turndb
# 或 PostgreSQL/MySQL/Redis：
# psql-userdb="host=localhost dbname=coturn user=coturn password=xxx"
# redis-userdb="ip=127.0.0.1 dbname=0 password=xxx"
```

数据库表 `turn_secret`：

```sql
CREATE TABLE turn_secret (
    realm varchar(127) default '',
    value varchar(256),
    primary key (realm,value)
);
```

用 `turnadmin` 管理 secret：

```bash
turnadmin -s mysecret -r yourcompany.com -b /var/db/turndb
turnadmin -S -r yourcompany.com -b /var/db/turndb
turnadmin -X oldsecret -r yourcompany.com -b /var/db/turndb
```

coturn 支持**多个 secret 并存**，静态 secret 和数据库 secret 都会被逐个尝试，便于无缝轮换。

### 4.3 鉴权模式互斥

`use-auth-secret` 走 shared-secret 校验路径；静态/数据库长期用户（`-u user:pass` 或 DB `turnusers_lt`）走另一套校验路径。两种方式不应混用；若同时配置，**shared secret 会覆盖**静态用户密码校验。

| 方式 | 配置 | 适用场景 |
|------|------|----------|
| **TURN REST API**（推荐） | `use-auth-secret` + secret（无需额外 `lt-cred-mech`） | WebRTC、凭据需限时、信令服务器签发 |
| 静态 lt-cred-mech | `lt-cred-mech` + `-u user:pass` 或 DB `turnusers_lt` | 测试、固定客户端、内部服务 |

## 5. 有效期设置

有效期由多层机制叠加控制。

### 5.1 凭据有效期（信令服务器控制，最主要）

```
expiry = now + TTL
username = expiry + ":" + userId
```

| TTL 建议 | 场景 |
|----------|------|
| 300–600s | 高安全场景 |
| 3600s（1h） | 常规 WebRTC |
| 86400s（24h） | 长会话（uclient 默认值） |

coturn 校验条件：`timestamp >= now`，过期即拒绝新的凭据校验 / 新会话认证；这**不是**对已建立 allocation 的强制断开信号。

### 5.2 TURN Allocation 生命周期（coturn 控制）

```ini
max-allocate-lifetime=3600   # 默认 3600s
max-allocate-timeout=60
channel-lifetime=600
permission-lifetime=300
```

Allocate / Refresh 中的 lifetime 由 `stun_adjust_allocate_lifetime()` 计算，受 `max-allocate-lifetime` 和 `max_session_time_auth` 共同影响。

REST API（secret-based）模式下，coturn **不会**把凭据剩余时间写入 `max_session_time_auth`；该字段主要由 OAuth 流程设置。因此：

- 新建 Allocate 或每次 Refresh 的**单次 lifetime** 仍受 `max-allocate-lifetime` 限制。
- 一旦认证成功，coturn 会把派生出的 `hmackey` / `pwd` 缓存到当前 session。
- 后续同一会话里的 Refresh / ChannelBind / CreatePermission 等请求，通常直接用缓存做完整性校验，不会再次解析 username 中的时间戳。

因此，username 里的过期时间主要限制的是**新的认证 / 新的会话建立**，不会在 TTL 到点时自动切断一个已经建立并持续刷新的 allocation。

**实际单次 allocation lifetime <= max-allocate-lifetime；实际总会话时长取决于客户端是否持续 Refresh，以及服务端是否重启、回收或主动清理，而不等于 `min(凭据 TTL, max-allocate-lifetime)`。**

如果你的目标是“会话到点必须失效”，不能只依赖 REST API 的时间戳；还需要业务层停止续期 / 主动清理，或使用能显式限制 `max_session_time_auth` 的认证方案（如 OAuth）。

### 5.3 Nonce 有效期

```ini
stale-nonce=600   # 默认 600s，过期后返回 438
```

### 5.4 配额限制

```ini
user-quota=10
total-quota=1200
max-bps=3000000
```

配额按**真实用户名**统计（`get_real_username` 会去掉时间戳前缀），同一 `userId` 无论时间戳如何变化，都计入同一配额。

如果 username 只有时间戳，那么每次签发出来的值都可能成为新的 quota key，无法稳定按真实用户限额。

## 6. 信令服务器 API 设计建议

### 6.1 roomId 还是 userId？

**结论：接口鉴权用 `roomId`（或从 token 推导），TURN username 里嵌入 `userId`。两者职责不同，不要混为一个参数。**

| 参数 | 用途 | 放在哪里 |
|------|------|----------|
| `roomId` | 业务鉴权：确认用户有权进入该房间、有权获取 TURN 凭据 | 接口 query / path，例如 `?roomId=abc123` |
| `userId` | TURN 身份标识：审计、按用户配额（coturn `user-quota`） | 从登录 token 解析，写入 `username = expiry:userId` |

推荐流程：

```
客户端请求: GET /api/v1/turn-credentials?roomId=abc123
            Authorization: Bearer <token>

信令服务器:
  1. 从 token 解析出 userId（例如 "user-42"）
  2. 校验该 userId 是否是 roomId=abc123 的成员
  3. 校验通过后，用 userId（不是 roomId）生成 TURN 凭据：
     username = "<expiry>:user-42"
     credential = base64(hmac-sha1(secret, username))
  4. 返回 iceServers
```

**为什么不把 roomId 写进 TURN username？**

- coturn 的 `user-quota` 按 `get_real_username()` 统计，即 username 中 `:` 后面的部分。
- 若写 `expiry:room-abc`，同一房间所有用户共用一个 quota key，无法按人限流。
- 若写 `expiry:room-abc:user-42` 可以，但不如直接用稳定 `userId` 简洁。
- `roomId` 是会话级概念，适合放在信令鉴权层；`userId` 是用户级概念，适合放进 TURN 凭据。

**接口是否还需要传 userId？**

通常**不需要**客户端显式传 `userId`，应从 token 解析，防止伪造。只有 token 不可信或调试阶段，才考虑让客户端传 `userId` 并由服务端校验一致性。

### 6.2 从 `Authorization: Bearer <token>` 获取 userId

`Bearer <token>` 本身只是一种传递方式；**如何取出 userId 取决于你签发 token 时用的认证方案**。常见有两类：

#### 类型一：JWT（最常见）

登录时服务端签发 JWT，payload 里放入用户 ID。TURN 凭据接口收到请求后，**验签 + 解析 payload** 即可。

JWT payload 示例：

```json
{
  "sub": "user-42",
  "userId": "user-42",
  "iat": 1735600000,
  "exp": 1735686400
}
```

`userId` 通常放在 `sub`（Subject）或自定义字段 `userId` / `uid` 中，取决于你的登录接口约定。

Node.js（`jsonwebtoken`）完整示例：

```javascript
const jwt = require('jsonwebtoken');

const JWT_SECRET = process.env.JWT_SECRET;  // 与登录签发时一致

/**
 * Express 中间件：从 Authorization: Bearer <token> 解析用户身份
 */
function authenticate(req, res, next) {
  const authHeader = req.headers.authorization || '';
  const match = authHeader.match(/^Bearer\s+(.+)$/i);
  if (!match) {
    return res.status(401).json({ error: 'missing bearer token' });
  }

  const token = match[1];

  try {
    const payload = jwt.verify(token, JWT_SECRET);  // 验签 + 检查 exp
    // 按你签发时的字段名取值（只选一种，保持一致）
    req.user = {
      id: payload.sub || payload.userId || payload.uid,
    };
    if (!req.user.id) {
      return res.status(401).json({ error: 'token missing user id claim' });
    }
    next();
  } catch (err) {
    return res.status(401).json({ error: 'invalid or expired token' });
  }
}
```

Python（PyJWT）示例：

```python
import re
import jwt
from flask import request, jsonify, g

JWT_SECRET = os.environ["JWT_SECRET"]

def get_user_id_from_bearer():
    auth = request.headers.get("Authorization", "")
    m = re.match(r"Bearer\s+(.+)", auth, re.I)
    if not m:
        return None, ("missing bearer token", 401)

    token = m.group(1)
    try:
        payload = jwt.decode(token, JWT_SECRET, algorithms=["HS256"])
        user_id = payload.get("sub") or payload.get("userId") or payload.get("uid")
        if not user_id:
            return None, ("token missing user id claim", 401)
        return user_id, None
    except jwt.ExpiredSignatureError:
        return None, ("token expired", 401)
    except jwt.InvalidTokenError:
        return None, ("invalid token", 401)

@app.get("/api/v1/turn-credentials")
def turn_credentials():
    user_id, err = get_user_id_from_bearer()
    if err:
        return jsonify({"error": err[0]}), err[1]

    room_id = request.args.get("roomId")
    # ... 校验 room 成员、生成凭据 ...
```

JWT 调试：可在 [jwt.io](https://jwt.io) 粘贴 token 查看 payload 结构（**不要把生产 token 贴到公网**）。

#### 类型二：Opaque Token（不透明令牌）

token 只是一串随机字符串，本身不含用户信息。服务端需查存储：

```
Authorization: Bearer a8f3c2e1-...

服务端:
  session = redis.get("session:a8f3c2e1-...")
  userId = session.userId
```

Node.js 示例：

```javascript
async function authenticate(req, res, next) {
  const match = (req.headers.authorization || '').match(/^Bearer\s+(.+)$/i);
  if (!match) return res.status(401).json({ error: 'missing bearer token' });

  const session = await redis.get(`session:${match[1]}`);
  if (!session) return res.status(401).json({ error: 'invalid token' });

  const data = JSON.parse(session);
  req.user = { id: data.userId };
  next();
}
```

#### 提取流程小结

```
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
                    │
                    ▼
         1. 用正则取出 token 字符串
                    │
         ┌──────────┴──────────┐
         ▼                     ▼
    JWT（可解码）          Opaque（不可解码）
         │                     │
    jwt.verify()           查 Redis/DB
         │                     │
         ▼                     ▼
    读 sub / userId       读 session.userId
         │                     │
         └──────────┬──────────┘
                    ▼
              userId = "user-42"
                    │
                    ▼
         生成 TURN username = "expiry:user-42"
```

#### 安全要点

1. **必须服务端验签**（JWT）或**查会话存储**（Opaque），不要只 `jwt.decode()` 不验签。
2. **不要信任客户端传的 `userId` 参数**；只从已验证的 token 中取。
3. token 过期（`exp`）应返回 401，客户端需重新登录后再申请 TURN 凭据。
4. `userId` 建议用稳定业务 ID（数据库主键、UUID），不要用每次登录变化的值。

### 6.3 接口示例

```
GET /api/v1/turn-credentials?roomId=abc123
Authorization: Bearer <用户登录 token>
```

服务端伪代码：

```javascript
app.get('/api/v1/turn-credentials', authenticate, async (req, res) => {
  const { roomId } = req.query;
  const userId = req.user.id;  // 从 token 解析，不要信任客户端传的 userId

  if (!await isRoomMember(roomId, userId)) {
    return res.status(403).json({ error: 'not a room member' });
  }

  const { username, credential, ttl, expiry } =
    generateTurnCredentials(userId, process.env.TURN_SECRET, 3600);

  res.json({
    iceServers: buildIceServers(username, credential),
    ttl,
    expiry,
    // 可选：单独返回，方便客户端调试
    username,
    credential,
  });
});
```

响应：

```json
{
  "iceServers": [
    { "urls": "stun:turn.example.com:3478" },
    {
      "urls": [
        "turn:turn.example.com:3478?transport=udp",
        "turn:turn.example.com:3478?transport=tcp",
        "turns:turn.example.com:5349?transport=tcp"
      ],
      "username": "1735693200:user-42",
      "credential": "base64encodedhmac=="
    }
  ],
  "ttl": 3600,
  "expiry": 1735693200,
  "username": "1735693200:user-42",
  "credential": "base64encodedhmac=="
}
```

响应中 `username` / `credential` 既在 `iceServers` 的 TURN 条目里，也可在顶层重复一份。顶层字段便于客户端单独读取和日志排查；`iceServers` 才是直接传给 `RTCPeerConnection` 的值。

### 6.4 设计要点

1. **Secret 只存在服务端**：信令服务器和 coturn 各持一份，通过 KMS/环境变量注入，不进代码仓库。
2. **userId 绑定稳定业务身份**：优先放稳定的 `userId`，不要只用时间戳；这样审计、配额和问题定位才有意义。
3. **短 TTL 的作用边界**：短 TTL 主要用于缩小凭据泄露后的滥用窗口，并限制新的 Allocate / 新认证；它不会强制结束已建立的 relay 会话。
4. **如需硬性到期，业务层配合**：例如停止下发新凭据、在房间结束时主动清理会话，或改用能显式限制会话上限的认证方案。
5. **HTTPS 传输**：凭据经公网下发，接口必须 TLS。
6. **Secret 轮换**：数据库多 secret 并存，信令服务器用新 secret 签发，coturn 新旧都接受。

## 7. WebRTC 客户端配置 coturn 地址与凭据

WebRTC 通过 `RTCPeerConnection` 的 `iceServers` 配置 STUN/TURN。浏览器和原生 SDK 都使用同一套 [W3C WebRTC 规范](https://www.w3.org/TR/webrtc/#rtciceserver-dictionary) 定义的字段。

### 7.1 字段说明

| 字段 | 说明 |
|------|------|
| `urls` | STUN/TURN 服务器地址，字符串或字符串数组 |
| `username` | TURN 用户名（REST API 生成的临时 username） |
| `credential` | TURN 密码（REST API 生成的 Base64 HMAC，**不是** `password`） |

注意：

- WebRTC API 中密码字段名是 **`credential`**，不是 `password`。
- `credential` 的值就是信令服务器按第 2 节算法生成的 Base64 字符串。
- STUN 条目通常不需要 `username` / `credential`；只有 TURN 中继才需要。

### 7.2 URL 格式

coturn 常见 URL 写法：

| URL | 含义 |
|-----|------|
| `stun:turn.example.com:3478` | STUN Binding，获取 reflexive 地址 |
| `turn:turn.example.com:3478` | TURN over UDP（默认） |
| `turn:turn.example.com:3478?transport=udp` | 显式指定 UDP |
| `turn:turn.example.com:3478?transport=tcp` | TURN over TCP |
| `turns:turn.example.com:5349?transport=tcp` | TURN over TLS（coturn 默认 TLS 端口 5349） |

生产环境建议同时提供 UDP 和 TCP/TLS，以应对对称 NAT、企业防火墙等场景。

### 7.3 浏览器 JavaScript 完整示例

以下示例展示从信令接口**请求 → 解析 → 提取 username/credential → 设置 iceServers → 创建 PeerConnection** 的完整流程。

#### 7.3.1 第一步：请求信令接口

```javascript
/**
 * 向信令服务器申请 TURN 凭据。
 * @param {string} roomId  - 当前房间 ID，用于服务端鉴权
 * @param {string} token   - 用户登录 token，服务端从中解析 userId
 * @returns {Promise<object>} 含 iceServers / username / credential / expiry
 */
async function fetchTurnCredentials(roomId, token) {
  const url = `/api/v1/turn-credentials?roomId=${encodeURIComponent(roomId)}`;

  const res = await fetch(url, {
    method: 'GET',
    headers: {
      Authorization: `Bearer ${token}`,
      Accept: 'application/json',
    },
  });

  if (!res.ok) {
    const err = await res.json().catch(() => ({}));
    throw new Error(`TURN credentials failed: ${res.status} ${err.error || res.statusText}`);
  }

  return res.json();
}
```

要点：

- 请求参数传 **`roomId`**，用于服务端判断"该用户是否有权在此房间使用 TURN"。
- **`userId` 不传**，由服务端从 `Authorization` token 解析，避免客户端伪造身份。

#### 7.3.2 第二步：从响应中提取 username 和 credential

信令服务器返回的 JSON 中，凭据可能出现在两个位置：

```json
{
  "iceServers": [
    { "urls": "stun:..." },
    {
      "urls": ["turn:..."],
      "username": "1735693200:user-42",
      "credential": "x3bP9k2..."
    }
  ],
  "username": "1735693200:user-42",
  "credential": "x3bP9k2...",
  "expiry": 1735693200,
  "ttl": 3600
}
```

提取方式一（推荐）：直接使用 `iceServers`，无需手动拆字段：

```javascript
const data = await fetchTurnCredentials(roomId, token);
const iceServers = data.iceServers;  // 直接用于 RTCPeerConnection
```

提取方式二：需要单独读取 username / credential 时（日志、过期判断）：

```javascript
const data = await fetchTurnCredentials(roomId, token);

// 优先读顶层字段
let username = data.username;
let credential = data.credential;

// 若顶层没有，从 iceServers 里找带 username 的 TURN 条目
if (!username || !credential) {
  const turnEntry = data.iceServers.find((s) => s.username && s.credential);
  if (turnEntry) {
    username = turnEntry.username;
    credential = turnEntry.credential;
  }
}

if (!username || !credential) {
  throw new Error('response missing TURN username/credential');
}

console.log('TURN username:', username);       // "1735693200:user-42"
console.log('TURN credential:', credential);   // Base64 字符串，即 coturn 意义上的 password
console.log('expires at:', data.expiry);     // Unix 秒级时间戳
```

字段对应关系：

| 信令接口字段 | coturn 概念 | WebRTC API 字段 | 说明 |
|-------------|------------|----------------|------|
| `username` | TURN username | `username` | 格式 `expiry:userId` |
| `credential` | TURN password | `credential` | Base64(HMAC-SHA1)，**不是** shared secret |

浏览器 WebRTC API 中密码字段名固定为 `credential`；部分原生 SDK（如 Android）方法名是 `setPassword()`，传入的值仍是这个 `credential` 字符串。

#### 7.3.3 第三步：组装 iceServers 并设置到 PeerConnection

方式 A — 服务端已返回完整 `iceServers`（推荐，直接用）：

```javascript
const data = await fetchTurnCredentials(roomId, token);

const pc = new RTCPeerConnection({
  iceServers: data.iceServers,
});
```

方式 B — 客户端自行拼装（适合前端硬编码 TURN 地址、只从服务端拿凭据）：

```javascript
const data = await fetchTurnCredentials(roomId, token);
const { username, credential } = data;

const TURN_HOST = 'turn.example.com';

const iceServers = [
  // STUN：不需要 username / credential
  {
    urls: `stun:${TURN_HOST}:3478`,
  },
  // TURN：必须带 username + credential
  {
    urls: [
      `turn:${TURN_HOST}:3478?transport=udp`,
      `turn:${TURN_HOST}:3478?transport=tcp`,
      `turns:${TURN_HOST}:5349?transport=tcp`,
    ],
    username: username,      // 从信令响应提取
    credential: credential,    // 从信令响应提取，即 password
  },
];

const pc = new RTCPeerConnection({ iceServers });
```

#### 7.3.4 第四步：完整通话建立流程

```javascript
async function startCall(roomId, token, localStream) {
  // --- 1. 获取并设置 TURN 凭据 ---
  const turnData = await fetchTurnCredentials(roomId, token);

  const pc = new RTCPeerConnection({
    iceServers: turnData.iceServers,
    // iceTransportPolicy: 'relay',  // 调试时可强制走 TURN
  });

  // --- 2. 添加本地媒体轨道 ---
  localStream.getTracks().forEach((track) => {
    pc.addTrack(track, localStream);
  });

  // --- 3. 监听 ICE 候选（验证 TURN 是否生效）---
  pc.onicecandidate = (event) => {
    if (!event.candidate) {
      console.log('ICE gathering complete');
      return;
    }
    const c = event.candidate.candidate;
    if (c.includes('typ relay')) {
      console.log('TURN relay candidate OK:', c);
    } else if (c.includes('typ srflx')) {
      console.log('STUN srflx candidate:', c);
    } else if (c.includes('typ host')) {
      console.log('host candidate:', c);
    }
    // 通过信令通道发送给对端
    sendSignaling({ type: 'ice-candidate', candidate: event.candidate });
  };

  pc.oniceconnectionstatechange = () => {
    console.log('ICE state:', pc.iceConnectionState);
  };

  // --- 4. 创建 Offer 并开始协商 ---
  const offer = await pc.createOffer();
  await pc.setLocalDescription(offer);
  sendSignaling({ type: 'offer', sdp: offer });

  return { pc, turnData };
}

// 对端 Answer 回来后的处理
async function handleAnswer(pc, answer) {
  await pc.setRemoteDescription(new RTCSessionDescription(answer));
}

// 对端 ICE candidate 回来后的处理
async function handleRemoteCandidate(pc, candidate) {
  if (candidate) {
    await pc.addIceCandidate(new RTCIceCandidate(candidate));
  }
}
```

#### 7.3.5 本地硬编码（仅开发调试）

```javascript
// 仅用于本地联调；生产环境必须由信令服务器动态签发
const pc = new RTCPeerConnection({
  iceServers: [
    { urls: 'stun:127.0.0.1:3478' },
    {
      urls: 'turn:127.0.0.1:3478?transport=udp',
      username: '1735693200:test-user',       // expiry:userId
      credential: 'base64encodedhmac==',      // 等效于 password
    },
  ],
});
```

不要把 shared secret 写进前端；`username` 和 `credential` 必须由服务端生成。

#### 7.3.6 凭据过期后更新

`RTCPeerConnection` 创建后**不能**修改 `iceServers`。长会话需在过期前重新申请：

```javascript
async function refreshPeerConnection(oldPc, roomId, token) {
  const turnData = await fetchTurnCredentials(roomId, token);
  const newPc = new RTCPeerConnection({ iceServers: turnData.iceServers });

  // 迁移媒体轨道
  oldPc.getSenders().forEach((sender) => {
    if (sender.track) newPc.addTrack(sender.track);
  });

  oldPc.close();
  return { pc: newPc, turnData };
}

// 可在过期前主动检查
function isCredentialExpired(expiry) {
  return Math.floor(Date.now() / 1000) >= expiry;
}
```

常见策略：

1. **短会话**：TTL 覆盖整次通话（如 3600s），无需中途更新。
2. **长会话**：在 `expiry` 前 5 分钟触发 `refreshPeerConnection`，重建连接并重新协商。
3. **未建连就过期**：销毁旧 `pc`，重新 `fetchTurnCredentials` 后重建。

### 7.4 React Native / react-native-webrtc 示例

```javascript
import { RTCPeerConnection } from 'react-native-webrtc';

const iceServers = [
  { urls: 'stun:turn.example.com:3478' },
  {
    urls: 'turn:turn.example.com:3478?transport=udp',
    username: credentials.username,
    credential: credentials.credential,
  },
];

const pc = new RTCPeerConnection({ iceServers });
```

字段名与浏览器一致：`username` + `credential`。

### 7.5 Android（org.webrtc）示例

```kotlin
val iceServers = listOf(
    PeerConnection.IceServer.builder("stun:turn.example.com:3478").createIceServer(),
    PeerConnection.IceServer.builder("turn:turn.example.com:3478?transport=udp")
        .setUsername(credentials.username)
        .setPassword(credentials.credential)   // Android SDK 方法名是 setPassword
        .createIceServer(),
    PeerConnection.IceServer.builder("turns:turn.example.com:5349?transport=tcp")
        .setUsername(credentials.username)
        .setPassword(credentials.credential)
        .createIceServer(),
)

val rtcConfig = PeerConnection.RTCConfiguration(iceServers)
val peerConnection = factory.createPeerConnection(rtcConfig, observer)
```

Android `org.webrtc` 的 builder 方法名叫 `setPassword()`，但传入的值仍是 WebRTC 规范中的 `credential` 字符串（Base64 HMAC），不是 coturn 的 shared secret。

### 7.6 iOS（WebRTC.framework）示例

```swift
let iceServers: [RTCIceServer] = [
    RTCIceServer(urlStrings: ["stun:turn.example.com:3478"]),
    RTCIceServer(
        urlStrings: ["turn:turn.example.com:3478?transport=udp"],
        username: credentials.username,
        credential: credentials.credential
    ),
    RTCIceServer(
        urlStrings: ["turns:turn.example.com:5349?transport=tcp"],
        username: credentials.username,
        credential: credentials.credential
    ),
]

let config = RTCConfiguration()
config.iceServers = iceServers
let peerConnection = factory.peerConnection(with: config, constraints: constraints, delegate: self)
```

### 7.7 iceTransportPolicy 选项

| 值 | 行为 |
|----|------|
| `all`（默认） | 同时尝试 host、srflx（STUN）、relay（TURN）候选 |
| `relay` | 只使用 TURN 中继，跳过 host / STUN |

```javascript
const pc = new RTCPeerConnection({
  iceServers,
  iceTransportPolicy: 'relay',  // 强制 TURN，便于验证 coturn 配置
});
```

调试 coturn 时，可临时设为 `relay`，确认 TURN 凭据和连通性正常后再改回 `all`。

### 7.8 验证 TURN 是否生效

在浏览器控制台查看 ICE 候选：

```javascript
pc.onicecandidate = (event) => {
  if (event.candidate) {
    console.log(event.candidate.candidate);
    // 成功走 TURN 时会看到类似：
    // candidate:... typ relay raddr ... rport ... generation ...
  }
};
```

- `typ host` — 本地网卡地址
- `typ srflx` — STUN 反射地址
- `typ relay` — TURN 中继地址（说明 coturn 凭据和连通性正常）

也可使用 [Trickle ICE](https://webrtc.github.io/samples/src/content/peerconnection/trickle-ice/) 在线工具测试：填入 TURN URL、`username`、`credential`，观察是否产生 `relay` 候选。

### 7.9 常见配置错误

| 错误 | 现象 | 修正 |
|------|------|------|
| 使用 `password` 而非 `credential` | 浏览器报配置无效或 TURN 认证失败 | 改用 `credential` 字段 |
| 前端硬编码 shared secret | 安全风险，secret 泄露 | secret 只放信令服务器和 coturn |
| 只配 STUN 不配 TURN | 对称 NAT 下无法连通 | 同时配置 `turn:` / `turns:` 条目 |
| `username` 时间戳已过期 | Allocate 返回 401/403 | 重新向信令服务器申请凭据 |
| coturn 在 NAT 后未设 `external-ip` | 有 `relay` 候选但媒体不通 | coturn 配置 `--external-ip` |
| TLS 证书不受信任 | `turns:` 连接失败 | 使用正规 CA 证书，或开发环境导入自签证书 |

## 8. 完整认证时序（客户端视角）

REST API 只是替换了 username/password 的生成方式，之后走标准长期凭据流程：

1. 客户端发 Allocate → coturn 回 **401** + `realm` + `nonce`。
2. 客户端计算 `key = MD5(username : realm : password)`，在请求中加入 `USERNAME`、`REALM`、`NONCE`、`MESSAGE-INTEGRITY`。
3. coturn 调用 `get_user_key()` 校验。
4. 成功后分配 relay 地址。

## 9. 验证配置

```bash
# 启动 coturn（REST API 模式）
turnserver --use-auth-secret --static-auth-secret=logen \
  --realm=north.gov -f -L 127.0.0.1 -E 127.0.0.1

# 客户端用 -W 传入相同 secret，-u 传入用户 ID
turnutils_uclient -u ninefingers -W logen -e 127.0.0.1 127.0.0.1
```

uclient 的 `-W` 会自动按 REST API 规则生成 username/password（`src/apps/uclient/mainuclient.c`）。这里故意不写 `-a`；`--use-auth-secret` 已自动启用长期凭据模式。

## 10. 常见问题

**凭据生成后认证失败？**

- 检查 secret 是否一致。
- 检查 username 格式（时间戳必须是**未来**时间）。
- 如果你用了 `userId:expiry` 这种后置时间戳格式，确保 `userId` 含非数字字符；生产上更推荐 `expiry:userId`。
- 检查 HMAC 是否 SHA-1 + Base64。
- 检查 `rest-api-separator` 是否一致（默认 `:`）。

**时间戳用创建时间还是过期时间？**

coturn 实现要求 **过期时间**（`now + TTL`），校验条件是 `timestamp >= now`。

**凭据过期会立即断开现有 relay 吗？**

不会。REST API 时间戳主要在 `get_user_key()` 查凭据时检查。认证成功后，coturn 会把派生的 `hmackey` / `pwd` 缓存在 session 中；同一会话后续请求通常直接用缓存校验。因此，短 TTL 主要限制新的认证 / 新的 Allocate，不是强制截断已建立会话。

**需要 `-a` / `lt-cred-mech` 吗？**

不需要单独写。`use-auth-secret` / `static-auth-secret` 会自动启用长期凭据模式。显式再写 `-a` 只是冗余，且 coturn 会给出配置提醒 / 警告；建议只保留 shared-secret 相关配置。

**如何按域名分配不同 realm？**

用 `turn_origin_to_realm` 表，把 Web 页面的 `ORIGIN` 映射到不同 realm，各 realm 可有独立 secret 和配额。

**接口参数用 roomId 还是 userId？**

- 接口 query 传 **`roomId`**，用于鉴权（用户是否有权在该房间获取 TURN 凭据）。
- TURN `username` 里嵌入 **`userId`**（从 token 解析），用于 coturn 审计和按用户配额。
- 客户端不应自行传 `userId` 参数，防止伪造；由信令服务器从 token 提取。

**WebRTC 里 password 字段叫什么？**

浏览器用 `credential`；Android SDK 用 `setPassword()`，但传入的值是信令返回的 `credential` 字符串（Base64 HMAC），不是 coturn 的 shared secret。

## 11. 相关源码与文档

| 路径 | 说明 |
|------|------|
| `src/apps/relay/userdb.c` | `get_user_key()`、`get_rest_api_timestamp()` |
| `src/server/ns_turn_server.c` | 会话内认证缓存、Allocate / Refresh 校验流程 |
| `src/client/ns_turn_msg.c` | `stun_adjust_allocate_lifetime()` |
| `src/apps/uclient/mainuclient.c` | 客户端凭据生成参考实现 |
| `src/apps/relay/mainrelay.c` | `--use-auth-secret` 配置解析 |
| `docker/coturn/turnserver.conf` | 配置注释与示例 |
| `man/man1/turnserver.1` | TURN REST API 官方说明 |
| `man/man1/turnadmin.1` | `turnadmin` secret 管理命令 |
| `examples/scripts/restapi/` | REST API 示例脚本 |
| `turndb/schema.sql` | `turn_secret` 表结构 |

参考规范：

- [draft-uberti-behave-turn-rest-00](https://tools.ietf.org/html/draft-uberti-behave-turn-rest-00)
- [RFC 5389 §10.2 长期凭据机制](https://tools.ietf.org/html/rfc5389#section-10.2)

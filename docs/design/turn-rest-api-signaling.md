# TURN REST API 与信令服务器集成设计

本文档描述如何用信令服务器动态生成 TURN 凭据，供 WebRTC 客户端向 coturn 发起请求并完成鉴权。该方案对应 coturn 的 **TURN REST API**（也称 secret-based timed authentication）。第 12 节给出 **公网 coturn + 内网全锥型 NAT 服务端 + 移动端** 的组网概览；**第 13 节** 给出首选方案 **「公云信令 + 内网 Server 出站注册」** 的完整设计与分步实施指南（无需 frp/nginx/DDNS）。

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

### 1.2 两种部署方案

TURN REST API 的凭据算法是**对称的**：只要持有相同的 shared secret，服务端或客户端都能独立生成通过 coturn 校验的 `username` / `credential`。根据 secret 的分发范围，有两种部署方式：

| | 方案 A：信令服务器下发 | 方案 B：本地生成（预置 Shared Secret） |
|---|---|---|
| **secret 持有方** | 信令服务器 + coturn | 客户端 + 服务端（可选）+ coturn |
| **凭据生成位置** | 信令服务器 | 客户端或服务端本地 |
| **是否需要 TURN 凭据接口** | 需要 | 不需要 |
| **coturn 配置** | `use-auth-secret` | 相同 |
| **适用场景** | 浏览器 WebRTC（推荐） | 原生 App、内网工具、测试联调 |
| **安全性** | secret 不暴露给客户端 | secret 内置在客户端，可被逆向提取 |

- **方案 A** 见第 7、8 节（信令 API + WebRTC 配置）。
- **方案 B** 见第 3 节（本地生成完整说明）。

两种方案共用第 2 节的凭据算法和第 5 节的 coturn 配置；coturn 校验逻辑完全相同，不区分凭据由谁生成。

## 2. 凭据生成算法（两种方案共用）

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

### 2.5 C 示例（OpenSSL，与 coturn 一致）

coturn 内部使用 `HMAC(EVP_sha1(), ...)` + `base64_encode()`（见 `src/apps/uclient/mainuclient.c`、`src/apps/relay/userdb.c`）。独立 C 程序可同样依赖 OpenSSL：

```c
#include <openssl/evp.h>
#include <openssl/hmac.h>
#include <openssl/bio.h>
#include <openssl/buffer.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>

#define REST_API_SEPARATOR ':'

/* Base64 编码，输出需 free() */
static char *base64_encode(const unsigned char *data, size_t len) {
  BIO *b64 = BIO_new(BIO_f_base64());
  BIO *mem = BIO_new(BIO_s_mem());
  if (!b64 || !mem) {
    return NULL;
  }
  mem = BIO_push(b64, mem);
  BIO_set_flags(mem, BIO_FLAGS_BASE64_NO_NL);
  if (BIO_write(mem, data, (int)len) <= 0) {
    BIO_free_all(mem);
    return NULL;
  }
  if (BIO_flush(mem) != 1) {
    BIO_free_all(mem);
    return NULL;
  }
  BUF_MEM *buf = NULL;
  BIO_get_mem_ptr(mem, &buf);
  char *out = malloc(buf->length + 1);
  if (!out) {
    BIO_free_all(mem);
    return NULL;
  }
  memcpy(out, buf->data, buf->length);
  out[buf->length] = '\0';
  BIO_free_all(mem);
  return out;
}

/*
 * 生成 TURN REST API 凭据。
 * secret  : 与 coturn static-auth-secret 相同
 * user_id : 业务用户 ID，可为 NULL（仅时间戳）
 * ttl     : 有效秒数
 * 返回 0 成功，-1 失败
 */
int generate_turn_credentials(const char *user_id, const char *secret, int ttl, char *username,
                              size_t username_sz, char *credential, size_t credential_sz) {
  const time_t expiry = time(NULL) + ttl;

  if (user_id && user_id[0]) {
    snprintf(username, username_sz, "%ld%c%s", (long)expiry, REST_API_SEPARATOR, user_id);
  } else {
    snprintf(username, username_sz, "%ld", (long)expiry);
  }

  unsigned char hmac[EVP_MAX_MD_SIZE];
  unsigned int hmac_len = 0;

  if (!HMAC(EVP_sha1(), secret, (int)strlen(secret), (unsigned char *)username, strlen(username), hmac,
            &hmac_len)) {
    return -1;
  }

  char *b64 = base64_encode(hmac, hmac_len);
  if (!b64) {
    return -1;
  }
  strncpy(credential, b64, credential_sz - 1);
  credential[credential_sz - 1] = '\0';
  free(b64);
  return 0;
}

int main(void) {
  const char *secret = "logen"; /* 与 coturn --static-auth-secret 一致 */
  char username[1024] = {0};
  char credential[256] = {0};

  if (generate_turn_credentials("user-42", secret, 3600, username, sizeof(username), credential,
                                sizeof(credential)) != 0) {
    fprintf(stderr, "generate failed\n");
    return 1;
  }

  printf("username  : %s\n", username);    /* 例: 1735693200:user-42 */
  printf("credential: %s\n", credential);  /* Base64(HMAC-SHA1) */
  return 0;
}
```

编译：

```bash
gcc -o turn_cred turn_cred.c -lssl -lcrypto
```

### 2.6 C++ 示例（OpenSSL）

```cpp
#include <openssl/evp.h>
#include <openssl/hmac.h>
#include <openssl/bio.h>
#include <openssl/buffer.h>

#include <cstdio>
#include <cstring>
#include <ctime>
#include <string>

static std::string base64_encode(const unsigned char *data, size_t len) {
  BIO *b64 = BIO_new(BIO_f_base64());
  BIO *mem = BIO_new(BIO_s_mem());
  mem = BIO_push(b64, mem);
  BIO_set_flags(mem, BIO_FLAGS_BASE64_NO_NL);
  BIO_write(mem, data, static_cast<int>(len));
  BIO_flush(mem);
  BUF_MEM *buf = nullptr;
  BIO_get_mem_ptr(mem, &buf);
  std::string out(buf->data, buf->length);
  BIO_free_all(mem);
  return out;
}

struct TurnCredentials {
  std::string username;
  std::string credential;
  long expiry;
  int ttl;
};

TurnCredentials generate_turn_credentials(const std::string &user_id, const std::string &secret, int ttl = 3600) {
  const long expiry = static_cast<long>(std::time(nullptr)) + ttl;

  TurnCredentials cred;
  cred.ttl = ttl;
  cred.expiry = expiry;

  if (!user_id.empty()) {
    cred.username = std::to_string(expiry) + ":" + user_id;
  } else {
    cred.username = std::to_string(expiry);
  }

  unsigned char hmac[EVP_MAX_MD_SIZE];
  unsigned int hmac_len = 0;

  HMAC(EVP_sha1(), secret.data(), static_cast<int>(secret.size()),
       reinterpret_cast<const unsigned char *>(cred.username.data()), cred.username.size(), hmac, &hmac_len);

  cred.credential = base64_encode(hmac, hmac_len);
  return cred;
}

int main() {
  const std::string secret = "logen";
  TurnCredentials cred = generate_turn_credentials("user-42", secret, 3600);

  std::printf("username  : %s\n", cred.username.c_str());
  std::printf("credential: %s\n", cred.credential.c_str());

  // 传给 WebRTC / libwebrtc：
  // iceServer.username = cred.username;
  // iceServer.credential = cred.credential;
  return 0;
}
```

编译：

```bash
g++ -std=c++17 -o turn_cred_cpp turn_cred.cpp -lssl -lcrypto
```

### 2.7 四种语言对照

| 步骤 | 算法 |
|------|------|
| 1 | `expiry = now + ttl` |
| 2 | `username = expiry + ":" + userId`（userId 可省略） |
| 3 | `credential = base64( HMAC-SHA1(key=secret, message=username) )` |

```javascript
// JavaScript（Node.js）— 与 2.3 节相同，一行对照
const username = `${Math.floor(Date.now()/1000) + ttl}:${userId}`;
const credential = crypto.createHmac('sha1', secret).update(username).digest('base64');
```

```python
# Python — 与 2.4 节相同
username = f"{int(time.time()) + ttl}:{user_id}"
credential = base64.b64encode(hmac.new(secret.encode(), username.encode(), 'sha1').digest()).decode()
```

```c
// C — 与 2.5 节相同
snprintf(username, sz, "%ld:%s", (long)(time(NULL) + ttl), user_id);
HMAC(EVP_sha1(), secret, strlen(secret), (unsigned char *)username, strlen(username), hmac, &hmac_len);
// credential = base64_encode(hmac, hmac_len)
```

验证有两种方式：

- **验证算法是否实现正确**：使用 `turnutils_uclient -u <userId> -W <secret>`，让 `uclient` 自己按同一算法本地生成凭据并连接 coturn。
- **验证你外部代码生成出来的那组结果**：把完整 `username` / `credential` 直接传给 `uclient`，例如 `turnutils_uclient -u <完整username> -w <credential> ...`。

两种方式都能连通时，可基本确认外部实现与 coturn 的 REST API 规则一致。

## 3. 方案 B：本地 Shared Secret 生成凭据

方案 B 中，**客户端或服务端**在本地用与 coturn 相同的 shared secret 和算法（第 2 节）生成凭据，无需调用信令接口。coturn 只校验结果是否正确，不关心凭据由哪一端生成。

### 3.1 架构

```mermaid
sequenceDiagram
    participant App as 客户端/服务端应用
    participant Turn as coturn

    Note over App: 本地持有 shared secret
    Note over App: username = expiry:userId<br/>credential = base64(hmac(secret, username))
    App->>Turn: STUN/TURN Allocate (401 挑战)
    Turn-->>App: 401 + realm + nonce
    App->>Turn: 带 MESSAGE-INTEGRITY 的认证请求
    Note over Turn: 用同一 secret 重算 HMAC 校验
    Turn-->>App: Allocate 成功
```

三方关系：

| 组件 | 持有 secret | 生成凭据 |
|------|------------|---------|
| coturn | 是（`static-auth-secret` 或 DB） | 否，只校验 |
| 服务端（可选） | 是 | 可按需本地生成 |
| 客户端 | 是（内置或安全存储） | 按需本地生成 |

### 3.2 coturn 配置（与方案 A 相同）

```ini
use-auth-secret
static-auth-secret=your-shared-secret
realm=yourcompany.com
listening-port=3478
fingerprint
```

三方必须使用**完全相同**的 `static-auth-secret` 字符串（或通过 DB `turn_secret` 表配置的值）。

### 3.3 服务端本地生成

适用：批量任务、后台服务、无浏览器的媒体网关等。

```javascript
const crypto = require('crypto');

const TURN_SECRET = process.env.TURN_SECRET;  // 与 coturn 一致
const TURN_HOST = 'turn.example.com';

function buildIceServers(userId, ttlSeconds = 3600) {
  const expiry = Math.floor(Date.now() / 1000) + ttlSeconds;
  const username = `${expiry}:${userId}`;

  const credential = crypto
    .createHmac('sha1', TURN_SECRET)
    .update(username)
    .digest('base64');

  return [
    { urls: `stun:${TURN_HOST}:3478` },
    {
      urls: [
        `turn:${TURN_HOST}:3478?transport=udp`,
        `turn:${TURN_HOST}:3478?transport=tcp`,
      ],
      username,
      credential,
    },
  ];
}

// 服务端直接生成，无需 HTTP 接口
const iceServers = buildIceServers('device-001', 3600);
```

### 3.4 客户端本地生成

适用：原生 App（Android / iOS / Electron / 桌面客户端）、内网嵌入式设备、C/C++ 媒体网关、开发联调。

C/C++ 完整实现见 **第 2.5、2.6 节**；JavaScript 见 **第 2.3、2.7 节**。

#### JavaScript（Electron / Node.js 风格）

```javascript
import crypto from 'crypto';  // Node/Electron；浏览器需 Web Crypto 见下

const TURN_SECRET = 'your-shared-secret';  // 与 coturn 相同，内网 App 可内置
const TURN_HOST = 'turn.example.com';

function generateTurnCredentials(userId, ttlSeconds = 3600) {
  const expiry = Math.floor(Date.now() / 1000) + ttlSeconds;
  const username = `${expiry}:${userId}`;

  const credential = crypto
    .createHmac('sha1', TURN_SECRET)
    .update(username)
    .digest('base64');

  return { username, credential, expiry };
}

// 创建 PeerConnection 前本地生成
const { username, credential } = generateTurnCredentials('user-42', 3600);

const pc = new RTCPeerConnection({
  iceServers: [
    { urls: `stun:${TURN_HOST}:3478` },
    {
      urls: `turn:${TURN_HOST}:3478?transport=udp`,
      username,
      credential,
    },
  ],
});
```

若是 **React Native**，通常不能直接使用 Node 内置 `crypto`；需要额外的 HMAC/crypto 库，或更推荐直接使用方案 A，由服务端下发 `username` / `credential`。

浏览器环境（Web Crypto API，仅适用于可接受 secret 暴露的内网页面）：

```javascript
async function generateTurnCredentialsBrowser(userId, secret, ttlSeconds = 3600) {
  const expiry = Math.floor(Date.now() / 1000) + ttlSeconds;
  const username = `${expiry}:${userId}`;

  const enc = new TextEncoder();
  const key = await crypto.subtle.importKey(
    'raw', enc.encode(secret), { name: 'HMAC', hash: 'SHA-1' }, false, ['sign']
  );
  const sig = await crypto.subtle.sign('HMAC', key, enc.encode(username));
  const credential = btoa(String.fromCharCode(...new Uint8Array(sig)));

  return { username, credential, expiry };
}
```

#### C / C++（原生客户端、嵌入式、媒体网关）

见第 **2.5**（C）、**2.6**（C++）节。生成后将 `username` / `credential` 传入 libwebrtc 或 Pion 等 SDK 的 ICE 配置即可：

```c
char username[1024], credential[256];
generate_turn_credentials("user-42", "logen", 3600, username, sizeof(username),
                          credential, sizeof(credential));
/* 填入 WebRTC IceServer.username / .credential */
```

#### Kotlin（Android）

```kotlin
import javax.crypto.Mac
import javax.crypto.spec.SecretKeySpec
import android.util.Base64

fun generateTurnCredentials(
    userId: String,
    secret: String,
    ttlSeconds: Long = 3600
): Pair<String, String> {
    val expiry = System.currentTimeMillis() / 1000 + ttlSeconds
    val username = "$expiry:$userId"

    val mac = Mac.getInstance("HmacSHA1")
    mac.init(SecretKeySpec(secret.toByteArray(), "HmacSHA1"))
    val credential = Base64.encodeToString(mac.doFinal(username.toByteArray()), Base64.NO_WRAP)

    return username to credential
}

// 使用
val (username, credential) = generateTurnCredentials("user-42", TURN_SECRET)
val iceServer = PeerConnection.IceServer.builder("turn:turn.example.com:3478?transport=udp")
    .setUsername(username)
    .setPassword(credential)
    .createIceServer()
```

#### Swift（iOS）

```swift
import CommonCrypto
import Foundation

func generateTurnCredentials(userId: String, secret: String, ttl: Int = 3600) -> (username: String, credential: String) {
    let expiry = Int(Date().timeIntervalSince1970) + ttl
    let username = "\(expiry):\(userId)"

    let key = secret.data(using: .utf8)!
    let msg = username.data(using: .utf8)!
    var hmac = [UInt8](repeating: 0, count: Int(CC_SHA1_DIGEST_LENGTH))
    key.withUnsafeBytes { k in
        msg.withUnsafeBytes { m in
            CCHmac(CCHmacAlgorithm(kCCHmacAlgSHA1), k.baseAddress, key.count,
                   m.baseAddress, msg.count, &hmac)
        }
    }
    let credential = Data(hmac).base64EncodedString()
    return (username, credential)
}
```

### 3.5 coturn 自带工具验证（uclient -W）

coturn 的 `turnutils_uclient` 就是方案 B 的参考实现：客户端本地用 `-W` 传入 secret，工具内部按 REST API 算法生成 username/password 再连接 coturn。

```bash
# 终端 1：启动 coturn
turnserver --use-auth-secret --static-auth-secret=logen \
  --realm=north.gov -f -L 127.0.0.1 -E 127.0.0.1

# 终端 2：客户端本地生成凭据并连接（-W 即 shared secret，-u 为 userId）
turnutils_uclient -u ninefingers -W logen -e 127.0.0.1 127.0.0.1
```

对应源码（`src/apps/uclient/mainuclient.c`）：收到 `-W logen` 后，在本地计算 `username = time(NULL)+86400 : ninefingers`，再 `HMAC-SHA1 → Base64` 得到 password，无需任何信令服务器。

### 3.6 服务端与客户端各自生成的协同

两端可**独立**生成凭据，只要满足：

1. `static-auth-secret` 相同；
2. `username` 中时间戳未过期（`expiry >= now`）；
3. `credential = base64(hmac-sha1(secret, username))` 算法一致；
4. coturn `realm` 与客户端 STUN 认证时使用的 realm 一致。

示例：服务端为设备 A 生成凭据写入配置；设备 A 客户端也可在运行时自行重新生成——互不影响，各自独立通过 coturn 校验。

```
服务端生成: username=1735693200:device-A  → coturn 校验通过
客户端生成: username=1735696800:device-A  → coturn 校验通过（时间戳不同但各自合法）
```

### 3.7 安全边界与选型建议

| 场景 | 推荐方案 |
|------|---------|
| 公网浏览器 WebRTC | **方案 A**（secret 不下发） |
| 企业内网原生 App | 方案 B 可接受，secret 放安全存储 |
| 开发/联调 | 方案 B + `turnutils_uclient -W` |
| IoT / 嵌入式 | 方案 B，secret 编译进固件（评估泄露风险） |

**方案 B 的风险**：客户端内置 secret 可被逆向，攻击者可自行生成合法凭据。缓解措施：

- 缩短 TTL（如 300s），减小滥用窗口；
- 配置 coturn `user-quota` / `total-quota` / `max-bps` 限流；
- 用 `denied-peer-ip` / `allowed-peer-ip` 限制中继目标；
- 内网部署 + 防火墙限制 coturn 入口。

## 4. coturn 服务端校验流程

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

## 5. coturn 配置方法

### 5.1 最小配置（静态 secret）

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

### 5.2 生产配置（数据库管理 secret，支持轮换）

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

### 5.3 鉴权模式互斥

`use-auth-secret` 走 shared-secret 校验路径；静态/数据库长期用户（`-u user:pass` 或 DB `turnusers_lt`）走另一套校验路径。两种方式不应混用；若同时配置，**shared secret 会覆盖**静态用户密码校验。

| 方式 | 配置 | 适用场景 |
|------|------|----------|
| **TURN REST API**（推荐） | `use-auth-secret` + secret（无需额外 `lt-cred-mech`） | WebRTC、凭据需限时、信令服务器签发 |
| 静态 lt-cred-mech | `lt-cred-mech` + `-u user:pass` 或 DB `turnusers_lt` | 测试、固定客户端、内部服务 |

## 6. 有效期设置

有效期由多层机制叠加控制。

### 6.1 凭据有效期（信令服务器控制，最主要）

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

### 6.2 TURN Allocation 生命周期（coturn 控制）

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

### 6.3 Nonce 有效期

```ini
stale-nonce=600   # 默认 600s，过期后返回 438
```

### 6.4 配额限制

```ini
user-quota=10
total-quota=1200
max-bps=3000000
```

配额按**真实用户名**统计（`get_real_username` 会去掉时间戳前缀），同一 `userId` 无论时间戳如何变化，都计入同一配额。

如果 username 只有时间戳，那么每次签发出来的值都可能成为新的 quota key，无法稳定按真实用户限额。

## 7. 方案 A：信令服务器 API 设计

### 7.1 roomId 还是 userId？

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

- coturn 的 `user-quota` 按 `get_real_username()` 统计。对推荐格式 `expiry:userId`，quota key 是 `:` 后面的 `userId`；对兼容格式 `userId:expiry`，quota key 是 `:` 前面的 `userId`。
- 若写 `expiry:room-abc`，同一房间所有用户共用一个 quota key，无法按人限流。
- 若写 `expiry:room-abc:user-42` 可以，但不如直接用稳定 `userId` 简洁。
- `roomId` 是会话级概念，适合放在信令鉴权层；`userId` 是用户级概念，适合放进 TURN 凭据。

**接口是否还需要传 userId？**

通常**不需要**客户端显式传 `userId`，应从 token 解析，防止伪造。只有 token 不可信或调试阶段，才考虑让客户端传 `userId` 并由服务端校验一致性。

### 7.2 从 `Authorization: Bearer <token>` 获取 userId

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
import os
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

### 7.3 接口示例

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

### 7.4 设计要点

1. **Secret 只存在服务端**：信令服务器和 coturn 各持一份，通过 KMS/环境变量注入，不进代码仓库。
2. **userId 绑定稳定业务身份**：优先放稳定的 `userId`，不要只用时间戳；这样审计、配额和问题定位才有意义。
3. **短 TTL 的作用边界**：短 TTL 主要用于缩小凭据泄露后的滥用窗口，并限制新的 Allocate / 新认证；它不会强制结束已建立的 relay 会话。
4. **如需硬性到期，业务层配合**：例如停止下发新凭据、在房间结束时主动清理会话，或改用能显式限制会话上限的认证方案。
5. **HTTPS 传输**：凭据经公网下发，接口必须 TLS。
6. **Secret 轮换**：数据库多 secret 并存，信令服务器用新 secret 签发，coturn 新旧都接受。

## 8. WebRTC 客户端配置 coturn 地址与凭据

WebRTC 通过 `RTCPeerConnection` 的 `iceServers` 配置 STUN/TURN。浏览器和原生 SDK 都使用同一套 [W3C WebRTC 规范](https://www.w3.org/TR/webrtc/#rtciceserver-dictionary) 定义的字段。

### 8.1 字段说明

| 字段 | 说明 |
|------|------|
| `urls` | STUN/TURN 服务器地址，字符串或字符串数组 |
| `username` | TURN 用户名（REST API 生成的临时 username） |
| `credential` | TURN 密码（REST API 生成的 Base64 HMAC，**不是** `password`） |

注意：

- WebRTC API 中密码字段名是 **`credential`**，不是 `password`。
- `credential` 的值就是信令服务器按第 2 节算法生成的 Base64 字符串。
- STUN 条目通常不需要 `username` / `credential`；只有 TURN 中继才需要。

### 8.2 URL 格式

coturn 常见 URL 写法：

| URL | 含义 |
|-----|------|
| `stun:turn.example.com:3478` | STUN Binding，获取 reflexive 地址 |
| `turn:turn.example.com:3478` | TURN over UDP（默认） |
| `turn:turn.example.com:3478?transport=udp` | 显式指定 UDP |
| `turn:turn.example.com:3478?transport=tcp` | TURN over TCP |
| `turns:turn.example.com:5349?transport=tcp` | TURN over TLS（coturn 默认 TLS 端口 5349） |

生产环境建议同时提供 UDP 和 TCP/TLS，以应对对称 NAT、企业防火墙等场景。

### 8.3 浏览器 JavaScript 完整示例

以下示例展示从信令接口**请求 → 解析 → 提取 username/credential → 设置 iceServers → 创建 PeerConnection** 的完整流程。

#### 8.3.1 第一步：请求信令接口

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

#### 8.3.2 第二步：从响应中提取 username 和 credential

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

#### 8.3.3 第三步：组装 iceServers 并设置到 PeerConnection

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

#### 8.3.4 第四步：完整通话建立流程

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

#### 8.3.5 本地硬编码（仅开发调试）

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

#### 8.3.6 凭据临近过期时如何处理

**不要把 REST username 中的 `expiry` 理解成“到点必须重建 `RTCPeerConnection`”。** 对 coturn 而言，已认证成功的 TURN session 通常会缓存派生出的认证材料；只要当前 allocation 持续存活并正常 Refresh，`expiry` 到点本身一般不会立刻打断现有媒体。

真正需要**重新获取凭据**的场景，通常是：

- 还没建连成功，准备重新发起 Allocate；
- 连接断开后要重连；
- 业务主动做 ICE restart / 重新协商；
- 需要新建 `RTCPeerConnection` / 新建 TURN allocation。

同时要注意：**同一个 TURN allocation 不能在原地切换成新的 username。** 对 coturn 的长期凭据实现来说，非 OAuth 场景下同一 session 若更换 username，会被判为错误凭据；因此新凭据意味着新的 Allocate/ICE 建链过程，而不是“给旧 allocation 热切换密码”。

可在重连或重协商前做一个简单检查：

```javascript
function isCredentialNearExpiry(expiry, safetySeconds = 300) {
  return Math.floor(Date.now() / 1000) + safetySeconds >= expiry;
}
```

常见策略：

1. **已有会话稳定、媒体正常**：通常无需因为 `expiry` 到点而立即重建 `RTCPeerConnection`。
2. **预期会重连 / ICE restart / 新建会话**：若凭据已过期或即将过期，先重新 `fetchTurnCredentials()`。
3. **更新 `iceServers` 的方式**：浏览器标准接口提供 `pc.setConfiguration({ iceServers })`；若你的 SDK / 运行环境不便更新，或你希望行为最一致，也可以直接重建 `RTCPeerConnection`。
4. **未建连就过期**：直接重新取凭据，再走新的建链流程。

### 8.4 React Native / react-native-webrtc 示例

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

### 8.5 Android（org.webrtc）示例

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

### 8.6 iOS（WebRTC.framework）示例

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

### 8.7 iceTransportPolicy 选项

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

### 8.8 验证 TURN 是否生效

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

### 8.9 常见配置错误

| 错误 | 现象 | 修正 |
|------|------|------|
| 使用 `password` 而非 `credential` | 浏览器报配置无效或 TURN 认证失败 | 改用 `credential` 字段 |
| 前端硬编码 shared secret | 安全风险，secret 泄露 | secret 只放信令服务器和 coturn |
| 只配 STUN 不配 TURN | 对称 NAT 下无法连通 | 同时配置 `turn:` / `turns:` 条目 |
| `username` 时间戳已过期 | Allocate 返回 401/403 | 重新向信令服务器申请凭据 |
| coturn 在 NAT 后未设 `external-ip` | 有 `relay` 候选但媒体不通 | coturn 配置 `--external-ip` |
| TLS 证书不受信任 | `turns:` 连接失败 | 使用正规 CA 证书，或开发环境导入自签证书 |

## 9. 完整认证时序（客户端视角）

REST API 只是替换了 username/password 的生成方式，之后走标准长期凭据流程：

1. 客户端发 Allocate → coturn 回 **401** + `realm` + `nonce`。
2. 客户端计算 `key = MD5(username : realm : password)`，在请求中加入 `USERNAME`、`REALM`、`NONCE`、`MESSAGE-INTEGRITY`。
3. coturn 调用 `get_user_key()` 校验。
4. 成功后分配 relay 地址。

## 10. 验证配置

```bash
# 启动 coturn（REST API 模式）
turnserver --use-auth-secret --static-auth-secret=logen \
  --realm=north.gov -f -L 127.0.0.1 -E 127.0.0.1

# 客户端用 -W 传入相同 secret，-u 传入用户 ID
turnutils_uclient -u ninefingers -W logen -e 127.0.0.1 127.0.0.1
```

uclient 的 `-W` 会自动按 REST API 规则生成 username/password（`src/apps/uclient/mainuclient.c`）。这里故意不写 `-a`；`--use-auth-secret` 已自动启用长期凭据模式。

## 11. 常见问题

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

**方案 B 本地生成的凭据 coturn 能认吗？**

能。只要 `static-auth-secret` 相同、算法一致（第 2 节）、username 未过期，服务端或客户端本地生成的凭据都能通过 coturn 校验。参考 `turnutils_uclient -W <secret>`（第 3.5 节）。

## 12. 部署拓扑：公网 coturn + 内网全锥型 NAT 服务端 + 移动端

典型场景：

| 组件 | 位置 | 网络特征 |
|------|------|----------|
| **coturn** | 公网（有独立公网 IP 或稳定 VPS） | 移动端与内网服务端均可主动访问；**优先部署在 IP 稳定的机器上** |
| **Server** | 内网 | 出口为**全锥型 NAT**（Full Cone NAT）网关；公网 IP **可能频繁变化**（家庭/企业宽带） |
| **Client** | 移动端（4G/5G/WiFi） | 多为端口限制型或对称型 NAT，往往无法被对端直连 |

目标：移动端 Client 能访问内网 Server（信令 + 媒体），并在 NAT 复杂时通过公网 coturn 中继。

**公网 IP 经常变化时**，本方案的核心原则是：

1. **信令与 TURN 共用「固定锚点」**：`signal.example.com`、`turn.example.com` 的 DNS A 记录均指向 **VPS 固定 IP**，不指向会变动的网关 WAN IP（§12.3）。
2. **客户端只认域名，不认 IP**：App 内置 URL 终身不变；接口、远程配置均不下发 WAN IP。
3. **内网 Server 只出站**：通过 WSS 注册或 frp/WG 隧道接入 VPS 锚点；网关 IP 变化时 Server 侧重连，Mobile 无感知。
4. **媒体优先双端 TURN 中继**（§12.5 策略 A）：与网关 WAN IP 解耦。
5. **全锥型 NAT**：优化 Server 出站与可选管理面；**不是** Mobile 访问信令的前置条件（§12.2.1）。
6. **DDNS 不用于信令主域名**：仅在没有 VPS 时的备选（§12.3.2 模式 C）。

### 12.1 先分清两条通路

WebRTC 系统有两条独立通路，需分别设计：

| 通路 | 协议 | 作用 | 本场景要点 |
|------|------|------|-----------|
| **信令面** | HTTPS / WSS | 登录、房间、SDP/ICE 交换、TURN 凭据下发 | 移动端必须能连到信令入口 |
| **媒体面** | UDP/TCP（SRTP） | 音视频 RTP | 由 ICE 选路，常需 TURN 中继 |

信令不通则无法协商；信令通但 ICE 失败则无声画。两者都要覆盖。

### 12.2 全锥型 NAT 对内网 Server 意味着什么

**全锥型 NAT**（Full Cone / NAT Type 1）：内网主机 `内网IP:端口` 映射到 `公网IP:端口` 后，**任意外网主机**向该公网端口发包，均可转发到内网主机（只要映射存在）。

对内网 Server 的利好：

- 在网关上做 **端口映射（DNAT）** 后，外网可主动连入（信令 HTTPS、媒体 UDP 均可）。
- Server 主动访问公网 coturn 时，网关会建立映射；全锥型下外网从 coturn 回包通常能穿透（比对称型 NAT 友好）。

局限：

- 映射不会自动出现，需在网关上 **显式配置** 或使用 **TURN 中继** 绕过入站问题。
- 移动端对称型 NAT 仍可能无法与内网 Server **直连**，即使 Server 侧是全锥型。
- **公网 IP 变化**时，静态写死的 `public_host=203.0.113.x` 会立刻失效；必须配合 DDNS + 运行时 STUN 发现，或改用 §12.5 策略 A。

#### 12.2.1 全锥型 NAT 网关配置要点

RFC 3489 意义上的 **Full Cone** 需同时满足：

| 特性 | 含义 | 对 WebRTC 的作用 |
|------|------|------------------|
| **EIM**（Endpoint Independent Mapping） | 内网 `192.168.1.10:50000` 无论访问哪个外网目标，都映射到同一 `公网IP:端口` | Server 出站访问 coturn 后，coturn 可从任意源回包 |
| **EIF**（Endpoint Independent Filtering） | 映射建立后，**任意**外网 IP 向该公网端口发包均可转入内网 | 移动端或 coturn 可主动连入已映射的信令/媒体端口 |

常见家用/企业网关上的对应设置（名称因厂商而异）：

```
NAT 类型 / 模式     →  Full Cone / 完全锥型 / NAT Type 1（若可选）
过滤策略            →  关闭「仅允许已访问过的对端回包」（即启用 EIF）
端口映射 / 虚拟服务器 →  见下表
Hairpin / NAT Loopback →  开启（便于内网设备用公网域名自测信令）
```

**推荐端口映射（绑定 WAN 口，不写死公网 IP）**

| 外网端口 | 协议 | 内网目标 | 用途 |
|---------|------|---------|------|
| 443 | TCP | `192.168.1.10:443` | WSS 信令 + TURN 凭据 API |
| 3478 | UDP/TCP | 仅当 coturn 与 Server 同网关时 | STUN/TURN 监听（见 §12.11） |
| 5349 | TCP | 同上 | TURNS |
| 49152-65535 | UDP | 同上 | TURN relay 端口段（同网关部署 coturn 时） |
| 50000-50100 | UDP | `192.168.1.10:50000-50100` | 可选，§12.5 策略 B 媒体端口 |

映射规则应配置为 **「WAN 接口:any → 内网主机:端口」**，不要填具体公网 IP 字段（多数网关会自动跟随当前 WAN IP）。

**与端口限制型、对称型的区别**

| NAT 类型 | 外网主动连入已映射口 | IP 变化后 |
|---------|---------------------|----------|
| 全锥型 | 允许 | 映射通常仍有效（规则绑 WAN） |
| 端口限制型 | 仅允许「Server 先访问过的对端」 | 同上，但 coturn 多 IP 回包可能失败 |
| 对称型 | 每目标一对映射，几乎无法做固定 DNAT | 不适合本方案策略 B |

若网关无法设为全锥型，仍可用 **§12.4 方式 3（Server 出站连公云信令）+ §12.5 策略 A（双端 TURN）** 完成业务，只是无法依赖「外网主动打映射口」这条捷径。

### 12.3 推荐总体架构

**信令入口设计目标：移动端 App 只配置一次固定域名，永远不需要知道、也不应该解析到内网网关的动态 WAN IP。**

实现方式是把信令入口放在 **固定公网 IP 的锚点**（VPS / 公云）上；内网 Server 通过 **出站长连接或隧道** 接入锚点。动态 IP 只存在于网关 WAN 口，对客户端完全透明。

```mermaid
flowchart TB
    subgraph Anchor["固定锚点（VPS / 公云，IP 不变）"]
        SIG["信令入口<br/>signal.example.com<br/>A → 203.0.113.10 固定"]
        TURN["coturn<br/>turn.example.com<br/>A → 203.0.113.10 固定"]
    end

    subgraph Mobile["移动端 Client"]
        APP["App 内置<br/>wss://signal.example.com<br/>turn.example.com"]
    end

    subgraph Intranet["内网"]
        SRVR["Server<br/>信令逻辑 + 媒体/SFU"]
    end

    GW["全锥型 NAT 网关<br/>WAN IP 动态变化<br/>客户端不可见"]

    APP -->|"① WSS / HTTPS<br/>只连固定域名"| SIG
    APP -->|"② STUN / TURN"| TURN

    SRVR -->|"③ 出站 WSS 注册<br/>或 frp / WireGuard 隧道"| SIG
    SRVR -->|"④ 出站 TURN Allocate"| TURN

    SRVR --- GW

    APP -.->|"⑤ 媒体：双端 relay"| TURN
    TURN -.-> SRVR

    SIG -.->|"⑥ 信令中转"| SRVR
```

**与「DDNS 指向动态 IP」方案的本质区别**

| | DDNS 方案（不推荐作信令主入口） | 固定锚点方案（推荐） |
|---|---|---|
| `signal.example.com` 解析目标 | 随 WAN IP 变化的 A 记录 | **始终**指向 VPS 固定 IP |
| 客户端是否感知 IP 变化 | 不配置 IP，但 DNS 缓存/TTL 会导致连不上旧 IP | **完全无感知** |
| 内网 Server 角色 | 被动等待 Mobile 连入映射口 | **主动出站**注册到锚点 |
| IP 变化时信令 | WSS 断开，等 DDNS 刷新 | **不受影响**（锚点 IP 不变） |
| App 配置 | 域名 + 需重连逻辑 | **域名写死即可**，与 coturn 相同 |

推荐组合：**「VPS 固定锚点承载信令 + VPS coturn 中继媒体 + 内网 Server 仅出站」**。全锥型 NAT 仍建议配置（§12.2.1），用于 Server 出站稳定性及可选的管理面，但 **不是 Mobile 访问信令的前置条件**。

#### 12.3.1 信令入口：客户端无感知动态 IP 的三层设计

```
┌─────────────────────────────────────────────────────────────────┐
│  Layer 0 — 客户端（Mobile / Web）                                  │
│  只持有固定 URL，生命周期内不变：                                    │
│    SIGNAL_URL  = wss://signal.example.com/ws                      │
│    API_BASE    = https://signal.example.com/api/v1                │
│    TURN_HOST   = turn.example.com   （iceServers，§12.5）          │
│  ❌ 禁止：硬编码 IP、从接口下发 WAN IP、DDNS 域名指向家庭宽带          │
└───────────────────────────────┬─────────────────────────────────┘
                                │ HTTPS / WSS（目标永远是 VPS）
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│  Layer 1 — 固定锚点（VPS 203.0.113.10，DNS A 记录永不变）            │
│  承担：TLS 终结、WSS 接入、REST API、房间路由、TURN 凭据签发（§7）     │
│  可选组件：Nginx / Caddy / 自研信令集群 / Redis 房间状态               │
└───────────────────────────────┬─────────────────────────────────┘
                                │ 出站隧道 或 内网 Server 主动注册
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│  Layer 2 — 内网 Server（192.168.x.x，网关 WAN IP 随意变化）          │
│  只发起出站连接，不开放入站 443                                      │
│  媒体经 coturn relay，不暴露网关动态 IP 给 Mobile                     │
└─────────────────────────────────────────────────────────────────┘
```

**Layer 0 客户端配置示例（一次写入，永久有效）**

```javascript
// config.js — 编译进 App 或通过远程配置下发「域名」，绝不下发 IP
export const RTC_CONFIG = {
  signalUrl: 'wss://signal.example.com/ws',
  apiBase:   'https://signal.example.com/api/v1',
  turnHost:  'turn.example.com',
};

// 连接信令 — 无需任何 IP 发现逻辑
const ws = new WebSocket(RTC_CONFIG.signalUrl, [], {
  headers: { Authorization: `Bearer ${token}` },
});

// 申请 TURN 凭据 — 走同一固定域名
const res = await fetch(`${RTC_CONFIG.apiBase}/turn-credentials?roomId=${roomId}`, {
  headers: { Authorization: `Bearer ${token}` },
});
```

远程配置（Firebase Remote Config、自有配置中心）若更新信令地址，也应只改 **域名**（如切换灾备集群 `signal2.example.com`），不下发 IP。

#### 12.3.2 三种锚点模式（按推荐顺序）

##### 模式 A — 公云信令 + 内网 Server 出站注册（首选，详见第 13 节）

Mobile 与内网 Server **都只连 VPS**，Mobile 从不接触网关 WAN IP。**完整设计与实施步骤见第 13 节。**

```
Mobile ──WSS──▶ signal.example.com (VPS)
                    ▲
内网 Server ──WSS 长连接出站──┘   （gateway IP 变了无所谓）
```

| 项目 | 说明 |
|------|------|
| DNS | `signal.example.com` → VPS 固定 A 记录，**不用 DDNS** |
| 网关 | 仅需 Server **出站** 443；无需端口映射、无需 frp/nginx |
| 内网 Server | 启动后 `connect('wss://signal.example.com/agent')`，维持心跳；网关重拨后自动重连 |
| Mobile | 连 `wss://signal.example.com/ws` 加入房间；信令由 VPS 转发给已注册的 Server |
| TURN 凭据 | VPS 签发（§7 方案 A）；coturn 同机部署，`turn.example.com` 同 VPS |

**优点**：零 DDNS、零端口映射、零 frp、网关 IP 任意变；App 配置最简单。  
**缺点**：需维护 VPS 上的信令服务与 coturn（一台 VPS 即可）。

##### 模式 B — VPS 反向代理 + 出站隧道（信令逻辑仍在内网 Server）

信令业务代码留在内网，VPS 只做 **固定入口 + TLS + 隧道**。

```
Mobile ──HTTPS/WSS──▶ signal.example.com (VPS Nginx)
                           │
                     frp / WireGuard / SSH -R
                           │
                     内网 Server :443
```

| 项目 | 说明 |
|------|------|
| DNS | `signal.example.com` → VPS 固定 IP |
| frp 示例 | Server 端 `./frpc.toml` 主动连 VPS，映射 `localPort=443` → VPS `7000`；Nginx `proxy_pass` 到 `127.0.0.1:7000` |
| 网关 IP 变化 | frpc 自动重连 VPS，**Mobile 无感知** |
| TURN 凭据 | 仍由内网 Server 签发（请求经隧道到达） |

frpc 配置片段：

```toml
# 内网 Server 上 — 只需出站
serverAddr = "203.0.113.10"
serverPort = 7000

[[proxies]]
name     = "signal-wss"
type     = "tcp"
localIP  = "127.0.0.1"
localPort = 443
remotePort = 8443   # VPS Nginx 反代此端口
```

**优点**：信令代码零改动，仍在内网；App 仍只认固定域名。  
**缺点**：需运维 frp/WG；VPS 单点。

##### 模式 C — DDNS + 端口映射（仅作备选，不推荐）

```
Mobile ──▶ signal.example.com ──DDNS──▶ 当前 WAN IP ──DNAT──▶ 内网 Server
```

Mobile 虽只配置域名，但 **DNS 层**仍绑定动态 IP：TTL 缓存、DDNS 延迟会导致 IP 变更窗口内连不上。需客户端重连 + 短 TTL，体验不如模式 A/B。

**仅适合**：无法租用 VPS、且必须信令跑在内网 Server 上的极端情况。此时仍应保证 App **只写域名**，DDNS 由网关/脚本维护，但运维成本与故障面显著高于 A/B。

#### 12.3.3 模式选型

| 模式 | Mobile 配置 | 需知道 WAN IP | 网关 IP 变 | 需端口映射 | 推荐度 |
|------|------------|--------------|-----------|-----------|--------|
| **A 公云信令 + 出站注册** | 固定域名 | **否** | 无影响 | 否 | ★★★★★ |
| **B VPS 反代 + 隧道** | 固定域名 | **否** | 无影响 | 否 | ★★★★ |
| C DDNS + 映射 | 固定域名 | 否（但 DNS 层绑定动态 IP） | 有窗口期 | 是 | ★★ |

**结论**：动态 WAN IP 场景下，信令入口应设计为 **「固定锚点域名 → 固定 VPS IP」**；内网 Server **出站**接入锚点。Mobile 与 coturn 使用同一套「只认域名、不认 IP」的配置策略。

### 12.4 信令面：移动端如何访问内网 Server

内网 Server 没有稳定公网 IP 时，**不要让 Mobile 通过 DDNS 解析到网关 WAN IP**（§12.3.2 模式 C 仅作备选）。应让 Mobile 连接 **VPS 固定锚点**上的 `signal.example.com`，由内网 Server **出站**接入。

以下三种做法与 §12.3.2 模式 A / B / C 一一对应：

#### 方式 1（模式 C）：全锥型网关端口映射 + DDNS — 备选

> **不推荐**作为动态 IP 场景的主方案。仅在没有 VPS 时使用；Mobile 虽只配域名，但 DNS 仍跟踪动态 WAN IP。

在 NAT 网关上配置（**规则绑定 WAN，不写死公网 IP**）：

```
WAN:443   →  内网 Server:443   （WSS 信令 + REST API）
WAN:8443  →  内网 Server:8443 （备用）
```

网关上启用 **DDNS 客户端**（或内网跑 `ddclient` / `inadyn`），将 `signal.example.com` 的 A 记录指向当前 WAN IP：

```
signal.example.com  →  <当前 WAN IP>   TTL 建议 60–300s
```

移动端连接：

```
wss://signal.example.com/room   →  仅域名，不配置 IP
```

Server 部署信令服务（Node/Go/Java + WebSocket），并提供第 7 节 `GET /api/v1/turn-credentials`。`buildIceServers()` 中 TURN 主机名使用 **`turn.example.com`**（指向 VPS），不要用网关当前 IP。

**IP 变更时**：DDNS 更新前 Mobile 可能连旧 IP；已建立 WSS 会断开，需 **指数退避重连**。若已有 VPS，应改用方式 2 或 3。

#### 方式 2（模式 B）：VPS 反向代理 + 出站隧道

```
Mobile --HTTPS/WSS--> signal.example.com (VPS Nginx) --frp/WG/专线--> 内网 Server
```

- Mobile 只访问 **VPS 固定 IP** 上的域名；DNS A 记录**永不指向**家庭宽带 WAN IP。
- 内网 Server 通过 frp / WireGuard 等 **出站**连 VPS，网关 IP 变化时隧道自动重连。
- TURN 凭据仍由内网 Server 签发（HTTP 请求经隧道到达）。
- 配置细节见 §12.3.2 模式 B。

#### 方式 3（模式 A）：信令在 VPS，内网 Server 出站注册 — 首选

> **完整实施步骤见第 13 节。**

```
Mobile --WSS--> signal.example.com (VPS 信令集群)
内网 Server --WSS 长连接出站--> 同一 VPS（/agent 注册）
```

- Mobile 与 Server **都只连 VPS**；网关无需端口映射、无需 DDNS、无需 frp。
- 内网 Server **主动连出**，网关 WAN IP 任意变化，出站重连即可。
- TURN 凭据在 VPS 签发（§7）；coturn 与信令同机部署。

| 方式 | 对应模式 | 网关改动 | Mobile 是否感知 WAN IP | 公网 IP 变化 | 推荐 |
|------|---------|---------|----------------------|-------------|------|
| 1 映射 + DDNS | C | 映射 + DDNS | 否（但 DNS 跟踪动态 IP） | 有窗口期 | 备选 |
| 2 VPS 反代 + 隧道 | B | 仅需 Server 出站 | **完全无感知** | **无影响** | 推荐 |
| 3 VPS 信令 + 出站注册 | A | 仅需 Server 出站 | **完全无感知** | **无影响** | **首选** |

**动态 WAN IP 场景**：优先 **方式 3** 或 **方式 2**；Mobile App 内置 `wss://signal.example.com`，与 `turn.example.com` 一样，**终身不需要知道网关 IP**。

### 12.5 媒体面：ICE 与 coturn 角色

#### 移动端 Client（几乎总是需要 TURN）

蜂窝网络多为对称型 NAT，外网无法主动连入；生产上**通常应配置公网 coturn**，在复杂 NAT / 运营商网络场景下几乎是必需的：

```javascript
const pc = new RTCPeerConnection({
  iceServers: [
    { urls: 'stun:turn.example.com:3478' },
    {
      urls: [
        'turn:turn.example.com:3478?transport=udp',
        'turn:turn.example.com:3478?transport=tcp',
        'turns:turn.example.com:5349?transport=tcp',
      ],
      username,   // 信令 API 下发，第 7 节
      credential,
    },
  ],
  iceTransportPolicy: 'all',  // 调试连通性时可临时改为 'relay'
});
```

预期：移动端 ICE 候选中应出现 `typ relay`（经 coturn 中继）。

#### 内网 Server 侧（两种策略）

**策略 A — Server 也走 coturn 中继（推荐；公网 IP 变化时首选）**

- Server 与移动端一样，向公网 coturn 做 TURN Allocate。
- 媒体路径：`Mobile ↔ coturn ↔ Server`（两端都经 relay，或一端 relay 一端能直达 coturn）。
- 不依赖网关上开大量 UDP 端口映射，**也不依赖网关当前公网 IP**。
- Server 可用第 3 节方案 B 在本地生成 TURN 凭据（C/C++/JavaScript 均可）。
- coturn 域名指向 **IP 稳定的 VPS** 时，网关 WAN IP 变化只影响信令入口（§12.4），**不影响已建立的 relay 媒体**；新会话需确保信令可达。

**策略 B — Server 利用全锥型 NAT 暴露 UDP 端口（省 coturn 带宽；公网 IP 变化时不推荐）**

在网关上映射 WebRTC 媒体端口段（**绑定 WAN，不写死 IP**）：

```
WAN:50000-50100 UDP  →  内网 Server:50000-50100 UDP
```

**禁止**在配置里写死 `public_host=203.0.113.x`。应通过 STUN 向公网 coturn 查询当前 reflexive 地址，或在 Server 启动时用 HTTP/DNS 探测当前 WAN IP：

```javascript
// 示例：用 coturn STUN 获取 Server 当前公网 reflexive 地址（网关 WAN IP）
const pc = new RTCPeerConnection({
  iceServers: [{ urls: 'stun:turn.example.com:3478' }],
});
pc.createDataChannel('probe');
pc.createOffer().then((o) => pc.setLocalDescription(o));
pc.onicecandidate = (e) => {
  if (!e.candidate) return;
  const m = e.candidate.candidate.match(/(\d+\.\d+\.\d+\.\d+) (\d+) typ srflx/);
  if (m) {
    console.log('current public host:', m[1], 'port:', m[2]);
    // 写入 SFU / libwebrtc 对外公布的 host 候选
  }
};
```

libwebrtc 若启用 `stun_server`，通常会自动产生 `typ srflx` 候选，无需手工填 `public_host`。IP 变化后需 **ICE restart** 重新收集候选。

同时仍配置 coturn 作为 fallback。移动端对称 NAT 时最终可能仍只有 relay 路径可用。

#### 媒体路径对照

| 路径 | 条件 | 公网 IP 变化时 | 稳定性 |
|------|------|---------------|--------|
| Client `relay` ↔ Server `relay` | 双方都用 coturn | coturn 在 VPS 上则**不受影响** | **高（推荐）** |
| Client `relay` ↔ Server `host/srflx` | Server 有 UDP 映射 + 正确 reflexive | srflx 失效，需 ICE restart | 中 |
| Client `host/srflx` ↔ Server `host/srflx` | 双方 NAT 均可穿透 | 双方 reflexive 均可能失效 | 低（移动端少见） |

### 12.6 公网 coturn 配置要点

#### 12.6.1 coturn 在 VPS（固定公网 IP，推荐）

coturn 在公网且有独立公网 IP 时，典型配置：

```ini
listening-port=3478
tls-listening-port=5349

listening-ip=203.0.113.20
relay-ip=203.0.113.20

use-auth-secret
static-auth-secret=<与信令服务器相同>

realm=example.com

min-port=49152
max-port=65535

fingerprint
no-multicast-peers

# 客户端 iceServers 使用 turn.example.com → 上述固定 IP
# 无需 external-ip
```

命令行等价：

```bash
turnserver \
  --use-auth-secret --static-auth-secret=SECRET \
  --realm=example.com \
  -L 203.0.113.20 -E 203.0.113.20 \
  --min-port=49152 --max-port=65535 \
  -f
```

#### 12.6.2 coturn 与 Server 同处动态 IP 网关后（不推荐，仅资源受限时）

若 coturn 必须部署在内网、与 Server 共用会变化的 WAN IP，需同时满足：

1. 网关 **全锥型 NAT** + **1:1 端口映射**（§12.2.1）：`3478/5349` 与 `49152-65535/udp` 整段映射到 coturn 内网主机。
2. 配置 `external-ip`，让 `XOR-RELAYED-ADDRESS` 返回当前公网 IP，而非内网 IP。
3. WAN IP 变化时 **刷新 `external-ip` 并重启 coturn**（coturn 不支持运行中热更新 `-X`）。
4. `turn.example.com` 走 DDNS 指向当前 WAN IP；已建立的 relay 会话在 IP 变化后**会中断**。

启动脚本示例（使用 coturn Docker 镜像自带的 `detect-external-ip`）：

```bash
#!/bin/bash
# /usr/local/bin/start-turnserver-dynamic.sh

PRIVATE_IP="192.168.1.20"
PUBLIC_IP="$(detect-external-ip --ipv4)" || exit 1

exec turnserver \
  --use-auth-secret --static-auth-secret="$TURN_SECRET" \
  --realm=example.com \
  -L "$PRIVATE_IP" -E "$PRIVATE_IP" \
  --external-ip="${PUBLIC_IP}/${PRIVATE_IP}" \
  --min-port=49152 --max-port=65535 \
  -f
```

配合 systemd timer 或 cron，在 WAN 重拨后探测 IP 变化并重启：

```bash
# /etc/cron.d/coturn-ip-watch — 每 5 分钟检查一次
*/5 * * * * root /usr/local/bin/coturn-ip-watch.sh
```

```bash
#!/bin/bash
# coturn-ip-watch.sh
CURRENT="$(detect-external-ip --ipv4 2>/dev/null)" || exit 0
STORED="$(cat /var/run/coturn-public-ip 2>/dev/null)"
if [ "$CURRENT" != "$STORED" ]; then
  echo "$CURRENT" > /var/run/coturn-public-ip
  systemctl restart coturn
fi
```

`detect-external-ip` 源码见 `docker/coturn/rootfs/usr/local/bin/detect-external-ip.sh`，通过 OpenDNS / Google / Cloudflare DNS 查询当前公网 IPv4。

**映射要求（coturn 官方说明）**：`external-ip` 对应的公网端口必须与内网 relay 端口 **一一对应**（relayed port 12345 必须始终映射到外网 12345）。全锥型 NAT + 固定 DNAT 通常满足；对称型 NAT 无法满足，不应在此部署 coturn。

### 12.7 端到端流程（方案 A 信令 + 双端 TURN）

```
1. Mobile 登录 → 获得 Bearer token
2. Mobile WSS 连接信令入口（映射/反代/公云，§12.4；URL 为域名）
3. Mobile GET /api/v1/turn-credentials?roomId=xxx
   → 信令 Server 从 token 取 userId，签发 username/credential
   → iceServers 中 TURN 主机名为 turn.example.com（VPS 固定 IP）
4. Mobile 创建 RTCPeerConnection(iceServers)，加入房间
5. 内网 Server 启动时已连信令（出站 WSS 或经映射入站）；本地生成 TURN 凭据（§3）
6. 信令交换 Offer/Answer + ICE candidates
7. ICE 连通：
   - Mobile 候选多为 relay（公网 coturn）
   - Server 候选为 relay（策略 A，推荐）
8. SRTP 媒体经 coturn 双向中继
9. （若网关 WAN IP 变化）WSS 断开 → 客户端重连 + 重取凭据 → ICE restart（§12.11.3）
```

```mermaid
sequenceDiagram
    participant M as 移动端 Client
    participant S as 信令入口
    participant I as 内网 Server
    participant T as coturn 公网

    M->>S: WSS 连接 + Bearer token
    M->>S: GET /turn-credentials?roomId=
    S->>I: （可选）校验房间成员
    S-->>M: iceServers + username/credential

    I->>S: WSS 已连接 / 加入房间
    I->>I: 本地生成 TURN 凭据（方案 B）

    M->>S: Offer + ICE candidates
    S->>I: 转发
    I->>S: Answer + ICE candidates
    S->>M: 转发

    M->>T: TURN Allocate (relay)
    I->>T: TURN Allocate (relay 或 STUN)

    Note over M,I: 媒体经 T 中继或 T 与 I 映射口互通
```

### 12.8 内网 Server 参考配置清单

**网络 / 网关**

- [ ] 信令端口映射或公网反代（§12.4）
- [ ] 网关设为全锥型 NAT（§12.2.1），端口映射绑定 WAN 口
- [ ] DDNS：`signal.example.com` 指向当前 WAN IP（方式 1）；`turn.example.com` 指向 VPS 固定 IP（推荐）
- [ ] （可选）媒体 UDP 端口段映射（§12.5 策略 B；公网 IP 变化时不推荐）
- [ ] 防火墙允许 Server **出站** 访问 coturn `3478/5349` 与 `49152-65535/udp`
- [ ] 若 coturn 同网关部署：映射 `3478/5349/tcp/udp` + `49152-65535/udp`，并部署 IP 探测重启脚本（§12.6.2）

**Server 进程**

- [ ] 信令服务监听内网 `0.0.0.0:443`（WSS）
- [ ] TURN REST API 或本地 secret 生成（第 7 / 3 节）
- [ ] WebRTC：`iceServers` 使用 **`turn.example.com` 域名**，不要用内网私网地址或 WAN IP
- [ ] 媒体策略优先 **双端 TURN relay**（§12.5 策略 A）
- [ ] libwebrtc / 自研栈：开启 ICE trickle；监听 `iceConnectionState` 断连后触发 ICE restart + 重取凭据
- [ ] WSS 客户端：WAN IP 变化导致长连接断开时，指数退避重连

**移动端 App**

- [ ] 信令 URL 使用公网域名（非 `192.168.x.x`，非硬编码 IP）
- [ ] `iceServers` 含 `turn:` 与 `turns:`（第 8 节），主机名为域名
- [ ] 蜂窝网络下测试 `typ relay` 是否出现
- [ ] 若预期会断线重连、ICE restart 或新建会话，凭据 TTL 覆盖该窗口，或在重连前重新获取凭据（第 6 节）
- [ ] DNS 缓存：IP 变化后若解析失败，可 `ConnectivityManager` / `NWPathMonitor` 触发重新解析

### 12.9 coturn 与配额

公网 coturn 同时服务移动端与内网 Server 时，建议：

```ini
user-quota=10
total-quota=5000
max-bps=0
```

- `username` 中嵌入稳定 `userId`（第 7.1 节），Mobile 与 Server 各自独立计数。
- 监控 coturn 带宽；双端 relay 时流量经 coturn 双倍经过，需容量规划。

### 12.10 常见问题

**移动端能 ping 通 coturn 公网 IP，但无 `typ relay` 候选**

- 检查 username/credential 是否过期、secret 是否与 coturn 一致。
- 检查运营商是否封锁 UDP 3478；改用 `turns:5349?transport=tcp`。

**信令 WSS 能连，媒体不通**

- 信令与媒体独立；重点查 ICE 与 TURN。
- 内网 Server 是否仍把 `iceServers` 配成内网 STUN 地址。
- coturn `relay-ip` 是否为公网可达地址。

**Server 在全锥型 NAT 后，是否还需要 TURN？**

- 不强制，但**强烈建议**至少作为 fallback。
- 移动端对称 NAT 无法直连 Server 映射口时，必须靠 TURN。

**能否让 Mobile 直连内网 Server 私网 IP？**

- 不能（除非 Mobile 与 Server 同一内网或 Mobile 先连 VPN 进内网）。
- 移动端应只使用公网信令地址 + 公网 coturn。

**信令入口能否用 DDNS 指向家庭宽带？**

- Mobile 只配域名时可以连，但 **DNS 层仍绑定动态 IP**，IP 变更窗口内会失败。
- 更便捷的做法：`signal.example.com` 固定指向 VPS，内网 Server **出站**接入（§12.3.2 模式 A/B）；Mobile **完全不需要知道 WAN IP**，也无需 DDNS 刷新等待。

**App 里信令 URL 应该怎么写？**

- 编译期或远程配置写死 `wss://signal.example.com/ws`，与 `turn.example.com` 同级。
- 接口响应、推送通知、二维码里均 **不要** 出现 IP 或「当前 WAN 地址」字段。

### 12.11 公网 IP 频繁变化：完整应对方案

本节汇总 §12 开头四条原则的可落地做法，适用于家庭/企业宽带 DHCP 重拨、4G CPE 换基站等场景。

#### 12.11.1 推荐拓扑（动态 IP 下的最佳实践）

```
┌──────────┐  ① WSS / HTTPS（固定域名）
│  Mobile  │──────────────────────────────────────────────┐
└──────────┘  signal.example.com                          │
       │         turn.example.com                          │
       │         DNS A → VPS 203.0.113.10（永不变）         ▼
       │              ┌─────────────────────────────────────┐
       │              │  VPS 固定锚点                         │
       │              │  • 信令入口 signal.example.com       │
       │              │  • coturn turn.example.com           │
       │              └───────────▲─────────────────────────┘
       │                          │
       │              ③ 出站 WSS 注册（Server 主动连 VPS，§13）
       │              ┌───────────┴────────────┐
       │              │ 全锥型 NAT 网关         │
       │              │ WAN IP 动态（客户端不可见）│
       │              └───────────▲────────────┘
       │                          │
       └──── ④ TURN relay ────────┼──── 内网 Server 192.168.1.10
                                  │
                    ② 无 DDNS、无 Mobile 入站
```

| 组件 | 动态 IP 策略 |
|------|-------------|
| **信令入口** | VPS 固定锚点；模式 A 出站注册（**第 13 节**） |
| **coturn** | 同 VPS，`turn.example.com` 固定 A 记录 |
| **内网 Server** | 出站注册或隧道；网关 IP 变 → Server 重连，Mobile 无感 |
| **Mobile** | App 内置两个固定域名；零 IP 发现逻辑 |
| **网关 WAN IP** | 对客户端完全透明 |

#### 12.11.2 DNS 配置要点

```
# 推荐：两个域名均指向 VPS 固定 IP，不指向家庭宽带
signal.example.com   A   203.0.113.10   TTL=3600   Proxied=否
turn.example.com     A   203.0.113.10   TTL=3600   Proxied=否

# 不推荐：信令域名 DDNS 到动态 WAN（仅模式 C 备选）
# signal.example.com   A   <动态 WAN IP>   ← 避免
```

注意：

- **不要**对 `signal.example.com` / `turn.example.com` 开 CDN 橙云代理；WSS 长连接与 UDP 3478 无法经 HTTP CDN 正确代理。
- VPS 上 TLS 证书用 **Let's Encrypt HTTP-01**（VPS IP 固定）或 DNS-01 / 通配符 `*.example.com`。
- 若必须使用模式 C（DDNS 备选），信令域名 TTL 设 60–120s，并见 §12.4 方式 1 的重连策略；仍建议尽快迁移到模式 A/B。

#### 12.11.3 IP 变化时各组件行为

| 事件 | 信令面（模式 A/B） | 信令面（模式 C / DDNS） | 媒体面（策略 A） |
|------|-------------------|------------------------|-----------------|
| 网关 WAN IP 变化 | **无影响**（Mobile 连 VPS） | WSS 断开；等 DDNS 刷新 | coturn 在 VPS：**不受影响** |
| VPS 故障 | Mobile / Server 均不可连 | 同左 | 同左 |
| 内网 Server 重启 | 出站重连 VPS 后恢复 | 同左 | 短暂中断后 relay 恢复 |

**客户端恢复流程（推荐实现）**：

```
1. WSS onclose / onerror → 指数退避重连（1s, 2s, 4s … 上限 60s）
2. 重连成功后重新 GET /turn-credentials
3. iceConnectionState → failed/disconnected 持续 N 秒 → ICE restart
4. ICE restart 前检查凭据 expiry，过期则先换凭据
5. 仍失败 → 提示用户检查网络，或降级为仅音频
```

#### 12.11.4 信令 API 下发 iceServers 的建议

信令服务器返回 TURN URL 时**始终使用域名**：

```javascript
function buildIceServers(username, credential) {
  const TURN_HOST = process.env.TURN_HOST; // turn.example.com，非 IP
  return [
    { urls: `stun:${TURN_HOST}:3478` },
    {
      urls: [
        `turn:${TURN_HOST}:3478?transport=udp`,
        `turn:${TURN_HOST}:3478?transport=tcp`,
        `turns:${TURN_HOST}:5349?transport=tcp`,
      ],
      username,
      credential,
    },
  ];
}
```

可选：在响应中附带 `turnHost` 域名与 `ttl`，便于客户端日志排查；**不要**下发网关 WAN IP。

#### 12.11.5 全锥型 NAT 在动态 IP 场景下的价值

即使 WAN IP 会变，全锥型 NAT 仍然必要，原因：

1. **端口映射规则绑 WAN 口**：IP 变化后，同一组 DNAT（443 → Server）自动作用于新 IP，无需逐条改规则。
2. **Server 出站 coturn**：EIM 保证 Server 访问 VPS coturn 时映射稳定，回包不受 IP 变化影响（映射在 WAN 口重建前有效）。
3. **外网主动连入信令**：EIF 保证 Mobile 在 DDNS 刷新后，能向新 IP 的 443 端口发起 WSS。

若网关只能端口限制型 NAT，方式 1（端口映射信令）仍可用，但 Server 侧访问 coturn 若遇多源 IP 回包，建议直接改 **方式 3（Server 出站连公云信令）** 消除入站依赖。

#### 12.11.6 验证清单（动态 IP 专项）

```bash
# 1. 确认 coturn 在 VPS 可达（与网关 IP 无关）
turnutils_uclient -u test -W "$SECRET" turn.example.com

# 2. 模拟 WAN IP 变化：手动改 DDNS 记录后，从新网络解析域名
dig +short signal.example.com
dig +short turn.example.com

# 3. 移动端蜂窝网：Trickle ICE 或 App 日志确认 typ relay

# 4. 网关重拨后：检查 WSS 是否在 2×TTL 内自动恢复
```

**网关公网 IP 变了，TURN 还能用吗？**

- coturn 在 VPS：**能用**，客户端始终连 `turn.example.com`。
- coturn 在同网关：**进行中会话中断**；需 DDNS 更新 + coturn 重启 + 客户端重连。

**能否不写 DDNS，直接把 WAN IP 下发给 App？**

- 不推荐。IP 变化时需发版或推送更新；且 Mobile 缓存旧 IP 会导致长时间不可用。应使用 DDNS + 域名。

**全锥型 NAT 配好了，是否就可以不用 TURN？**

- 不能省略。全锥型解决的是 **内网 Server 被外网连入**；移动端对称 NAT 仍无法被 Server 主动连入，媒体仍需 coturn relay。

模式 A（第 13 节）下，全锥型 NAT **不是信令前置条件**；仍建议配置以利于 Server 出站稳定性。

---

## 13. 首选方案实施指南：公云信令 + 内网 Server 出站注册

本节是 §12.3.2 **模式 A** 的完整设计与分步部署手册。适用场景：

- 内网 Server 网关 **公网 IP 频繁变化**（家庭/企业宽带、4G CPE）；
- Mobile 需访问内网 Server 的 WebRTC 能力（信令 + 媒体）；
- 希望 **不部署 frp、不在内网开入站 443、不用 DDNS 指向家庭宽带**；
- 接受 **一台 VPS**（固定公网 IP）承载信令入口与 coturn。

### 13.1 设计目标与约束

| 目标 | 实现方式 |
|------|----------|
| Mobile **不知道**网关 WAN IP | App 只配置 `signal.example.com` / `turn.example.com` |
| 网关 IP 变化 **不影响** Mobile 连信令 | 域名 A 记录指向 VPS，不指向家庭宽带 |
| 内网 **无需公网入站** | Server 仅 **出站 WSS** 注册到 VPS |
| 部署 **尽量简单** | VPS：coturn + 信令 + Caddy（TLS）；内网：业务 + Agent 客户端 |
| 复杂 NAT 下媒体可达 | 双端 TURN relay（§12.5 策略 A） |

**刻意不使用的组件**

| 组件 | 原因 |
|------|------|
| frp / cloudflared | 模式 A 用应用层 WSS 注册替代 L4 反向隧道 |
| 内网 nginx | TLS 在 VPS Caddy 终结；内网无入站 HTTPS |
| 信令 DDNS | DNS 不跟踪动态 WAN IP |
| 网关 443 端口映射 | Mobile 不连家庭宽带 IP |

### 13.2 总体架构

```mermaid
flowchart TB
    subgraph VPS["VPS 203.0.113.10（固定 IP）"]
        CADDY["Caddy :443<br/>TLS 终结"]
        SIG["信令服务 :8080<br/>WSS + REST API"]
        TURN["coturn<br/>3478 / 5349"]
        CADDY --> SIG
    end

    subgraph Mobile["Mobile App"]
        APP["RTCPeerConnection<br/>+ WSS Client"]
    end

    subgraph LAN["内网"]
        AGENT["Agent 客户端<br/>出站 WSS"]
        SVC["Server 业务<br/>WebRTC / SFU"]
        AGENT --- SVC
    end

    GW["NAT 网关<br/>WAN IP 动态"]

    APP -->|"① wss://signal.../ws"| CADDY
    APP -->|"② GET /api/v1/turn-credentials"| CADDY
    APP -->|"③ STUN/TURN turn.example.com"| TURN

    SVC --> AGENT
    AGENT -->|"④ wss://signal.../agent<br/>出站注册"| CADDY
    SVC -->|"⑤ TURN Allocate 出站"| TURN

    SVC --- GW

    APP -.->|"⑥ SRTP 经 relay"| TURN
    TURN -.-> SVC
```

**角色分工**

| 组件 | 部署位置 | 职责 |
|------|----------|------|
| **Caddy** | VPS | HTTPS/WSS TLS、自动 Let's Encrypt |
| **信令服务** | VPS | Agent 注册表、房间路由、JWT 校验、TURN 凭据签发 |
| **coturn** | VPS | STUN/TURN；`static-auth-secret` 与信令服务相同 |
| **Agent 客户端** | 内网 Server | 出站 WSS 长连接、心跳、转发 Mobile 信令到本地业务 |
| **Server 业务** | 内网 Server | WebRTC 媒体逻辑、Answer/ICE 处理 |
| **Mobile App** | 手机 | WSS 信令、REST 取凭据、`RTCPeerConnection` |

### 13.3 域名与 DNS

两台主机名可指向 **同一 VPS IP**：

```
signal.example.com   A   203.0.113.10   TTL=3600
turn.example.com     A   203.0.113.10   TTL=3600
```

- **不要** 开 CDN 橙云代理（WSS 长连接、UDP TURN 无法经 HTTP CDN 代理）。
- TLS 证书由 VPS 上 Caddy 通过 **HTTP-01** 自动申请（IP 固定，最简单）。

Mobile / 内网 Server **终身只使用上述域名**，配置中不出现 IP。

### 13.4 信令协议设计

#### 13.4.1 HTTP / WSS 路径

| 路径 | 协议 | 连接方 | 说明 |
|------|------|--------|------|
| `/ws` | WSS | Mobile | 房间信令：join、offer、answer、ice-candidate |
| `/agent` | WSS | 内网 Server | Agent 出站注册 + 接收转发信令 |
| `/api/v1/turn-credentials` | HTTPS GET | Mobile | 签发 TURN 凭据（§7） |
| `/api/v1/health` | HTTPS GET | 运维 | 健康检查 |

Caddy 将 `signal.example.com` 全部反代到信令服务 `127.0.0.1:8080`；coturn 独立监听 3478/5349，由 `turn.example.com` 解析到同 VPS。

#### 13.4.2 Agent 注册（内网 → VPS）

内网 Server 建立 `wss://signal.example.com/agent` 后，**首条消息**：

```json
{
  "type": "register",
  "deviceId": "device-001",
  "token": "<AGENT_TOKEN>",
  "meta": { "version": "1.0.0", "hostname": "srv-lan-01" }
}
```

VPS 校验 `AGENT_TOKEN`（预共享或 JWT），成功后：

```json
{ "type": "registered", "deviceId": "device-001", "ts": 1735689600 }
```

之后每 **30s** 发送心跳：

```json
{ "type": "ping", "deviceId": "device-001" }
```

VPS 回复：

```json
{ "type": "pong", "ts": 1735689630 }
```

**断线重连**：指数退避 `1s → 2s → 4s → … → 60s`；重连后重新 `register`。

#### 13.4.3 Mobile 加入房间（Mobile → VPS → Agent）

Mobile 连接 `wss://signal.example.com/ws`，Header 带用户 JWT：

```
Authorization: Bearer <user_jwt>
```

加入房间：

```json
{
  "type": "join",
  "roomId": "room-abc",
  "targetDeviceId": "device-001"
}
```

VPS 逻辑：

1. 校验 user JWT，解析 `userId`；
2. 查 `agents.get(targetDeviceId)`；
3. 若在线，向 Agent 转发（附 `clientUserId`、`roomId`）：

```json
{
  "type": "join",
  "roomId": "room-abc",
  "clientUserId": "user-42",
  "relayId": "uuid-for-this-session"
}
```

4. 若 Agent 离线：

```json
{ "type": "error", "code": "DEVICE_OFFLINE", "deviceId": "device-001" }
```

#### 13.4.4 WebRTC 信令转发

Mobile 与内网 Server 之间的 SDP / ICE **原样经 VPS 中转**，消息统一格式：

```json
{
  "type": "signal",
  "roomId": "room-abc",
  "targetDeviceId": "device-001",
  "payload": {
    "kind": "offer",
    "sdp": "v=0\r\n..."
  }
}
```

```json
{
  "type": "signal",
  "roomId": "room-abc",
  "targetDeviceId": "user-42",
  "payload": {
    "kind": "ice-candidate",
    "candidate": { "candidate": "...", "sdpMid": "0", "sdpMLineIndex": 0 }
  }
}
```

VPS **不解析** SDP 内容，只做路由（`targetDeviceId` → Agent 或 Mobile 连接）。

#### 13.4.5 连接状态模型（VPS 内存）

```
agents: Map<deviceId, AgentConnection>
clients: Map<relayId, ClientConnection>   // relayId 由 join 分配
rooms: Map<roomId, { clientRelayId, deviceId }>
```

Agent 断开 → 清理 `agents` 中条目，向相关 Mobile 推送 `DEVICE_OFFLINE`。

### 13.5 安全设计

| 项目 | 做法 |
|------|------|
| **Mobile 身份** | 登录 JWT；`/ws` 与 `/turn-credentials` 必须 `Authorization: Bearer` |
| **Agent 身份** | 预共享 `AGENT_TOKEN` 或服务端签发 Agent JWT；仅允许注册已知 `deviceId` |
| **TURN secret** | 仅 VPS 信令服务 + coturn 持有；不下发 Mobile |
| **TURN username** | `expiry:userId`（§7.1）；`userId` 从 Mobile JWT 解析 |
| **传输** | 全程 TLS；`signal.example.com` 必须 WSS / HTTPS |
| **Agent 令牌轮换** | 环境变量注入；轮换时内网 Agent 重启拉新 token |

内网 Server **不需要**对 Mobile 开放任何端口；Agent 出站即可。

### 13.6 VPS 部署步骤

以下示例 VPS IP 为 `203.0.113.10`，系统 Ubuntu 24.04。

#### 步骤 1：DNS

在域名服务商添加 A 记录（见 §13.3），等待生效：

```bash
dig +short signal.example.com   # 应返回 203.0.113.10
dig +short turn.example.com
```

#### 步骤 2：防火墙

```bash
ufw allow 22/tcp
ufw allow 80/tcp      # Caddy HTTP-01
ufw allow 443/tcp     # HTTPS / WSS
ufw allow 3478/tcp
ufw allow 3478/udp    # STUN/TURN
ufw allow 5349/tcp    # TURNS
ufw allow 49152:65535/udp   # TURN relay
ufw enable
```

#### 步骤 3：安装 coturn

```bash
apt-get update && apt-get install -y coturn

cat > /etc/turnserver.conf <<'EOF'
listening-port=3478
tls-listening-port=5349
listening-ip=203.0.113.10
relay-ip=203.0.113.10
use-auth-secret
static-auth-secret=REPLACE_WITH_LONG_RANDOM_SECRET
realm=example.com
min-port=49152
max-port=65535
fingerprint
no-multicast-peers
no-cli
EOF

sed -i 's/#TURNSERVER_ENABLED=1/TURNSERVER_ENABLED=1/' /etc/default/coturn
systemctl enable --now coturn
```

`REPLACE_WITH_LONG_RANDOM_SECRET` 与信令服务环境变量 `TURN_SECRET` **必须相同**。

验证：

```bash
turnutils_uclient -u testuser -W "REPLACE_WITH_LONG_RANDOM_SECRET" turn.example.com
```

#### 步骤 4：安装 Caddy

```bash
apt-get install -y debian-keyring debian-archive-keyring apt-transport-https curl
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | tee /etc/apt/sources.list.d/caddy-stable.list
apt-get update && apt-get install -y caddy
```

```caddyfile
# /etc/caddy/Caddyfile
signal.example.com {
    reverse_proxy 127.0.0.1:8080
}
```

```bash
systemctl reload caddy
```

coturn 使用独立域名 `turn.example.com`，**不经过 Caddy**（UDP/TCP 3478 直连 VPS IP）。

#### 步骤 5：部署信令服务

以下示例为单文件 Node.js 服务（生产可换 Go/Java，协议按 §13.4 保持一致）。

**环境变量**（`/etc/signal-hub/env`）：

```bash
PORT=8080
JWT_SECRET=replace-user-jwt-secret
TURN_SECRET=REPLACE_WITH_LONG_RANDOM_SECRET
TURN_HOST=turn.example.com
TURN_PORT=3478
AGENT_TOKEN=replace-agent-pre-shared-token
ALLOWED_DEVICES=device-001,device-002
```

**依赖与启动**：

```bash
mkdir -p /opt/signal-hub && cd /opt/signal-hub
npm init -y && npm install ws express jsonwebtoken
# 将 §13.7.1 参考实现保存为 server.js
node server.js
```

**systemd**（`/etc/systemd/system/signal-hub.service`）：

```ini
[Unit]
Description=WebRTC Signal Hub (Mode A)
After=network.target

[Service]
EnvironmentFile=/etc/signal-hub/env
WorkingDirectory=/opt/signal-hub
ExecStart=/usr/bin/node /opt/signal-hub/server.js
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
systemctl daemon-reload
systemctl enable --now signal-hub
```

#### 步骤 6：VPS 验收

```bash
curl -s https://signal.example.com/api/v1/health
# {"ok":true}

# TURN（与 §10 相同）
turnutils_uclient -u test -W "$TURN_SECRET" turn.example.com
```

### 13.7 内网 Server 部署步骤

内网 Server **只需出站**，网关无需映射 443。

#### 步骤 1：环境变量

```bash
# /etc/webrtc-agent/env
SIGNAL_AGENT_URL=wss://signal.example.com/agent
DEVICE_ID=device-001
AGENT_TOKEN=replace-agent-pre-shared-token
TURN_SECRET=REPLACE_WITH_LONG_RANDOM_SECRET
TURN_HOST=turn.example.com
```

#### 步骤 2：Agent 出站客户端

Agent 职责：**维持 WSS 注册** + **把 VPS 转来的信令交给本地 WebRTC 业务**。

```javascript
// agent.js — 内网 Server 上运行
const WebSocket = require('ws');

const URL = process.env.SIGNAL_AGENT_URL;
const DEVICE_ID = process.env.DEVICE_ID;
const AGENT_TOKEN = process.env.AGENT_TOKEN;

let backoff = 1000;

function connect() {
  const ws = new WebSocket(URL);

  ws.on('open', () => {
    backoff = 1000;
    ws.send(JSON.stringify({ type: 'register', deviceId: DEVICE_ID, token: AGENT_TOKEN }));
    setInterval(() => {
      if (ws.readyState === WebSocket.OPEN) {
        ws.send(JSON.stringify({ type: 'ping', deviceId: DEVICE_ID }));
      }
    }, 30000);
  });

  ws.on('message', (raw) => {
    const msg = JSON.parse(raw);
    if (msg.type === 'registered' || msg.type === 'pong') return;
    // 交给本地 WebRTC 业务：join / offer / answer / ice-candidate
    require('./webrtc-handler').onSignaling(msg);
  });

  ws.on('close', () => setTimeout(connect, backoff));
  ws.on('error', () => ws.close());
  ws.on('close', () => { backoff = Math.min(backoff * 2, 60000); });
}

connect();
```

**systemd**：

```ini
[Unit]
Description=WebRTC Agent (outbound WSS)
After=network-online.target

[Service]
EnvironmentFile=/etc/webrtc-agent/env
WorkingDirectory=/opt/webrtc-agent
ExecStart=/usr/bin/node /opt/webrtc-agent/agent.js
Restart=always

[Install]
WantedBy=multi-user.target
```

#### 步骤 3：内网 WebRTC 与 TURN

Server 侧媒体使用 **§3 方案 B** 本地生成 TURN 凭据，指向 VPS coturn：

```javascript
const crypto = require('crypto');

function turnCredentials(userId, ttl = 3600) {
  const expiry = Math.floor(Date.now() / 1000) + ttl;
  const username = `${expiry}:${userId}`;
  const credential = crypto.createHmac('sha1', process.env.TURN_SECRET)
    .update(username).digest('base64');
  const host = process.env.TURN_HOST;
  return {
    iceServers: [
      { urls: `stun:${host}:3478` },
      {
        urls: [
          `turn:${host}:3478?transport=udp`,
          `turn:${host}:3478?transport=tcp`,
          `turns:${host}:5349?transport=tcp`,
        ],
        username,
        credential,
      },
    ],
  };
}
```

收到 Mobile 的 `offer` 后，内网 Server 创建 PeerConnection（`iceServers` 如上），生成 `answer` 经 Agent **回传 VPS → Mobile**。

#### 步骤 4：内网防火墙

内网 Server 仅需 **出站**：

```
允许 OUT:443/tcp   → signal.example.com（Agent WSS）
允许 OUT:3478/udp,tcp 5349/tcp 49152-65535/udp → turn.example.com
```

**无需**入站 443/3478。

#### 步骤 5：内网验收

```bash
systemctl status webrtc-agent   # active, 日志有 registered
# VPS 上信令服务日志应出现 device-001 online
```

### 13.8 Mobile App 部署步骤

#### 步骤 1：固定配置（编译进 App）

```javascript
export const CONFIG = {
  signalWs:  'wss://signal.example.com/ws',
  apiBase:   'https://signal.example.com/api/v1',
  // turn 凭据从 API 获取，host 在 iceServers 内
};
```

#### 步骤 2：登录 → 取 TURN 凭据 → 连 WSS

```javascript
async function prepareCall(roomId, targetDeviceId, userJwt) {
  const credRes = await fetch(
    `${CONFIG.apiBase}/turn-credentials?roomId=${encodeURIComponent(roomId)}`,
    { headers: { Authorization: `Bearer ${userJwt}` } }
  );
  const { iceServers } = await credRes.json();

  const pc = new RTCPeerConnection({ iceServers });

  const ws = new WebSocket(CONFIG.signalWs, {
    headers: { Authorization: `Bearer ${userJwt}` },
  });

  ws.onopen = () => {
    ws.send(JSON.stringify({ type: 'join', roomId, targetDeviceId }));
  };

  ws.onmessage = async (ev) => {
    const msg = JSON.parse(ev.data);
    if (msg.type === 'signal') {
      const { kind, sdp, candidate } = msg.payload;
      if (kind === 'offer') {
        await pc.setRemoteDescription({ type: 'offer', sdp });
        const answer = await pc.createAnswer();
        await pc.setLocalDescription(answer);
        ws.send(JSON.stringify({
          type: 'signal', roomId, targetDeviceId,
          payload: { kind: 'answer', sdp: answer.sdp },
        }));
      } else if (kind === 'ice-candidate' && candidate) {
        await pc.addIceCandidate(candidate);
      }
    }
  };

  return { pc, ws };
}
```

#### 步骤 3：验证 ICE

```javascript
pc.onicecandidate = (e) => {
  if (e.candidate?.candidate?.includes('typ relay')) {
    console.log('TURN OK:', e.candidate.candidate);
  }
};
```

蜂窝网络下应出现 `typ relay`。

### 13.9 VPS 信令服务参考实现（Node.js）

```javascript
// server.js — VPS /opt/signal-hub/server.js
const express = require('express');
const http = require('http');
const crypto = require('crypto');
const jwt = require('jsonwebtoken');
const { WebSocketServer } = require('ws');

const PORT = process.env.PORT || 8080;
const JWT_SECRET = process.env.JWT_SECRET;
const TURN_SECRET = process.env.TURN_SECRET;
const TURN_HOST = process.env.TURN_HOST || 'turn.example.com';
const AGENT_TOKEN = process.env.AGENT_TOKEN;
const ALLOWED = new Set((process.env.ALLOWED_DEVICES || '').split(',').filter(Boolean));

const agents = new Map();   // deviceId -> ws
const clients = new Map();  // ws -> { userId, roomId, targetDeviceId }

const app = express();
app.get('/api/v1/health', (_, res) => res.json({ ok: true }));

app.get('/api/v1/turn-credentials', (req, res) => {
  const userId = authBearer(req);
  if (!userId) return res.status(401).json({ error: 'unauthorized' });
  const ttl = 3600;
  const expiry = Math.floor(Date.now() / 1000) + ttl;
  const username = `${expiry}:${userId}`;
  const credential = crypto.createHmac('sha1', TURN_SECRET).update(username).digest('base64');
  const iceServers = buildIceServers(username, credential);
  res.json({ iceServers, ttl, expiry, username, credential });
});

const server = http.createServer(app);
const wss = new WebSocketServer({ server });

wss.on('connection', (ws, req) => {
  const path = req.url?.split('?')[0] || '';
  if (path === '/agent') return onAgent(ws);
  if (path === '/ws') return onClient(ws, req);
  ws.close();
});

function onAgent(ws) {
  let deviceId = null;
  ws.on('message', (raw) => {
    let msg; try { msg = JSON.parse(raw); } catch { return; }
    if (msg.type === 'register') {
      if (msg.token !== AGENT_TOKEN) return ws.close(4401, 'bad agent token');
      if (!ALLOWED.has(msg.deviceId)) return ws.close(4403, 'device not allowed');
      deviceId = msg.deviceId;
      agents.set(deviceId, ws);
      ws.send(JSON.stringify({ type: 'registered', deviceId, ts: Date.now() }));
      return;
    }
    if (msg.type === 'ping') {
      ws.send(JSON.stringify({ type: 'pong', ts: Date.now() }));
      return;
    }
    // Agent → Mobile 转发
    if (msg.type === 'signal' && msg.targetUserId) {
      for (const [cws, meta] of clients) {
        if (meta.userId === msg.targetUserId && cws.readyState === 1) {
          cws.send(JSON.stringify(msg));
        }
      }
    }
  });
  ws.on('close', () => { if (deviceId) agents.delete(deviceId); });
}

function onClient(ws, req) {
  const userId = authBearer(req);
  if (!userId) return ws.close(4401, 'unauthorized');
  clients.set(ws, { userId });

  ws.on('message', (raw) => {
    let msg; try { msg = JSON.parse(raw); } catch { return; }
    if (msg.type === 'join') {
      const agent = agents.get(msg.targetDeviceId);
      if (!agent) {
        ws.send(JSON.stringify({ type: 'error', code: 'DEVICE_OFFLINE', deviceId: msg.targetDeviceId }));
        return;
      }
      clients.set(ws, { userId, roomId: msg.roomId, targetDeviceId: msg.targetDeviceId });
      agent.send(JSON.stringify({
        type: 'join', roomId: msg.roomId, clientUserId: userId, relayId: crypto.randomUUID(),
      }));
      return;
    }
    if (msg.type === 'signal') {
      const agent = agents.get(msg.targetDeviceId);
      if (agent) agent.send(raw);
    }
  });
  ws.on('close', () => clients.delete(ws));
}

function authBearer(req) {
  const h = req.headers['authorization'] || req.headers['Authorization'] || '';
  const m = h.match(/^Bearer\s+(.+)$/i);
  if (!m) return null;
  try {
    const p = jwt.verify(m[1], JWT_SECRET);
    return p.sub || p.userId;
  } catch { return null; }
}

function buildIceServers(username, credential) {
  return [
    { urls: `stun:${TURN_HOST}:3478` },
    {
      urls: [
        `turn:${TURN_HOST}:3478?transport=udp`,
        `turn:${TURN_HOST}:3478?transport=tcp`,
        `turns:${TURN_HOST}:5349?transport=tcp`,
      ],
      username, credential,
    },
  ];
}

server.listen(PORT, '127.0.0.1', () => console.log(`signal-hub on ${PORT}`));
```

### 13.10 端到端通话时序

```mermaid
sequenceDiagram
    participant M as Mobile
    participant V as VPS 信令
    participant A as 内网 Agent
    participant S as 内网 WebRTC
    participant T as coturn

    A->>V: WSS /agent register
    V-->>A: registered

    M->>V: GET /turn-credentials (JWT)
    V-->>M: iceServers

    M->>V: WSS /ws join room
    V->>A: join (clientUserId)
    A->>S: 通知新会话

    M->>V: signal offer
    V->>A: forward offer
    A->>S: handle offer
    S->>T: TURN Allocate
    M->>T: TURN Allocate

    S->>A: answer + ICE
    A->>V: signal answer
    V->>M: forward answer

    M->>T: SRTP relay
    T->>S: SRTP relay
```

### 13.11 运维与排障

| 现象 | 排查 |
|------|------|
| Mobile WSS 连不上 | `dig signal.example.com`；Caddy 证书；`curl https://signal.example.com/api/v1/health` |
| `DEVICE_OFFLINE` | 内网 `systemctl status webrtc-agent`；VPS 日志 Agent 是否 registered |
| 无 `typ relay` | `TURN_SECRET` 是否与 coturn 一致；`turnutils_uclient -W`；运营商 UDP 3478 |
| 信令通、无媒体 | ICE 日志；双端是否都 Allocate；coturn `relay-ip` |
| 网关重拨后 Mobile 正常、内网断 | Agent 应自动重连；检查内网出站 443 |
| 网关重拨后 Agent 正常、Mobile 正常 | 模式 A 下 **信令不受影响**（均连 VPS） |

**日志建议**

- VPS：Agent register/断开、join 失败、TURN 签发 audit（userId，不含 secret）
- 内网：Agent 重连次数、WebRTC `iceConnectionState` 变化

**备份与密钥**

- `TURN_SECRET`、`JWT_SECRET`、`AGENT_TOKEN` 放 `/etc/.../env`，权限 `600`
- 轮换 `AGENT_TOKEN`：更新 VPS + 内网 env，重启两边服务

### 13.12 部署清单（Checklist）

**VPS**

- [ ] DNS：`signal.example.com`、`turn.example.com` → VPS IP
- [ ] 防火墙：443、3478、5349、49152-65535/udp
- [ ] coturn：`use-auth-secret`，`relay-ip` = VPS 公网 IP
- [ ] Caddy：`signal.example.com` → `127.0.0.1:8080`
- [ ] signal-hub：systemd 运行，health OK
- [ ] `turnutils_uclient -W` 通过

**内网 Server**

- [ ] Agent systemd 运行，日志 `registered`
- [ ] 出站 443 / 3478 放行
- [ ] **无** 443 入站映射、**无** frp、**无** DDNS
- [ ] WebRTC 使用 `turn.example.com` relay

**Mobile App**

- [ ] 只配置域名
- [ ] JWT 登录 + turn-credentials + WSS join 流程通
- [ ] 蜂窝网 `typ relay` 验证

### 13.13 与 §12 其他模式的关系

| 模式 | 何时使用 |
|------|----------|
| **模式 A（本节）** | 默认首选；动态 WAN IP；希望最少组件 |
| 模式 B（frp + Caddy） | 信令代码暂不能改、必须跑在内网 HTTP 服务上 |
| 模式 C（DDNS） | 无 VPS；临时备选 |

完成本节部署后，§12 中的拓扑即落地为：**Mobile 与内网 Server 均只连 VPS 固定域名；网关 WAN IP 对客户端完全透明。**

---

## 14. 相关源码与文档

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

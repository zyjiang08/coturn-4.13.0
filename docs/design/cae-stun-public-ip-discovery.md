# CAE 通过 coturn/STUN 动态发现公网 IP 设计

> 日期：2026-07-20
> 适用范围：`nexartc-cloud-phone-access-engine/` WebRTC server 侧 host candidate 生成

## 1. 背景

当前 CAE 需要优先走 **host → p2p/srflx → TURN relay**。其中 host 直连要求浏览器拿到：

1. CAE 内网地址；
2. CAE 对外公网映射地址；
3. 一个固定且已做端口映射的 UDP 端口。

历史上 `webrtc_public_ip` 采用手工静态填写，但该方式有三个问题：

- 家庭宽带公网 IP 变化后容易失效；
- 运维成本高，调试时经常忘记同步；
- 与“固定端口 + 动态发现公网地址”的真实网络状态不一致。

因此改为：

- `listen_port_h5=50000`
- `webrtc_port_range_begin=50000`
- `webrtc_port_range_end=50000`
- `webrtc_public_ip` 默认留空
- CAE 在启动 WebRTC transport 时，主动向 coturn/STUN 发起 **Binding Request**，动态得到自己的公网映射 IP。

## 2. 目标

### 2.1 主目标

- 不再依赖 `webrtc_public_ip` 手工静态配置；
- 继续保持 host 优先；
- 端口固定为 `50000`，与 `listen_port_h5` 共用同一数字；
- 若公网映射端口不是 `50000`，则不发送公网 host candidate，自动回退到 p2p/srflx 或 relay。

### 2.2 非目标

- 不在本阶段把公网映射端口动态改写到 candidate；
- 不在本阶段支持 IPv6 STUN；
- 不在本阶段移除 `webrtc_public_ip` 兼容字段，现阶段仅保留为 deprecated fallback。

## 3. 接口设计

新增独立接口：

- `nexartc-cloud-phone-access-engine/app/src/main/cpp/cae_service/PublicIpResolver.h`
- `nexartc-cloud-phone-access-engine/app/src/main/cpp/cae_service/PublicIpResolver.cpp`

### 3.1 抽象接口

```cpp
class IPublicIpResolver {
public:
    virtual ~IPublicIpResolver() = default;
    virtual PublicIpResolution Resolve(const std::string& stunServerUrl,
                                       uint16_t localPort,
                                       int timeoutMs) = 0;
};
```

返回值：

```cpp
struct PublicIpResolution {
    std::string ip;
    uint16_t port = 0;
    bool IsValid() const;
};
```

### 3.2 具体实现

`CoturnStunPublicIpResolver` 负责：

1. 解析 `stun:host:port`；
2. 绑定本地 UDP 端口（这里传入 `50000`）；
3. 向 coturn/STUN 发送 Binding Request；
4. 解析 Binding Success Response；
5. 提取 `XOR-MAPPED-ADDRESS` 中的公网 IP 和公网端口。

### 3.3 调度策略

`IPublicIpResolver` 采用 **按需获取**，不是后台周期轮询：

- 由 `WebRtcServerTransport` 在初始化 / 新建 PeerConnection 前触发一次；
- 结果在当前 transport 生命周期内缓存，供后续同一会话复用；
- 只有当会话重建、缓存失效或探测失败时，才再次触发解析；
- 不启动独立定时任务，避免无谓 STUN 流量和日志噪声。

推荐实现语义：

```text
start transport / create peer connection
  └─ if public ip cache empty or stale
        └─ call IPublicIpResolver::Resolve()
```

这样可以保证：

- host candidate 生成前拿到最新公网映射；
- 不会因为周期轮询把 NAT 映射/UDP 状态弄脏；
- 代码路径清晰，便于单元测试与故障回退。

## 4. 集成点

集成位置：

- `nexartc-cloud-phone-access-engine/app/src/main/cpp/cae_service/WebRtcServerTransport.cpp`

调用时机：

- `WebRtcServerTransport::Init()`

流程：

1. 读取 `webrtc_local_ip`；
2. 读取 `webrtc_stun_server`；
3. 使用 `port_range_begin`（默认 `50000`）做 STUN 探测；
4. 若返回 `public_port == 50000`，则记录 `m_resolvedPublicIp`；
5. 若返回端口不一致，则禁用公网 host candidate，仅保留 local host / p2p / relay；
6. 若 STUN 探测失败且 `webrtc_public_ip` 非空，则打印 deprecated warning，并走兼容 fallback。

## 5. 配置约定

建议配置：

```ini
[server]
listen_port_h5=50000

[webrtc]
webrtc_ice_mode=hybrid
webrtc_local_ip=192.168.x.x
webrtc_public_ip=
webrtc_port_range_begin=50000
webrtc_port_range_end=50000
webrtc_stun_server=stun:www.signalling-nexartc.cn:3478
webrtc_turn_host=www.signalling-nexartc.cn
webrtc_turn_port=3478
webrtc_turn_secret=<TURN_SECRET>
```

说明：

- `webrtc_public_ip`：默认留空，由 STUN 动态发现；
- `listen_port_h5=50000`：WSS 监听端口；
- `webrtc_port_range_begin/end=50000`：host candidate 使用同一数字端口；
- 路由器需完成 `50000/UDP` 映射。

## 6. 失败策略

### 6.1 STUN 解析失败

行为：

- 不发送公网 host candidate；
- 浏览器继续走 p2p/srflx；
- 若仍失败，则依赖 TURN relay。

### 6.2 映射端口不等于 50000

行为：

- 认为当前网络不满足“固定映射端口”前提；
- 记录 warning；
- 不注入公网 host candidate；
- 自动回退到 p2p/srflx / relay。

## 7. 单元测试

新增测试：

- `test/turn/test_cae_public_ip_resolver.sh`

覆盖点：

1. `ParseStunEndpoint()` URL 解析；
2. `ParseBindingResponse()` 对 `XOR-MAPPED-ADDRESS` 的解析；
3. 本地 fake STUN server 网络往返；
4. 结果校验为 `198.51.100.7 50000`。

执行：

```bash
bash test/turn/test_cae_public_ip_resolver.sh
```

## 8. 文档联动

以下文档已同步为新口径：

- `nexartc-cloudPhoneAccess-web/docs/WEB_ENVIRONMENT_AND_TEST.md`
- `coturn-4.13.0/docs/design/nexartc-turn-mode-a-implementation.md`
- `coturn-4.13.0/docs/design/nexartc-turn-mode-a-vps-deployment.md`

统一口径：

- 端口：`50000`
- 公网 IP：coturn/STUN 动态发现
- 优先级：host → p2p/srflx → relay

## 9. 实现状态（2026-07-20）

| 项 | 状态 |
|----|------|
| `PublicIpResolver.h/.cpp` | ✅ |
| `WebRtcServerTransport::Init` STUN 探测 + 端口校验 | ✅ |
| `webrtc_public_ip` deprecated fallback | ✅ |
| 固定端口 50000（不再因 TURN 扩到 49152–65535） | ✅ |
| `test/turn/test_cae_public_ip_resolver.sh` | ✅ |
| 设备侧清空静态 `webrtc_public_ip` + 路由器 DNAT 50000 | ⏳ 部署时执行 |

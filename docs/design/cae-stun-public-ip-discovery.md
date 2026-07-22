# CAE 通过 coturn/STUN 动态发现公网 IP 设计

> 日期：2026-07-20（调度策略 2026-07-21 更新）  
> 适用范围：`nexartc-cloud-phone-access-engine/` WebRTC server 侧 host candidate 生成  
> 联调日志路径与四类前缀见 [`nexartc-logging-design.md`](./nexartc-logging-design.md)（CAE：`cae_server_*.log` 搜 `[FLOW] local_ip` / candidate / `public IP`；device：`ICE path`）

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
- CAE 通过独立 STUN 探测任务动态得到自己的公网映射 IP（启动同步一次 + 周期刷新 + 建连失败重探）。

## 2. 目标

### 2.1 主目标

- 不再依赖 `webrtc_public_ip` 手工静态配置；
- 继续保持 host 优先；
- 端口固定为 `50000`，与 `listen_port_h5` 共用同一数字；
- 若公网映射端口不是 `50000`，则不发送公网 host candidate，自动回退到 p2p/srflx 或 relay；
- 公网 IP 变化（拨号重拨）可在合理时间内自动感知，无需重启 CAE。

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

由 `WebRtcServerTransport` 内 **独立后台线程**（`cae-pubip`）调度，而不是仅在首次 `Init` 取一次：

| 触发 | 时机 | reason 日志 |
|------|------|-------------|
| 启动同步 | `Init()` 内立刻 `RefreshPublicIp("init")` | `init` |
| 周期刷新 | `Start()` 后独立线程按间隔等待 | `periodic` |
| 建连失败 | `PeerConnection` → `Failed` | `ice_failed` |
| 收集失败 | gathering complete 且本地候选数为 0 | `gathering_empty` |
| 创建失败 | `CreatePeerConnection` 异常 / 失败 | `pc_create_failed` |

规则：

1. **独立任务**：`StartPublicIpRefreshThread()` / `StopPublicIpRefreshThread()`，与 ICE 会话生命周期解耦；`Stop()` / 析构时 join。
2. **周期间隔**：`webrtc_public_ip_refresh_sec`（默认 **300** 秒）。设为 `0` 则关闭周期轮询，仍保留失败路径按需重探。
3. **失败重探**：ICE Failed / gathering 空候选 / PC 创建失败 → `RequestPublicIpRefresh`，经 condition_variable 唤醒线程。
4. **防抖**：按需请求距上次成功探测不足 **15s** 则跳过，避免失败风暴打满 STUN。
5. **串行化**：`m_publicIpResolveMutex` 保证同一时刻只有一个 Binding 在飞。
6. **端口占用**：固定端口探测失败时，退化为 ephemeral **仅刷新 IP**；若此前已校验过 `mapped_port == 50000`，则保留“可注入公网 host”状态并更新 IP。
7. **`ice_mode=relay` / `force_relay`**：不启动公网 IP 发现（host 注入无意义）。

```text
Init()
  └─ RefreshPublicIp("init")

Start()
  └─ thread cae-pubip
        loop:
          wait(interval) or wake(on_demand)
          RefreshPublicIp(reason)

ICE Failed / gathering_empty / pc_create_failed
  └─ RequestPublicIpRefresh(reason)  // debounce ≥15s
```

这样可以保证：

- 拨号换 IP 后周期内自动更新缓存，下次新会话注入新公网 host；
- 建连失败立刻尝试刷新，缩短“IP 已变但仍用旧缓存”的窗口；
- 无活跃会话时固定端口探测完整校验映射；有会话时尽量不与 ICE 抢包。

## 4. 集成点

集成位置：

- `nexartc-cloud-phone-access-engine/app/src/main/cpp/cae_service/WebRtcServerTransport.cpp`
- 配置：`CaeConfigManage::GetWebRtcPublicIpRefreshSec()` / `CaeConfig.ini`

调用时机：

- `Init()`：同步首次探测
- `Start()` / `Stop()`：启停刷新线程
- ICE / gathering / PC 失败回调：按需唤醒

流程（单次 `RefreshPublicIp`）：

1. 读取 `webrtc_stun_server` 与 `port_range_begin`（默认 `50000`）；
2. 固定端口 STUN Binding；
3. 若 `public_port == discoveryPort`，更新 `m_resolvedPublicIp` 并标记 port validated；
4. 若端口不一致，清空公网 host 注入；
5. 若固定端口探测失败，ephemeral 探测：仅在已 validated 时更新 IP；
6. 若仍失败且 `webrtc_public_ip` 非空，deprecated fallback。

## 5. 配置约定

建议配置：

```ini
[server]
listen_port_h5=50000

[webrtc]
webrtc_ice_mode=host
webrtc_local_ip=192.168.x.x
# 默认 host；需要 TURN fallback 时再显式切 hybrid/relay
webrtc_public_ip=
webrtc_public_ip_refresh_sec=300
webrtc_port_range_begin=50000
webrtc_port_range_end=50000
webrtc_stun_server=stun:www.signalling-nexartc.cn:3478
webrtc_turn_host=www.signalling-nexartc.cn
webrtc_turn_port=3478
webrtc_turn_secret=<TURN_SECRET>
```

说明：

- `webrtc_public_ip`：默认留空，由 STUN 动态发现；
- `webrtc_public_ip_refresh_sec`：周期刷新秒数，默认 300；`0`=仅失败重探；
- `listen_port_h5=50000`：WSS 监听端口；
- `webrtc_port_range_begin/end=50000`：host candidate 使用同一数字端口；
- 路由器需完成 `50000/UDP` 映射。

## 6. 失败策略

### 6.1 STUN 解析失败

行为：

- 保留上次有效缓存（若有），打 `[STAB]` 日志；
- 无缓存时不发送公网 host candidate；
- 浏览器继续走 p2p/srflx / TURN relay；
- 周期或下次失败事件会再次尝试。

### 6.2 映射端口不等于 50000

行为：

- 认为当前网络不满足“固定映射端口”前提；
- 记录 warning；
- 不注入公网 host candidate；
- 自动回退到 p2p/srflx / relay。

### 6.3 建连失败后的刷新

行为：

- `ice_failed` / `gathering_empty` / `pc_create_failed` 触发按需重探；
- 新结果仅影响**后续**会话的 host candidate；当前已 Failed 的 PC 不会原地改写。

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
- 公网 IP：coturn/STUN 动态发现（周期 + 失败重探）
- 优先级：host → p2p/srflx → relay

## 9. 实现状态

| 项 | 状态 |
|----|------|
| `PublicIpResolver.h/.cpp` | ✅ |
| `WebRtcServerTransport::Init` 首次 STUN + 端口校验 | ✅ |
| 独立 `cae-pubip` 周期刷新线程 | ✅ |
| ICE Failed / gathering 空 / PC 创建失败按需重探 | ✅ |
| `webrtc_public_ip_refresh_sec`（默认 300） | ✅ |
| `webrtc_public_ip` deprecated fallback | ✅ |
| 固定端口 50000（不再因 TURN 扩到 49152–65535） | ✅ |
| `test/turn/test_cae_public_ip_resolver.sh` | ✅ |
| 设备侧清空静态 `webrtc_public_ip` + 路由器 DNAT 50000 | ⏳ 部署时执行 |

# nexartc Web GPS/方向传感器 → CAE → 高德导航现状评估

> 评估日期：2026-07-24  
> 修复更新：2026-07-24（已落地 Web、CAE C++、Android root 后端的可实施项）  
> 评估范围：Web 端真机浏览器采集 GPS、设备方向及相关传感器，经 WebRTC 传输到 CAE，再注入 Android 并运行高德地图  
> 评估性质：基于当前工作树的代码 review 与本地构建；本轮未向真机注入 GPS、未启动高德，也未修改设备运行状态

## 1. 结论摘要

当前代码具备一条可复用的基础链路，可以完成 root 真机上的固定点定位演示，并有条件地支持移动 GPS 的 PoC：

```text
浏览器 Geolocation / DeviceMotion / DeviceOrientation
        → WebRTC control DataChannel
        → CaeConnectionAgent
        → VMI SENSOR / LOCATION
        → VirtualDeviceManager
        → Android TestProvider / Location.bearing
        → 地图 App
```

本轮已修复方向类型、浏览器授权、采样频率、provider 错误传播、输入校验、状态清理和单 owner 隔离等可在现有架构内完成的问题。当前仍不能宣称“端到端可靠支持高德实时导航”，剩余边界是：

1. Android 端仍未发布真正的 `SensorManager`/sensor HAL 事件，只把远端 heading 映射到 `Location.bearing`。
2. nonroot 变体仍明确不支持系统 GPS mock。
3. 普通网页在后台、锁屏或被系统冻结后不能保证持续采集。
4. WGS-84/GCJ-02、gps/network/fused provider 与当前高德版本尚未完成实机 A/B 验收。
5. 协议已有服务端拒绝码和日志，但尚无浏览器可消费的逐包 ACK、sequence/session id。
6. Kiosk 可以启动指定 App，但当前没有高德目的地 Intent、自动开始导航或路线回放状态机。

### 1.1 能力评级

| 能力 | 当前评级 | 说明 |
|---|---|---|
| Web GPS 采集 | PoC 可用 | 1 Hz 上限、约 3 m 去抖、5 s 保活发送、范围校验和前后台生命周期已接线 |
| Web 设备方向 | PoC 可用 | 10 Hz 上限，iOS compass/absolute orientation/屏幕旋转已处理；真北语义仍需实机校准 |
| WebRTC 传输 | 可复用 | control DataChannel 路径已存在；TURN 只负责连通性 |
| root GPS mock | PoC 可用 | TestProvider 错误可回传到 CAE，输入/时效校验已补；仍需当前 Android/高德实机回归 |
| nonroot GPS mock | 不支持 | 代码明确返回不支持 |
| 真正 Android SensorManager 注入 | 未实现 | 当前只更新内部 compass |
| 打开高德 | 可配置 | 通用 Kiosk 已有，但默认关闭 |
| 自动进入路线导航 | 未实现 | 没有高德 URI/Intent 或目的地业务协议 |
| 实时路线移动 | 未验收 | 历史报告只证明固定点显示 |
| GPX/路线模拟 | 未实现 | 手动面板只能重复固定坐标 |

## 2. 当前代码链路

### 2.1 Web 端

- 消息类型在 `nexartc-cloudPhoneAccess-web/device/src/protocol.ts:15-28` 定义：`SENSOR=23`、`GPS_LOCATION=24`。
- GPS 帧为 `8B StreamMsgHead + 16B MSG_HEADER + 文本字段`，字段包括 longitude、latitude、altitude、speed、bearing、accuracy、timestamp（`protocol.ts:166-200`）。
- Sensor 帧为 `8B StreamMsgHead + 16B MSG_HEADER + x:y:z:accuracy`（`protocol.ts:202-229`）。
- Web 采集逻辑位于 `nexartc-cloudPhoneAccess-web/device/src/main.ts`：GPS 最多 1 Hz、约 `0.00003°` 去抖，并保证最长 5 秒发送一次；motion/orientation 最多 10 Hz。
- GPS 帧的 `bearing` 仍只承载 `GeolocationCoordinates.heading`（运动 course）；设备朝向独立用 sensor type `3` 的 `[azimuth, pitch, roll]` 发送，不再混入 GPS course。
- `devicemotion` 使用 type `1`（accelerometer），优先无重力 `acceleration`；`deviceorientation` 使用 type `3`，优先 iOS `webkitCompassHeading`，否则转换 absolute alpha 并补偿屏幕旋转。
- Connect 用户手势会调用 iOS motion/orientation `requestPermission()`；静态服务设置同源 `Permissions-Policy`。页面进入后台时停止 watch，回到前台且 control channel 仍打开时恢复。
- `webrtc.ts` 现在等待 DataChannel 的 `open` 事件后才启动采集；control channel 关闭会立即停止 Geolocation 和 sensor listener，消除了首包 race 和断链泄漏。

### 2.2 DataChannel 和 CAE

服务端已创建专用通道：

- `control`：ordered/reliable；
- `gps`：ordered/reliable；
- `sensor`：unordered、`maxRetransmits=0`。

定义见 `nexartc-cloud-phone-access-engine/app/src/main/cpp/cae_service/WebRtcServerTransport.cpp:135-151`。当前 Web 仍将 GPS 和 sensor 都发到 `control`，功能上可工作，但输入/传感器共享同一条可靠通道，可能出现队头阻塞和旧数据排队。

`transport/src/ProtocolSession.cpp:120-187` 对 WebRTC 专用通道按 ChannelId 分发；因此如果以后切换到 `gps`/`sensor` 通道，Web 不能直接把当前完整的 8 字节外层帧原样发送，必须明确“专用通道传 raw payload、control 通道传外层 StreamMsgHead”的协议差异。

CAE 控制通道在 `CaeConnectionAgent.cpp:924-995` 解析 type 23/24，随后：

- sensor → `VmiDeviceSend(SENSOR, ...)`；
- location → `HandleLocationMsg()` → `VmiDeviceSend(LOCATION, ...)`。

本轮已在这一路径增加：

- 外层 `StreamMsgHead` magic、checksum、声明 payload 长度校验；control 通用上限 16 MiB，sensor/location 收紧为 64 KiB；
- 内层 `MSG_HEADER` 最小长度、声明长度和 optType 校验；
- sensor/location 共用的单 owner lease：首个发送者持有 30 秒租约，持续发送会续租，断开立即释放，租约超时才允许另一连接接管；owner 新建、接管或释放时通过 JNI 清空上一会话 GPS/compass 缓存；
- `VmiDeviceSend()`/`HandleLocationMsg()` 真实返回值传播和 rejected/provider error 日志。

因此 `OnSensorData()`/`OnLocationData()` 不再忽略 `conn_id`。当前 owner 是进程内连接级租约，尚未升级为持久 session id 或带鉴权的业务 owner。

### 2.3 Android GPS 与传感器

`VirtualDeviceManager.java` 的 root 路径会：

1. 设置 mock location/AppOps；
2. 关闭 Wi-Fi/BLE 扫描并修改 `location_mode`；
3. 通过隐藏 `ILocationManager` 添加 `gps`/`network` TestProvider；
4. 调用 `setTestProviderLocation()`。

GPS 注入和文本解析位于 `VirtualDeviceManager.java`。本轮增加 VMI header/optType、数值范围、finite、最大帧、client timestamp 过期/未来窗口校验；provider 创建与注入失败会返回失败，不再吞错。

Android 后端识别以下 sensor type：

| Android type | 含义 |
|---:|---|
| 1 | accelerometer |
| 2 | magnetic field |
| 3 | orientation |
| 4 | gyroscope |
| 11 | rotation vector |
| 15 | game rotation vector |
| 20 | geomagnetic rotation vector |

`handleSensorData()` 最终仍通过 `updateCompassFromSensor()` 更新内部 compass，没有把事件发布到 Android `SensorManager` 或 sensor HAL。为改善当前 PoC，低速/静止时 heading 变化会在 1 Hz provider 上限内触发最新 Location bearing 注入；移动速度达到阈值后仍优先 GPS course。

### 2.4 root/nonroot

`app/src/nonroot/java/com/nexartc/cae/media/LimitedVirtualDeviceBackend.java:47-104` 对 location 的 init/enable/send 保持兼容返回，但 `enableGps()` 和 `injectGpsLocation()` 明确返回失败。因此系统级 GPS mock 必须使用 root/特权环境。

## 3. 关键问题与风险

### 3.1 P0：方向协议错误——已修复现有 PoC 路径

Web 已停止使用无效 type `0`：线性加速度使用 type `1`，方向使用 type `3`，VMI `devType` 为 `3`。方向 payload 明确定义为 `[azimuth, pitch, roll]`，Android 只消费第一项作为 heading。Web 优先使用 iOS `webkitCompassHeading`；absolute alpha 会转换为顺时针方位角并补偿屏幕旋转。

剩余限制：type `3` 仍是兼容现有 CAE 的过渡编码，不等价于 Android 原生 orientation sensor；真北/磁北差、厂商浏览器轴定义仍需 Android Chrome 与 iOS Safari 实机校准。若协议继续演进，建议新增显式 `deviceHeading` 字段而不是长期复用原生 sensor type。

### 3.2 P0：浏览器权限与运行时限制——部分修复

Connect 按钮的用户手势现在会调用 iOS `DeviceMotionEvent`/`DeviceOrientationEvent.requestPermission()`；服务端响应增加同源 geolocation/accelerometer/gyroscope/magnetometer `Permissions-Policy`。页面进入后台会主动停止采集，回前台且 control channel 存活时恢复；断链也会停止 watch/listener。

仍有不可由网页代码消除的限制：必须使用受信任 HTTPS，设备锁屏、浏览器冻结或系统回收后不能承诺后台持续采集。因此“页面保持前台”仍是产品约束；若要求锁屏导航采集，应使用原生客户端或 PWA/原生容器的受控后台能力。

### 3.3 P0：GPS course 与 device heading 混用——已修复

Android 后端现在按以下策略选择方向：

- speed 达到移动阈值时保留 GPS course；
- 静止/低速时才使用新鲜 device heading；
- 仅在 `force_gps_bearing=1` 时允许 heading 覆盖移动 course；
- 浏览器 course 不可用时发送 `-1`，Android 在推导出 course/heading 前不伪造北向 bearing；
- 不再为获得 bearing 强制最小 `3 m/s`，静止 Location 可以只带 bearing 而不伪造速度。

剩余限制是高德是否在静止时采用 `Location.bearing` 尚未实机确认；真正控制其 `SensorManager` 箭头仍需要 framework/HAL 级虚拟传感器。

### 3.4 P0：刷新、频率和方向更新不足——已修复代码侧节流

当前频率策略为：Web GPS 最多 1 Hz、约 3 m 去抖且最长 5 秒保活；heading/motion 最多 10 Hz；Android provider 最多 1 Hz、移动门槛 1 m。低速/静止 heading 更新会触发最新 GPS Location 的 bearing 刷新，移动时不会覆盖 course。

`enable_gps_refresh=0` 现在确实不启动刷新线程，运行时切换也会启停线程。该线程仍以 10 秒做无新客户端数据时的保活/短时 dead reckoning，而正常前台数据走 1 Hz provider 路径。1 Hz 是否为当前高德版本的最佳频率仍需实机性能、路线吸附和耗电测试。

### 3.5 P0：错误处理、输入校验和 ACK——输入校验已修复，业务 ACK 待补

已实现：

- 外层 magic/checksum、声明长度、实际长度校验，以及 control 16 MiB、sensor/location 64 KiB 分类型上限；
- 内层 `MSG_HEADER` 长度/optType 校验，避免短包预读；
- 经纬度、海拔、speed、bearing、accuracy 的 finite/range 校验；
- GPS client timestamp 旧 120 秒、未来 30 秒窗口校验；
- sensor type、值数量、finite 和 accuracy 校验；
- TestProvider 创建、启用、反射注入失败返回；
- C++ 传播 `VmiDeviceSend()`/`HandleLocationMsg()` 的实际结果并记录 rejected/provider error；
- nonroot 继续返回明确的不支持错误。

尚未实现浏览器可消费的 `accepted/rejected/provider_error/owner_conflict` wire ACK，也没有每包 sequence 和 sensor timestamp。GPS 旧包可按 timestamp 丢弃；sensor 在迁移到 unordered 专用通道或扩展带版本的 payload 前，仍可能在可靠 control channel 中排队。

### 3.6 P0：多客户端与重连污染——核心覆盖已修复

CAE 已增加进程内单导航 owner lease：首个 sensor/location 发送连接获取 30 秒租约，持续发送续租，其他连接被拒绝；连接断开立即释放，异常断链时租约超时允许接管。owner 创建、接管或释放都会立即清空 Java 侧坐标、速度、bearing、compass、accel/magnet 和计数，避免共享虚拟设备继续刷新上一会话路线；全部虚拟设备关闭时还会恢复原 location mode、Wi-Fi/BLE scan 及 mock_location 设置。

剩余限制：owner 仍以 transport `conn_id` 标识，没有持久 session id、用户身份或 UI 上的“申请/释放控制权”反馈；进程崩溃后的外部系统设置恢复仍需 Supervisor 启动清理兜底。

### 3.7 P1：坐标系和高德 provider 语义未定——未修复，必须实机决策

当前代码可选 WGS-84 → GCJ-02，资产配置仍默认开启。Android GPS provider 通常承载 WGS-84，而高德也可能自行转换；未经 A/B 测试直接转换存在二次偏移风险。必须在当前高德版本上对北京/上海/境外点验证：

- 高德实际读取 gps、network 还是 fused；
- 是否发生二次坐标转换；
- 是否使用 `Location.hasBearing()`；
- 是否接受 TestProvider/mock 标记；
- gps/network 两个 provider 是否竞争。

在实测结论前，不应把 `enable_gcj02_transform=1` 固化为跨设备生产默认值。

### 3.8 P1：隐私和权限——默认日志已脱敏，治理待补

Web 和 Java 的正常日志已不再输出精确经纬度、原始 GPS payload 或传感器数值；精确坐标诊断开关默认关闭。手动 GPS 对话框仍会在当前 UI 显示操作者输入的坐标，这是功能所需且不进入普通日志。

剩余工作是对诊断开关、日志导出、保留期限和访问审计形成产品级权限策略；源码中的诊断能力不能仅靠常量作为最终授权边界。

## 4. 高德启动与导航范围

### 4.1 已有能力

`CaeKioskManager.cpp:60-84` 支持：

- 仅配置 package 时用 `monkey` 启动默认 launcher；
- 配置 activity 时用 `am start -n` 启动指定组件。

当前配置 `app/src/main/assets/config/CaeConfig.ini:173-187` 默认 `enable_kiosk=0` 且 package/activity 为空。

只读检查的 MI 9 上已安装：

```text
package: com.autonavi.minimap
activity: com.autonavi.map.activity.SplashActivity
versionName: 16.21.0.2010
```

这证明可以配置启动高德，但不证明当前注入链路已通过导航验收。

### 4.2 当前没有的能力

- 没有高德包名/Activity 硬编码；
- 没有自动设置起点/终点；
- 没有 `androidamap://route` 或等价 Intent；
- 没有“连接后进入导航页面”的业务状态机；
- 没有路线回放/插值器。

因此“打开高德”和“开始导航”应拆成两个功能：一次性启动 App，以及显式传入目的地/路线参数。Kiosk 只适合需要单应用锁定的场景。

## 5. design 目录逐项审查

`coturn-4.13.0/docs/design/` 的文档主要是网络、部署、媒体和运维设计，没有完整 GPS/传感器业务规范。

| 文档 | 与本功能的关系 |
|---|---|
| `cae-stun-public-ip-discovery.md` | 解决公网 IP、固定端口和 host candidate；不定义 GPS payload |
| `cae-supervisor-remote-admin-design.md` | 可用于 CAE 崩溃自愈，但未规定重启时清理 mock 状态 |
| `nexartc-install-deploy-guide.md` | 明确 Mode A 主要使用 root APK；缺少 GPS/高德验收步骤 |
| `nexartc-logging-design.md` | 日志路径和 `[FLOW]/[FUNC]/[STAB]/[EXC]` 规范完整；本轮代码已默认脱敏 GPS，文档仍需补访问/保留/审计策略 |
| `nexartc-turn-mode-a-implementation.md` | TURN/Hub/信令落地方案；TURN 只提供连通性，不承载传感器语义 |
| `nexartc-turn-mode-a-vps-deployment.md` | VPS 联调记录；文档中存在明文 `STREAM_TOKEN`，必须轮换并清理历史 |
| `nexartc-webrtc-ice-udp-mux-logical-separation.md` | 解决多 PeerConnection 的 mux 生命周期；本轮另在 CAE 增加 sensor/location owner lease |
| `turn-rest-api-signaling.md` | TURN REST HMAC/TTL 设计；正确要求 shared secret 不下发浏览器，但不负责 GPS 鉴权 |
| `nexartc-cae-scene-change-blur-qp-design.md` | 只间接影响高德画面清晰度 |
| `nexartc-scene-change-blur-phase2-codec-resolution.md` | 只间接影响远程地图操作体验 |
| `nexartc-chrome-h264-level-diagnosis-and-remediation.md` | 只影响浏览器观看高德画面的兼容性 |
| `nexartc-web-cae-h265-support-design.md` | 编码协商，与传感器业务无直接关系 |
| `RTC_Token.zip` | 第三方 RTC token 示例，与 coturn REST 和 GPS 无直接关系，建议移到 examples/third_party 并补许可证说明 |

### 5.1 相关历史文档的漂移

仓库其他目录中有两份更直接的 GPS 文档，但不能作为当前 WebRTC 方案的验收依据：

- `docs/地图导航_GPS与传感器数据流详解.md` 面向 Android CAS + QUIC（文档自身注明适用模块），使用旧 stream ID 和 Android 采集实现；
- `docs/cae_analysis/05_realtime_gps_sensor_injection.md` 声称已验证，但验证范围是手动注入东京/北京固定点，高德显示目标城市；文档同时承认没有 SensorManager 注入，也将路线回放列为后续工作。

当前正式 Web 入口是 Vite `/device/`。旧 `webrtc_cloud_device.html` 在当前 VPS 入口返回 404，文档中的旧部署路径需要统一清理。

## 6. 实施状态与后续顺序

### Phase 0：root GPS PoC

- [x] 浏览器用户手势授权、absolute/iOS heading 和生命周期处理。
- [x] course 与 device heading 分离，不再使用 type `0`。
- [x] CAE 外层/内层长度、数值、类型和 GPS 时效校验。
- [x] provider 错误传播、单 owner lease、断开清理和系统设置恢复。
- [x] 1 Hz GPS/provider、10 Hz heading，以及静止 heading 的 Location bearing 刷新。
- [x] 普通日志默认移除精确 GPS/sensor 数据。
- [ ] UI 明确显示 nonroot 不支持系统 GPS mock。
- [ ] 浏览器 wire ACK、sequence/session id 和 owner 冲突反馈。
- [ ] MI 9 + 高德 16.21 坐标系、provider、mock 标记 A/B 实测。
- [ ] 一次性启动高德与目的地 Intent；暂不把 Kiosk 与路线状态机混合。

### Phase 1：实时导航可靠性

1. 完成前台 GPS/heading 自适应采样与最新值合并；GPS 已能丢弃旧包，sensor 需增加版本化 timestamp/sequence。
2. GPS 使用可靠 ordered channel，heading 使用专用低延迟 channel；明确专用 channel 不带外层 8B header。
3. 增加路线/GPX 插值回放、速度和 course 生成器。
4. 对高德启动、路线设置、导航状态和异常做业务 ACK。
5. 在现有默认脱敏基础上补齐诊断授权、采样、导出审计和保留期限。

### Phase 2：真正系统传感器

如果要求高德通过 `SensorManager` 读取远端磁力计、陀螺仪或旋转矢量，需要评估 Android framework/HAL 虚拟传感器、系统权限和 API 版本兼容性；当前 Java compass 映射不能替代该能力。

## 7. 本轮构建验证

本轮只验证代码与本地构建，不等同于高德实机验收：

```text
cd nexartc-cloudPhoneAccess-web/device
npm run build

cd ../server
npm run build

cd ../../nexartc-cloud-phone-access-engine
./gradlew :app:externalNativeBuildRootDebug \
  :app:compileRootDebugJavaWithJavac --no-daemon --console=plain
```

三项均成功；C++/Java 构建仅出现既有依赖与 deprecated API warning。Web 与 engine 子仓库 `git diff --check` 通过。

## 8. 实机验收清单

### 浏览器

- Chrome Android 真机：位置、加速度、方向授权均成功；页面前后台切换后可恢复。
- Safari iOS：用户手势授权成功，横竖屏旋转后 heading 仍正确。
- 受信任 HTTPS 域名，不能依赖证书不匹配的 IP 页面。
- 授权诊断模式可关联 sequence、时间戳、course、deviceHeading 和屏幕方向，普通日志保持脱敏。

### 链路和 CAE

- Hub、CAE、TURN 日志可按 session/owner 关联。
- control/gps/sensor 三种通道的 framing 明确，异常长度包不会导致崩溃。
- Web 能看到 provider 成功、nonroot 不支持、过期包和 owner 冲突的明确结果。
- 断开/重连后不再注入上一个会话坐标或方向。

### Android/高德

- `dumpsys location` 确认 gps/network/fused 的实际 provider 和更新时间。
- 对 WGS-84/GCJ-02 各做一次已知点和道路吸附测试，排除二次转换。
- 高德 16.21 在静止旋转、步行、车辆移动、急转弯场景分别验证箭头和路线。
- 验证 mock flag、AppOps、Wi-Fi/BLE 恢复以及高德重启后的行为。
- 验证 package 启动与目的地 Intent；仅启动 SplashActivity 不算导航验收。

### 多客户端和网络

- 第二个 viewer 不能覆盖导航 owner 的 GPS/方向。
- host、hybrid、relay 三种 ICE 模式分别验证 DataChannel 数据完整性。
- TURN 断网、浏览器切后台、CAE 重启后，状态机能安全恢复。

## 9. 最终判断

现有代码已补齐 root 真机 GPS/heading PoC 所需的主要代码侧防线：方向协议、权限、采样/注入频率、provider 错误、恶意/陈旧 GPS 包、owner 隔离、断开清理和日志脱敏。下一步可以进入真机固定点、静止旋转、步行和车辆路线验收。

但在完成 WGS-84/GCJ-02 与 provider A/B、浏览器业务 ACK/sequence、高德目的地/路线状态机以及必要时的真正 SensorManager/HAL 注入前，仍不应把“Web 真机传感器驱动高德实时导航”标记为生产完成。

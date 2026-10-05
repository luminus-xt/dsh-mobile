## 背景

在 iOS 客户端经 WebSocket 连接 DSH 移动网关的场景下，长期观察到超时与「连接失败」。逐项排查后确认其中**两处**连接生命周期缺陷与服务端日志能一一对应。本 PR 只改 `GatewayClient.swift`（+29/-2），不涉及 UI 与协议。

## FIX-1：ping 失败零容忍拆链

`startHeartbeat` 里 `socket.sendPing` 的回调把**任何一次** error 都交给 `handleFailure`。实测一次网络抖动就足以同时杀掉 control 与 conversation 两条通道：

- `sendPing` 返回 `-1001`（`NSURLErrorTimedOut`）时，`handleFailure` 只挡 `-999`（`NSURLErrorCancelled`），**并不挡 `-1001`**
- 于是走 `fail()` → `socket.cancel(with: .goingAway)`，**不发送 close 帧**
- 服务端只能看到 `code=1006 reason=-`，无法判断是谁先断开的
- 两条通道各自心跳，恰好在同一毫秒被拆，表现为**成对断开**

改为与服务端 keepalive 相同的策略：容忍连续失败 N 次再判定链路死亡。

**pong 正常回来即把计数清零** —— 只在**连续**失败时累积，否则分散在几小时内的瞬时抖动会攒够阈值，反而误杀连接。

取 `PING_FAILURE_LIMIT = 3`（≈90s），与服务端 `KEEPALIVE_MISS_LIMIT` 同量级；完全连不上时，首条 `hello` 的 15s 超时仍然兜底，不会掩盖真实的连接失败。

**现场实测：连接寿命从 3~5 秒提升到 43 秒以上。**

## FIX-2：`scenePhase` 重入取消握手中的 socket

`applicationDidBecomeActive` 中 `scenePhase == .active` 可能连续投递多次，每次多余投递都会走 `beginConnection()`，而它第一行是 `disconnect(reconnect: false)`，把**仍在握手中**的 socket 取消掉 → `NSPOSIXErrorDomain 53`，网关侧表现为连接后数十毫秒即 `code=1006`。

已在握手中时直接跳过；真正卡住的尝试仍由 `startConnectionTimeout` 兜底，所以不会掩盖超时。

## 备注

现场长期运行的是自签包；本 PR 无任何诊断或埋点代码。

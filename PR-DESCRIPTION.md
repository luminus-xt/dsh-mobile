## 背景

在 iOS 客户端经 WebSocket 连接 DSH 移动网关的场景下，长期观察到超时与「连接失败」。逐项排查后确认其中**两处**连接生命周期缺陷与服务端日志能一一对应。本 PR 只改三个文件，不涉及 UI 布局与协议，不涉及 UI 与协议。

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

---

## FIX-3：长按任意消息可复制正文

复制按钮是**有**的（`copyButton` + `UIPasteboard` 均已实现），但显示条件很窄：

```swift
// ConversationView.swift:2623-2639
// Agent 消息只有在「下一条正式回复之前是最后一条」时才显示复制按钮
result[index].showsCopyButton = nextConversationalMessage != .assistant
```

这是刻意设计（匹配 WebUI 只在最终答案放操作、避免过程叙述被按钮干扰），
但副作用是**多数助手消息根本没有复制途径** —— 想复制中间某段分析结论时无从下手。

在 collection view 层面挂 `UILongPressGestureRecognizer`，一处代码覆盖全部三种行
（流式回复、用户消息、MarkdownUI 自绘行）：

- 行**源文本**经新增的 `ConversationViewportEntry.copyableText` 传递 ——
  MarkdownUI 行只有 `AnyView`，文本取不到，必须显式带上
- 复制后弹「已复制」提示
- `cancelsTouchesInView = false`，不干扰行内链接、复制按钮与折叠手势
- 长按与滚动天然不冲突

同时把 `StreamingAssistantCell` 的 `textView.isSelectable` 置 true：只读文本视图开启选择
零成本，流式行即刻获得系统选区（该属性原本是主动关闭的）。

**刻意不改的**：按钮的显示条件保持原样。它匹配 WebUI 的动作位惯例，视觉噪音是有意的
取舍；长按作为通用补充，两者互补而非互相取代。


## 备注

现场长期运行的是自签包；本 PR 无任何诊断或埋点代码。

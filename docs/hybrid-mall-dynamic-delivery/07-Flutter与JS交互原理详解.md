# Flutter与JS交互原理详解

## 1. 核心原理概述

### 1.1 底层技术基础

Flutter与JS的交互本质上是**通过WebView作为桥梁**，利用WebView提供的原生能力实现跨语言通信。

```
┌─────────────────────────────────────────────┐
│         Flutter Layer (Dart)                │
│                                             │
│  ┌──────────────────────────────────┐      │
│  │    WebViewController             │      │
│  │  - runJavaScript()               │      │
│  │  - addJavaScriptChannel()        │      │
│  └──────────────────────────────────┘      │
└─────────────────────────────────────────────┘
                    ▲
                    │ Platform Channel
                    ▼
┌─────────────────────────────────────────────┐
│      Native WebView (Platform-specific)     │
│                                             │
│  iOS: WKWebView                             │
│  - evaluateJavaScript()                     │
│  - WKScriptMessageHandler                   │
│                                             │
│  Android: WebView                           │
│  - evaluateJavascript()                     │
│  - @JavascriptInterface                     │
└─────────────────────────────────────────────┘
                    ▲
                    │ JavaScriptCore / V8
                    ▼
┌─────────────────────────────────────────────┐
│         JavaScript Layer (H5)               │
│                                             │
│  window.NativeBridge.postMessage()          │
│  window.__NATIVE_BRIDGE_CALLBACK__()        │
└─────────────────────────────────────────────┘
```

### 1.2 关键机制

**Flutter不直接与JS通信**，而是：
1. Flutter通过Platform Channel调用原生WebView API
2. 原生WebView执行JavaScript代码
3. JS通过WebView的通道回传消息给原生
4. 原生再通过Platform Channel返回给Flutter

**核心API**：
- `runJavaScript()`: Flutter主动执行JS代码
- `addJavaScriptChannel()`: 注册JS调用通道

## 2. 通信方式详解

### 2.1 Flutter调用JS（Native → JS）

**原理**：使用WebView的`evaluateJavaScript`能力

```dart
// Flutter端
await controller.runJavaScript(
  'window.__NATIVE_BRIDGE_CALLBACK__($responseJson)'
);
```

**底层执行流程**：
```
1. Flutter调用 runJavaScript()
   ↓
2. 通过MethodChannel传递到Platform层
   methodChannel.invokeMethod('runJavaScript', jsCode)
   ↓
3. iOS: WKWebView.evaluateJavaScript(_:completionHandler:)
   Android: WebView.evaluateJavascript(_:resultCallback:)
   ↓
4. WebView的JavaScript引擎执行代码
   ↓
5. JS代码执行完成（可选返回值）
   ↓
6. 结果通过MethodChannel回传给Flutter
```

**特点**：
- ✅ 单向通信，Flutter主动发起
- ✅ 可以获取JS执行结果（异步）
- ✅ 可以传递复杂数据（JSON序列化）
- ⚠️ 执行在WebView的JS线程，不阻塞Flutter主线程

### 2.2 JS调用Flutter（JS → Native）

**原理**：使用JavaScriptChannel机制

#### Step 1: Flutter注册通道

```dart
controller.addJavaScriptChannel(
  'NativeBridge',
  onMessageReceived: (JavaScriptMessage message) {
    // 处理来自JS的消息
    print('Received: ${message.message}');
  },
);
```

**底层注册流程**：
```
1. Flutter调用 addJavaScriptChannel('NativeBridge', handler)
   ↓
2. 通过Platform Channel通知原生层
   ↓
3. iOS: 注册 WKScriptMessageHandler
   userContentController.add(handler, name: "NativeBridge")
   ↓
4. Android: 注入JavascriptInterface
   @JavascriptInterface
   public void postMessage(String message) { }
   ↓
5. 在WebView的全局对象上创建可调用接口
   window.NativeBridge = { postMessage: function() {...} }
```

#### Step 2: JS调用通道

```javascript
// H5端
window.NativeBridge.postMessage(
  JSON.stringify({ method: 'getUserInfo', params: {} })
);
```

**底层调用流程**：
```
1. JS执行 window.NativeBridge.postMessage(data)
   ↓
2. WebView拦截此调用，触发原生handler
   iOS: userContentController(_:didReceive:)
   Android: @JavascriptInterface标注的方法
   ↓
3. 原生handler通过MethodChannel发送到Flutter
   ↓
4. Flutter的onMessageReceived回调被触发
   ↓
5. Flutter处理消息并执行业务逻辑
```

**特点**：
- ✅ 单向通信，JS主动发起
- ✅ 消息是字符串（需手动序列化/反序列化）
- ⚠️ 只能传递String，不能直接传递对象
- ⚠️ 同步调用，JS会等待返回（但不建议耗时操作）

### 2.3 双向通信的实现

要实现真正的双向通信和回调，需要**自行设计协议**：

```
JS调用Native并期待回调：
1. JS生成唯一ID（如 cb_12345）
2. JS通过postMessage发送消息，包含ID
3. JS注册回调函数到本地Map: callbacks[cb_12345] = callback
4. Native收到消息，处理业务逻辑
5. Native通过runJavaScript调用JS回调函数
   window.__CALLBACK__(cb_12345, result)
6. JS执行回调，从Map中取出callback并执行
7. JS清理已完成的回调
```

**关键点**：
- ID必须唯一（时间戳 + 递增序号）
- 需要超时机制（避免回调永不返回）
- 需要错误处理（消息丢失、解析失败）

## 3. 数据传递机制

### 3.1 数据类型限制

**JavaScriptChannel只支持String**：
```
支持：String
不支持：Number, Boolean, Object, Array（需转JSON）
```

**解决方案**：JSON序列化
```javascript
// JS端发送
const data = { userId: 123, name: 'test' };
window.NativeBridge.postMessage(JSON.stringify(data));
```

```dart
// Flutter端接收
onMessageReceived: (msg) {
  final data = json.decode(msg.message);
  print(data['userId']); // 123
}
```

### 3.2 大数据传输

**性能考虑**：
- 每次通信都涉及序列化/反序列化
- 大数据会阻塞JS线程和Native线程
- 建议单次消息 < 1MB

**优化策略**：
```
1. 批量处理（积累多条消息一次发送）
2. 数据压缩（gzip压缩JSON）
3. 分片传输（大数据分多次传输）
4. 只传ID（大数据存本地，只传引用）
```

### 3.3 消息队列

为避免消息丢失和并发问题：

```
Flutter端维护发送队列：
┌──────────────┐
│ Message 1    │ → 发送中
├──────────────┤
│ Message 2    │ → 等待
├──────────────┤
│ Message 3    │ → 等待
└──────────────┘

JS端维护接收队列：
┌──────────────┐
│ Handler 1    │ → 处理中
├──────────────┤
│ Handler 2    │ → 等待
└──────────────┘
```

## 4. 线程模型

### 4.1 执行线程

**Flutter端**：
- `runJavaScript()`: 在Platform线程执行，不阻塞UI
- `onMessageReceived`: 在Platform线程回调

**JavaScript端**：
- 所有JS代码在WebView的JS线程（单线程）
- 即使Native异步返回，JS执行也是串行的

### 4.2 异步处理

**关键点**：
```
1. Flutter调用JS是异步的
   await controller.runJavaScript() // 返回Future

2. JS调用Native看似同步，实际Native处理是异步的
   window.NativeBridge.postMessage() // 立即返回
   但Native处理可能很慢（数据库、网络等）

3. 所以需要回调机制来异步返回结果
```

**最佳实践**：
```dart
// ❌ 不好：阻塞等待
final result = await controller.runJavaScript('someLongTask()');

// ✅ 好：异步回调
controller.runJavaScript('someLongTask()'); // 不等待
// 通过JavaScriptChannel接收结果
```

## 5. 初始化时序

### 5.1 关键时间点

```
WebView生命周期：
┌──────────────────────────────────────┐
│ 1. WebView创建                       │
│    controller = WebViewController()  │
├──────────────────────────────────────┤
│ 2. 配置JavaScriptChannel             │
│    controller.addJavaScriptChannel() │
├──────────────────────────────────────┤
│ 3. 加载URL/HTML                      │
│    controller.loadRequest()          │
├──────────────────────────────────────┤
│ 4. 页面开始加载                       │
│    onPageStarted()                   │
├──────────────────────────────────────┤
│ 5. DOM解析完成                        │
│    DOMContentLoaded事件              │
├──────────────────────────────────────┤
│ 6. 页面加载完成                       │
│    onPageFinished()                  │
├──────────────────────────────────────┤
│ 7. 注入额外JS（可选）                 │
│    controller.runJavaScript()        │
└──────────────────────────────────────┘
```

### 5.2 Bridge就绪检测

**问题**：H5代码可能在Bridge注册前就执行

**解决方案**：
```javascript
// 方案1: 轮询检测
function waitForBridge(callback) {
  if (window.NativeBridge) {
    callback();
  } else {
    setTimeout(() => waitForBridge(callback), 50);
  }
}

// 方案2: 事件监听
window.addEventListener('nativeBridgeReady', () => {
  // Bridge已就绪
});

// 方案3: Promise包装
const bridgeReady = new Promise(resolve => {
  if (window.NativeBridge) {
    resolve();
  } else {
    window.addEventListener('nativeBridgeReady', resolve);
  }
});
```

## 6. 调试与监控

### 6.1 消息追踪

**建议实现消息日志**：
```
每条消息记录：
- 消息ID
- 方向（JS→Native 或 Native→JS）
- 方法名
- 参数（摘要）
- 发送时间
- 接收时间
- 处理时间
- 结果/错误
```

### 6.2 常见问题排查

| 问题 | 可能原因 | 排查方法 |
|------|---------|---------|
| JS调用无响应 | Bridge未注册 | 检查Channel名称是否一致 |
| Native调用JS失败 | 页面未加载完成 | 在onPageFinished后调用 |
| 数据传递异常 | JSON序列化失败 | 打印原始字符串检查 |
| 回调未执行 | ID不匹配 | 检查ID生成和传递 |
| 性能慢 | 频繁通信 | 批量处理，减少调用次数 |

### 6.3 性能监控

**关键指标**：
```
- 消息往返时间（RTT）: < 50ms
- 单次消息处理时间: < 10ms
- 消息队列长度: < 10
- 超时消息数: 0
- 失败率: < 0.1%
```

## 7. 安全性考虑

### 7.1 XSS防护

**风险**：恶意JS代码通过WebView执行

**防护措施**：
```
1. 只加载可信来源的HTML
2. 校验资源包完整性（SHA256）
3. 使用CSP（Content Security Policy）
4. 限制WebView可访问的API
```

### 7.2 方法白名单

```dart
final allowedMethods = {
  'getUserInfo', 'navigateTo', 'showToast', ...
};

onMessageReceived: (msg) {
  final data = json.decode(msg.message);
  if (!allowedMethods.contains(data['method'])) {
    // 拒绝执行
    return;
  }
  // 处理...
}
```

### 7.3 参数校验

```dart
// 校验必需参数
if (params['orderId'] == null || params['amount'] == null) {
  throw ArgumentError('Missing required params');
}

// 校验参数类型
if (params['amount'] is! num || params['amount'] <= 0) {
  throw ArgumentError('Invalid amount');
}
```

## 8. 最佳实践总结

### 8.1 设计原则

1. **消息要幂等**：同一消息多次执行结果一致
2. **超时要处理**：设置合理超时，避免无限等待
3. **错误要友好**：返回明确的错误码和消息
4. **日志要完整**：记录所有通信细节便于排查
5. **性能要优化**：批量处理，减少通信次数

### 8.2 常见陷阱

⚠️ **陷阱1**：在页面未加载完成就调用JS
```dart
// ❌ 错误
controller.loadRequest(url);
controller.runJavaScript('init()'); // 此时页面还没加载

// ✅ 正确
controller.loadRequest(url);
// 在onPageFinished中调用
```

⚠️ **陷阱2**：忘记JSON序列化
```javascript
// ❌ 错误
window.NativeBridge.postMessage({ key: 'value' }); // 传对象

// ✅ 正确
window.NativeBridge.postMessage(JSON.stringify({ key: 'value' }));
```

⚠️ **陷阱3**：同步等待异步结果
```javascript
// ❌ 错误
const result = window.NativeBridge.getUserInfo(); // 无法同步获取

// ✅ 正确
callNative('getUserInfo').then(result => {
  // 使用result
});
```

⚠️ **陷阱4**：回调函数未清理
```javascript
// 忘记清理会导致内存泄漏
callbacks[id] = callback;
// 执行后必须删除
delete callbacks[id];
```

## 9. 与其他方案对比

### 9.1 Flutter vs 原生iOS/Android

| 特性 | Flutter | 原生 |
|------|---------|------|
| 实现复杂度 | 简单（统一API） | 复杂（iOS/Android不同） |
| 性能 | 多一层转发 | 直接调用 |
| 调试 | 较难（跨3层） | 较易 |
| 维护成本 | 低（一套代码） | 高（两套代码） |

### 9.2 与其他通信方式对比

**URL Scheme**（已过时）：
```javascript
// 通过修改URL触发拦截
window.location = 'myapp://method?params=xxx';
```
- ❌ 有长度限制
- ❌ 无法传递复杂数据
- ❌ 已被JavaScriptChannel替代

**Prompt拦截**（已废弃）：
```javascript
prompt('native-call', JSON.stringify(data));
```
- ❌ 不推荐使用
- ❌ 体验差

**现代方案**：JavaScriptChannel（推荐）
- ✅ 官方支持
- ✅ 性能好
- ✅ 可靠性高

## 10. 总结

### 核心要点
1. Flutter与JS通信通过WebView桥接，不是直接通信
2. 两个方向：runJavaScript（Flutter→JS）和JavaScriptChannel（JS→Flutter）
3. 数据只能传String，需JSON序列化
4. 双向回调需自行设计消息协议
5. 注意初始化时序，确保Bridge就绪

### 关键流程
```
完整通信流程：
JS调用 → postMessage → Native拦截 → Platform Channel 
→ Flutter处理 → Platform Channel → Native → runJavaScript 
→ JS回调执行
```

这就是Flutter与JS交互的完整原理！


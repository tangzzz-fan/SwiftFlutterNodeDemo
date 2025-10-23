# Flutter与JS交互 - 重点难点与最佳实践

## 1. 核心挑战概览

### 1.1 三大核心问题

```
┌─────────────────────────────────────────┐
│  1. 线程安全问题                         │
│     - JS单线程 vs Flutter多线程          │
│     - UI线程阻塞风险                     │
│     - 并发调用的竞态条件                  │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│  2. 性能瓶颈                             │
│     - 数据序列化/反序列化开销             │
│     - 跨线程消息传递延迟                  │
│     - 频繁通信导致的帧率下降              │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│  3. 可靠性问题                           │
│     - 消息丢失                           │
│     - 回调泄漏                           │
│     - 异步时序混乱                       │
└─────────────────────────────────────────┘
```

## 2. 线程模型深度解析

### 2.1 线程分布

**Flutter端（Dart）**：
```
UI线程（Platform Thread）
  - 渲染界面
  - 处理用户交互
  - ⚠️ 不能阻塞（>16ms会掉帧）

Platform Channel线程
  - 处理与Native通信
  - 执行Platform代码

Isolate线程（可选）
  - 处理CPU密集任务
  - 完全隔离，通过消息通信
```

**WebView端（JavaScript）**：
```
JS主线程（唯一线程）
  - 执行所有JS代码
  - 渲染H5页面
  - ⚠️ 任何耗时操作都会卡顿

Web Worker线程（可选）
  - 后台计算
  - 不能操作DOM
```

### 2.2 通信的线程跳转

**完整调用链**：
```
JS调用Flutter:
┌──────────────┐
│ JS主线程      │ window.NativeBridge.postMessage()
└──────┬───────┘
       │ (WebView内部调度)
       ▼
┌──────────────┐
│ Native主线程  │ WKScriptMessageHandler
└──────┬───────┘
       │ (Platform Channel)
       ▼
┌──────────────┐
│ Dart UI线程   │ onMessageReceived回调
└──────────────┘

耗时：2-5ms（跨3次线程切换）
```

**关键问题**：
- 每次线程切换都有开销（上下文切换）
- 如果Dart回调耗时，会阻塞UI线程
- 高频调用会导致线程切换过多

### 2.3 线程安全陷阱

**❌ 陷阱1：在UI线程执行耗时操作**

```dart
// 错误示例
controller.addJavaScriptChannel('Bridge',
  onMessageReceived: (msg) {
    // ❌ 这在UI线程执行，会阻塞渲染
    final data = json.decode(msg.message); // 如果数据大，会卡顿
    
    // ❌ 同步的数据库操作
    final result = database.query('SELECT * FROM products'); // 阻塞
    
    // ❌ 网络请求
    final response = http.get(url); // 阻塞
  }
);
```

**✅ 正确做法：异步处理**

```dart
controller.addJavaScriptChannel('Bridge',
  onMessageReceived: (msg) async {
    // ✅ 快速返回，异步处理
    _handleMessageAsync(msg);
  }
);

Future<void> _handleMessageAsync(JavaScriptMessage msg) async {
  // 在后台处理
  final data = await compute(_parseMessage, msg.message);
  
  // 数据库操作
  final result = await database.query(...);
  
  // 网络请求
  final response = await http.get(url);
  
  // 处理完成后回调JS
  await _sendResponseToJS(data);
}

// compute会在独立Isolate中执行
Map<String, dynamic> _parseMessage(String message) {
  return json.decode(message);
}
```

**❌ 陷阱2：JS端同步等待**

```javascript
// ❌ 错误：JS尝试同步等待Native结果
function getUserInfo() {
  let result = null;
  window.NativeBridge.postMessage(
    JSON.stringify({ method: 'getUserInfo' })
  );
  // ❌ result永远是null，因为Native是异步返回的
  return result;
}
```

**✅ 正确：使用Promise封装**

```javascript
function getUserInfo() {
  return new Promise((resolve, reject) => {
    const id = generateId();
    
    // 注册回调
    callbacks[id] = (result) => {
      resolve(result);
      delete callbacks[id];
    };
    
    // 发送消息
    window.NativeBridge.postMessage(
      JSON.stringify({ method: 'getUserInfo', id })
    );
    
    // 超时处理
    setTimeout(() => {
      if (callbacks[id]) {
        delete callbacks[id];
        reject(new Error('Timeout'));
      }
    }, 30000);
  });
}
```

## 3. 性能优化深度实践

### 3.1 数据序列化开销

**问题分析**：
```
1000条数据的传输：
┌────────────────────────────────────┐
│ JS端序列化：JSON.stringify()       │
│ 耗时：~10ms                        │
├────────────────────────────────────┤
│ Native端反序列化：json.decode()     │
│ 耗时：~15ms                        │
├────────────────────────────────────┤
│ 总开销：25ms                        │
│ 占用一帧的时间：25/16.67 = 1.5帧   │
└────────────────────────────────────┘
```

**优化策略**：

#### 策略1：批量处理
```javascript
// ❌ 逐条发送
events.forEach(event => {
  sendToNative(event); // 1000次序列化
});

// ✅ 批量发送
sendToNative({ events: events }); // 1次序列化
```

#### 策略2：数据压缩
```javascript
// 大数据压缩传输
import pako from 'pako';

function sendLargeData(data) {
  const json = JSON.stringify(data);
  
  if (json.length > 10000) { // 大于10KB
    // 压缩
    const compressed = pako.deflate(json);
    const base64 = btoa(String.fromCharCode(...compressed));
    
    window.NativeBridge.postMessage(JSON.stringify({
      method: 'receiveLargeData',
      compressed: true,
      data: base64
    }));
  } else {
    // 直接发送
    window.NativeBridge.postMessage(json);
  }
}
```

```dart
// Flutter端解压
Future<void> _handleLargeData(Map<String, dynamic> message) async {
  if (message['compressed'] == true) {
    // 在后台线程解压
    final decompressed = await compute(_decompress, message['data']);
    // 处理数据
  }
}
```

#### 策略3：增量传输
```javascript
// 只传差异数据
const previousState = { ... };
const currentState = { ... };

// 计算diff
const changes = computeDiff(previousState, currentState);

// 只发送变化的部分
sendToNative({ type: 'update', changes });
```

### 3.2 频繁通信的帧率影响

**问题场景**：滚动监听
```javascript
// ❌ 每次滚动都通知Native
window.addEventListener('scroll', () => {
  const scrollTop = window.scrollY;
  window.NativeBridge.postMessage(
    JSON.stringify({ event: 'scroll', y: scrollTop })
  );
  // 滚动时每帧触发60次，导致严重卡顿
});
```

**优化：节流（Throttle）**
```javascript
// ✅ 限制调用频率
let lastCall = 0;
const throttleInterval = 100; // 100ms最多调用一次

window.addEventListener('scroll', () => {
  const now = Date.now();
  if (now - lastCall < throttleInterval) {
    return; // 跳过
  }
  
  lastCall = now;
  const scrollTop = window.scrollY;
  window.NativeBridge.postMessage(
    JSON.stringify({ event: 'scroll', y: scrollTop })
  );
});
```

**优化：防抖（Debounce）**
```javascript
// ✅ 只在停止后触发
let timer = null;

window.addEventListener('scroll', () => {
  clearTimeout(timer);
  
  timer = setTimeout(() => {
    const scrollTop = window.scrollY;
    window.NativeBridge.postMessage(
      JSON.stringify({ event: 'scrollEnd', y: scrollTop })
    );
  }, 150); // 停止150ms后触发
});
```

**优化：requestAnimationFrame**
```javascript
// ✅ 与浏览器渲染同步
let ticking = false;

window.addEventListener('scroll', () => {
  if (!ticking) {
    requestAnimationFrame(() => {
      const scrollTop = window.scrollY;
      window.NativeBridge.postMessage(
        JSON.stringify({ event: 'scroll', y: scrollTop })
      );
      ticking = false;
    });
    ticking = true;
  }
});
```

### 3.3 内存优化

**❌ 问题：回调泄漏**
```javascript
// 回调没有清理，导致内存泄漏
const callbacks = {};

function callNative(method) {
  const id = generateId();
  callbacks[id] = (result) => {
    console.log(result);
    // ❌ 忘记删除，callbacks越来越大
  };
}
```

**✅ 自动清理机制**
```javascript
class CallbackManager {
  constructor() {
    this.callbacks = new Map();
    this.timeouts = new Map();
  }
  
  register(id, callback, timeout = 30000) {
    this.callbacks.set(id, callback);
    
    // 自动清理
    const timer = setTimeout(() => {
      this.cleanup(id);
    }, timeout);
    
    this.timeouts.set(id, timer);
  }
  
  execute(id, result) {
    const callback = this.callbacks.get(id);
    if (callback) {
      callback(result);
      this.cleanup(id);
    }
  }
  
  cleanup(id) {
    this.callbacks.delete(id);
    
    const timer = this.timeouts.get(id);
    if (timer) {
      clearTimeout(timer);
      this.timeouts.delete(id);
    }
  }
  
  // 定期清理过期回调
  startAutoCleanup() {
    setInterval(() => {
      const now = Date.now();
      for (const [id, timer] of this.timeouts) {
        // 清理超时的回调
      }
    }, 60000); // 每分钟清理一次
  }
}
```

## 4. 并发与竞态条件

### 4.1 并发调用问题

**场景**：用户快速点击
```javascript
// 用户连续点击按钮
button.onclick = async () => {
  // 第1次点击：id=1
  // 第2次点击：id=2（第1次还没返回）
  // 第3次点击：id=3
  
  const result = await callNative('purchase');
  // 问题：三次调用的返回顺序可能是 2, 1, 3
  // 导致UI状态混乱
};
```

**✅ 解决：去重与锁**
```javascript
class RequestQueue {
  constructor() {
    this.pending = new Set();
  }
  
  async call(method, params) {
    const key = `${method}:${JSON.stringify(params)}`;
    
    // 如果相同请求正在进行，直接返回
    if (this.pending.has(key)) {
      throw new Error('Duplicate request');
    }
    
    this.pending.add(key);
    
    try {
      const result = await callNative(method, params);
      return result;
    } finally {
      this.pending.delete(key);
    }
  }
}

// 使用
const queue = new RequestQueue();
button.onclick = async () => {
  try {
    const result = await queue.call('purchase', { productId: 123 });
  } catch (e) {
    // 重复请求被阻止
  }
};
```

### 4.2 消息顺序问题

**问题**：消息乱序
```
发送顺序：A → B → C
接收顺序：B → A → C（网络延迟、线程调度）
```

**✅ 解决：序列号机制**
```dart
class OrderedMessageHandler {
  int _expectedSeq = 0;
  final Map<int, dynamic> _buffer = {};
  
  void handleMessage(int seq, dynamic data) {
    if (seq == _expectedSeq) {
      // 期望的消息，立即处理
      _processMessage(data);
      _expectedSeq++;
      
      // 检查缓冲区是否有后续消息
      _processPendingMessages();
    } else if (seq > _expectedSeq) {
      // 乱序消息，缓存起来
      _buffer[seq] = data;
    } else {
      // 过期消息，丢弃
      print('Duplicate message: $seq');
    }
  }
  
  void _processPendingMessages() {
    while (_buffer.containsKey(_expectedSeq)) {
      _processMessage(_buffer[_expectedSeq]!);
      _buffer.remove(_expectedSeq);
      _expectedSeq++;
    }
  }
}
```

## 5. WebView生命周期陷阱

### 5.1 初始化时序问题

**❌ 常见错误**：
```dart
// 创建WebView
final controller = WebViewController();

// ❌ 立即调用JS（此时WebView还没加载完成）
await controller.runJavaScript('init()'); // 失败

// 加载页面
await controller.loadRequest(Uri.parse(url));
```

**✅ 正确顺序**：
```dart
final controller = WebViewController()
  ..setNavigationDelegate(
    NavigationDelegate(
      onPageFinished: (url) {
        // ✅ 页面加载完成后才调用
        controller.runJavaScript('init()');
      },
    ),
  )
  ..loadRequest(Uri.parse(url));
```

### 5.2 页面销毁时的清理

**❌ 忘记清理**：
```dart
@override
void dispose() {
  // ❌ 忘记清理，导致内存泄漏
  super.dispose();
}
```

**✅ 完整清理**：
```dart
@override
void dispose() {
  // 取消定时器
  _heartbeatTimer?.cancel();
  
  // 清理回调
  _callbacks.clear();
  
  // 通知JS页面即将销毁
  controller.runJavaScript('window.onNativeDestroy?.()');
  
  // 释放WebView
  // controller.dispose(); // 根据实际API
  
  super.dispose();
}
```

**JS端清理**：
```javascript
window.onNativeDestroy = () => {
  // 清理定时器
  clearInterval(heartbeatTimer);
  
  // 清理事件监听
  window.removeEventListener('scroll', scrollHandler);
  
  // 清理回调
  Object.keys(callbacks).forEach(id => {
    delete callbacks[id];
  });
};
```

## 6. 调试与监控

### 6.1 通信日志

**实现完整的消息追踪**：
```dart
class MessageLogger {
  final List<MessageLog> _logs = [];
  
  void logSend(String method, dynamic params) {
    _logs.add(MessageLog(
      direction: 'Flutter→JS',
      method: method,
      params: params,
      timestamp: DateTime.now(),
    ));
  }
  
  void logReceive(String method, dynamic params) {
    _logs.add(MessageLog(
      direction: 'JS→Flutter',
      method: method,
      params: params,
      timestamp: DateTime.now(),
    ));
  }
  
  void printStats() {
    // 统计分析
    final methodCounts = <String, int>{};
    int totalTime = 0;
    
    for (var log in _logs) {
      methodCounts[log.method] = (methodCounts[log.method] ?? 0) + 1;
    }
    
    print('=== Bridge Stats ===');
    print('Total messages: ${_logs.length}');
    print('Methods: $methodCounts');
    print('Average time: ${totalTime / _logs.length}ms');
  }
}
```

### 6.2 性能监控

**检测耗时操作**：
```dart
Future<T> _measurePerformance<T>(
  String operation,
  Future<T> Function() fn,
) async {
  final stopwatch = Stopwatch()..start();
  
  try {
    final result = await fn();
    stopwatch.stop();
    
    final elapsed = stopwatch.elapsedMilliseconds;
    
    if (elapsed > 16) { // 超过一帧
      print('⚠️ Slow operation: $operation took ${elapsed}ms');
    }
    
    return result;
  } catch (e) {
    stopwatch.stop();
    print('❌ Failed: $operation (${stopwatch.elapsedMilliseconds}ms)');
    rethrow;
  }
}

// 使用
await _measurePerformance('getUserInfo', () async {
  return await callNative('getUserInfo');
});
```

## 7. 最佳实践总结

### 7.1 性能优化清单

```
✅ 数据传输优化
  - 批量处理消息
  - 压缩大数据
  - 只传必要字段
  - 使用增量更新

✅ 频率控制
  - 高频事件使用throttle/debounce
  - 使用requestAnimationFrame
  - 限制并发请求数

✅ 内存管理
  - 及时清理回调
  - 设置超时机制
  - 监控内存使用

✅ 线程优化
  - 耗时操作使用compute
  - 避免阻塞UI线程
  - 合理使用异步
```

### 7.2 可靠性保障清单

```
✅ 错误处理
  - 所有调用都有try-catch
  - 设置合理的超时时间
  - 失败重试机制

✅ 状态管理
  - 消息序列号
  - 去重机制
  - 顺序保证

✅ 生命周期
  - 正确的初始化顺序
  - 完整的清理逻辑
  - 页面销毁通知
```

### 7.3 开发建议

```
✅ 调试工具
  - 完整的日志记录
  - 性能监控
  - 统计分析

✅ 测试覆盖
  - 单元测试
  - 压力测试
  - 边界条件测试
  - 弱网测试

✅ 文档规范
  - 明确的API文档
  - 数据格式约定
  - 错误码定义
```

## 8. 典型问题解决方案

### 8.1 问题：页面白屏

**原因分析**：
```
1. JS执行错误导致页面崩溃
2. 资源加载失败
3. WebView初始化失败
4. 内存不足
```

**解决方案**：
```dart
// 1. 捕获JS错误
controller.setNavigationDelegate(
  NavigationDelegate(
    onWebResourceError: (error) {
      print('Resource error: ${error.description}');
      // 降级处理
    },
  ),
);

// 2. 超时检测
final pageLoadTimeout = Timer(Duration(seconds: 10), () {
  if (!_pageLoaded) {
    // 页面加载超时，显示错误页
    _showErrorPage();
  }
});

// 3. 内存监控
DeviceInfoPlugin().iosInfo.then((info) {
  // 检查可用内存
});
```

### 8.2 问题：消息丢失

**原因**：
```
1. WebView还未就绪
2. 页面正在刷新
3. 网络中断
4. 缓冲区溢出
```

**解决**：
```dart
class ReliableMessageSender {
  final List<Message> _queue = [];
  bool _webViewReady = false;
  
  void markReady() {
    _webViewReady = true;
    _flushQueue();
  }
  
  Future<void> send(Message msg) async {
    if (_webViewReady) {
      await _doSend(msg);
    } else {
      _queue.add(msg);
    }
  }
  
  void _flushQueue() {
    for (var msg in _queue) {
      _doSend(msg);
    }
    _queue.clear();
  }
}
```

### 8.3 问题：帧率下降

**原因**：
```
1. 频繁的跨线程通信
2. 大数据序列化
3. 同步操作阻塞UI
```

**优化**：
```dart
// 使用FPS监控
class FPSMonitor {
  int _frameCount = 0;
  DateTime _lastCheck = DateTime.now();
  
  void onFrame() {
    _frameCount++;
    
    final now = DateTime.now();
    if (now.difference(_lastCheck).inSeconds >= 1) {
      print('FPS: $_frameCount');
      
      if (_frameCount < 50) {
        print('⚠️ Low FPS detected!');
        // 触发优化措施
      }
      
      _frameCount = 0;
      _lastCheck = now;
    }
  }
}

// 在每帧调用
SchedulerBinding.instance.addPostFrameCallback((_) {
  fpsMonitor.onFrame();
});
```

## 9. 核心要点总结

### 关键原则

1. **异步优先**：所有通信都应该是异步的
2. **批量处理**：减少通信次数
3. **及时清理**：防止内存泄漏
4. **错误容忍**：任何调用都可能失败
5. **性能监控**：持续监控性能指标

### 记住这些数字

```
16ms  - 一帧的时间，超过会掉帧
50ms  - 用户可感知的延迟
100ms - 需要显示加载提示
1s    - 用户焦虑的阈值
10KB  - 考虑压缩的数据阈值
30s   - 合理的超时时间
```

这些都是实战中总结出的经验，遵循这些最佳实践可以避免90%的坑！


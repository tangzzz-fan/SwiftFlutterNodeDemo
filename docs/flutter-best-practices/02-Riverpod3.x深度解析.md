# Riverpod 3.x 深度解析

## 1. 核心原理

### 1.1 Riverpod是什么？

**本质**：一个**依赖注入 + 状态管理**的框架

```
传统方式：
Widget → InheritedWidget → 子Widget
问题：编译时不安全，运行时才知道错误

Riverpod方式：
Widget → Provider → Ref → 状态
优势：编译时安全，依赖明确
```

### 1.2 与Provider的区别

| 特性 | Provider | Riverpod |
|------|----------|----------|
| 依赖BuildContext | ✅ 是 | ❌ 否 |
| 编译时安全 | ❌ 否 | ✅ 是 |
| 可测试性 | 一般 | 优秀 |
| 性能 | 好 | 更好 |
| 学习曲线 | 平缓 | 略陡 |

**Riverpod的核心优势**：
```
1. 无需context
   - 可以在任何地方访问Provider
   - 不依赖Widget树

2. 编译时安全
   - 类型检查
   - 依赖关系明确

3. 更好的性能
   - 精确的rebuild控制
   - 自动dispose

4. 易于测试
   - 可以轻松override Provider
   - 不需要mock BuildContext
```

### 1.3 核心概念

**三个核心角色**：

```dart
// 1. Provider - 定义状态
final counterProvider = StateProvider<int>((ref) => 0);

// 2. Ref - 访问其他Provider
final doubleProvider = Provider<int>((ref) {
  final count = ref.watch(counterProvider); // 读取并监听
  return count * 2;
});

// 3. Consumer - 在Widget中使用
class MyWidget extends ConsumerWidget {
  Widget build(BuildContext context, WidgetRef ref) {
    final count = ref.watch(counterProvider);
    return Text('$count');
  }
}
```

## 2. Provider类型详解

### 2.1 Provider - 只读，不可变

**用途**：提供不变的值或计算结果

```dart
// 简单值
final apiUrlProvider = Provider<String>((ref) => 'https://api.example.com');

// 依赖注入服务
final apiServiceProvider = Provider<ApiService>((ref) {
  final url = ref.watch(apiUrlProvider);
  return ApiService(baseUrl: url);
});

// 计算值
final totalPriceProvider = Provider<double>((ref) {
  final cart = ref.watch(cartProvider);
  return cart.items.fold(0, (sum, item) => sum + item.price);
});
```

**特点**：
```
✅ 创建后不可修改
✅ 缓存结果
✅ 依赖变化时自动重新计算
```

### 2.2 StateProvider - 简单状态

**用途**：管理简单的可变状态（类似useState）

```dart
final counterProvider = StateProvider<int>((ref) => 0);

// 读取
final count = ref.watch(counterProvider);

// 修改
ref.read(counterProvider.notifier).state = 10;
ref.read(counterProvider.notifier).update((state) => state + 1);
```

**适用场景**：
```
✅ 简单的计数器
✅ 开关状态
✅ 表单输入
✅ UI临时状态
```

**限制**：
```
❌ 不适合复杂逻辑
❌ 不适合需要验证的状态
❌ 状态修改逻辑分散
```

### 2.3 NotifierProvider - 复杂状态（推荐）

**核心**：将状态和逻辑封装在一起

```dart
// 1. 定义Notifier类
class CounterNotifier extends Notifier<int> {
  @override
  int build() => 0; // 初始状态
  
  void increment() {
    state = state + 1;
  }
  
  void decrement() {
    state = state - 1;
  }
  
  void reset() {
    state = 0;
  }
}

// 2. 创建Provider
final counterProvider = NotifierProvider<CounterNotifier, int>(
  () => CounterNotifier(),
);

// 3. 使用
final count = ref.watch(counterProvider);
ref.read(counterProvider.notifier).increment();
```

**优势**：
```
✅ 逻辑集中
✅ 方法明确
✅ 易于测试
✅ 类型安全
```

### 2.4 AsyncNotifierProvider - 异步状态

**用途**：管理异步数据（API请求、数据库等）

```dart
class UserNotifier extends AsyncNotifier<User> {
  @override
  Future<User> build() async {
    // 初始化：加载用户数据
    return await _fetchUser();
  }
  
  Future<void> refresh() async {
    // 设置loading状态
    state = const AsyncLoading();
    
    // 加载数据
    state = await AsyncValue.guard(() async {
      return await _fetchUser();
    });
  }
  
  Future<User> _fetchUser() async {
    final api = ref.read(apiServiceProvider);
    return await api.getUser();
  }
}

final userProvider = AsyncNotifierProvider<UserNotifier, User>(
  () => UserNotifier(),
);
```

**AsyncValue的三种状态**：
```dart
final user = ref.watch(userProvider);

user.when(
  data: (user) => Text('Hello ${user.name}'),
  loading: () => CircularProgressIndicator(),
  error: (err, stack) => Text('Error: $err'),
);
```

### 2.5 StreamProvider - 流式数据

**用途**：处理Stream数据（WebSocket、实时更新）

```dart
final messagesProvider = StreamProvider<List<Message>>((ref) {
  final api = ref.watch(apiServiceProvider);
  return api.messageStream(); // 返回Stream
});

// 使用
final messages = ref.watch(messagesProvider);
messages.when(
  data: (msgs) => ListView(children: msgs.map(...)),
  loading: () => CircularProgressIndicator(),
  error: (err, _) => Text('Error'),
);
```

### 2.6 FutureProvider - 一次性异步

**用途**：一次性的异步操作

```dart
final configProvider = FutureProvider<Config>((ref) async {
  // 只执行一次
  return await loadConfig();
});
```

**与AsyncNotifierProvider的区别**：
```
FutureProvider：
- 一次性加载
- 不能主动刷新
- 简单场景

AsyncNotifierProvider：
- 可以多次刷新
- 有方法控制
- 复杂场景
```

## 3. Ref详解

### 3.1 watch vs read vs listen

```dart
class MyNotifier extends Notifier<int> {
  int build() {
    // ✅ watch - 监听变化，自动重建
    final config = ref.watch(configProvider);
    
    // ✅ read - 只读取一次，不监听
    final api = ref.read(apiServiceProvider);
    
    // ✅ listen - 监听变化，执行副作用
    ref.listen(authProvider, (prev, next) {
      if (next == null) {
        // 用户登出，清理数据
        state = 0;
      }
    });
    
    return 0;
  }
}
```

**何时使用**：
```
watch:
- 依赖其他Provider的值
- 值变化时需要重建
- 用在build方法中

read:
- 只需要读取一次
- 不关心变化
- 调用方法时使用

listen:
- 需要执行副作用
- 导航、显示Toast等
- 不需要rebuild
```

### 3.2 依赖管理

**自动依赖追踪**：
```dart
final userIdProvider = StateProvider<String>((ref) => 'user1');

final userProvider = Provider<User>((ref) {
  final userId = ref.watch(userIdProvider); // 注册依赖
  return fetchUser(userId);
  // 当userId变化，自动重新执行
});
```

**依赖图**：
```
userIdProvider
    ↓ (watch)
userProvider
    ↓ (watch)
profileWidget (rebuild)
```

### 3.3 生命周期钩子

```dart
final myProvider = Provider<MyService>((ref) {
  // 创建
  final service = MyService();
  
  // 监听其他Provider
  ref.listen(configProvider, (prev, next) {
    service.updateConfig(next);
  });
  
  // 清理
  ref.onDispose(() {
    service.dispose();
  });
  
  return service;
});
```

## 4. 与外界交互

### 4.1 从外部读取Provider

**场景**：在非Widget代码中访问Provider

```dart
// 1. 创建ProviderContainer
final container = ProviderContainer();

// 2. 读取Provider
final count = container.read(counterProvider);

// 3. 监听变化
container.listen(counterProvider, (prev, next) {
  print('Count changed: $next');
});

// 4. 清理
container.dispose();
```

**实际应用**：
```dart
// 在main函数中初始化
void main() {
  final container = ProviderContainer();
  
  // 预加载配置
  container.read(configProvider);
  
  runApp(
    UncontrolledProviderScope(
      container: container,
      child: MyApp(),
    ),
  );
}
```

### 4.2 ProviderScope与依赖覆盖

**覆盖Provider（测试/多环境）**：

```dart
// 生产环境
final apiProvider = Provider<ApiService>((ref) {
  return ApiService(baseUrl: 'https://prod.api.com');
});

// 测试时覆盖
testWidgets('Test', (tester) async {
  await tester.pumpWidget(
    ProviderScope(
      overrides: [
        apiProvider.overrideWithValue(MockApiService()),
      ],
      child: MyApp(),
    ),
  );
});
```

**多环境配置**：
```dart
void main() {
  final env = const String.fromEnvironment('ENV', defaultValue: 'dev');
  
  runApp(
    ProviderScope(
      overrides: [
        if (env == 'dev')
          apiUrlProvider.overrideWithValue('https://dev.api.com'),
        if (env == 'prod')
          apiUrlProvider.overrideWithValue('https://prod.api.com'),
      ],
      child: MyApp(),
    ),
  );
}
```

### 4.3 Platform Channel集成

**与Native交互**：

```dart
class NativeServiceNotifier extends Notifier<String> {
  static const platform = MethodChannel('com.example.app/native');
  
  @override
  String build() {
    _listenToNative();
    return 'initial';
  }
  
  void _listenToNative() {
    // 监听Native事件
    platform.setMethodCallHandler((call) async {
      if (call.method == 'dataUpdate') {
        state = call.arguments as String;
      }
    });
  }
  
  Future<void> callNative() async {
    try {
      final result = await platform.invokeMethod('getData');
      state = result;
    } catch (e) {
      // 错误处理
    }
  }
}

final nativeServiceProvider = NotifierProvider<NativeServiceNotifier, String>(
  () => NativeServiceNotifier(),
);
```

### 4.4 与JSBridge集成

**WebView通信**：

```dart
class JSBridgeNotifier extends Notifier<Map<String, dynamic>> {
  late WebViewController _controller;
  
  @override
  Map<String, dynamic> build() {
    return {};
  }
  
  void setController(WebViewController controller) {
    _controller = controller;
    _setupBridge();
  }
  
  void _setupBridge() {
    _controller.addJavaScriptChannel(
      'FlutterBridge',
      onMessageReceived: (msg) {
        final data = json.decode(msg.message);
        state = {...state, ...data};
      },
    );
  }
  
  Future<void> callJS(String method, dynamic params) async {
    await _controller.runJavaScript(
      'window.$method(${json.encode(params)})'
    );
  }
}
```

## 5. 性能优化

### 5.1 精确控制rebuild

**使用select**：
```dart
// ❌ 整个对象变化都会rebuild
final user = ref.watch(userProvider);

// ✅ 只监听name字段
final userName = ref.watch(
  userProvider.select((user) => user.name),
);
```

### 5.2 Family修饰符

**参数化Provider**：
```dart
// 根据ID获取不同的用户
final userProvider = Provider.family<User, String>((ref, userId) {
  return fetchUser(userId);
});

// 使用
final user1 = ref.watch(userProvider('user1'));
final user2 = ref.watch(userProvider('user2'));
```

**缓存机制**：
```
userProvider('user1') - 缓存实例1
userProvider('user2') - 缓存实例2
相同参数返回相同实例
```

### 5.3 AutoDispose

**自动释放不用的Provider**：
```dart
// 自动dispose
final tempProvider = Provider.autoDispose<String>((ref) {
  // 当没有监听者时自动dispose
  return 'temp';
});

// 保持存活
final keepAliveProvider = Provider.autoDispose<String>((ref) {
  final link = ref.keepAlive();
  
  // 某些条件下释放
  Future.delayed(Duration(minutes: 5), () {
    link.close();
  });
  
  return 'data';
});
```

## 6. 最佳实践

### 6.1 Provider组织

```
lib/
├── providers/
│   ├── auth/
│   │   ├── auth_provider.dart
│   │   └── auth_state.dart
│   ├── user/
│   │   ├── user_provider.dart
│   │   └── user_notifier.dart
│   └── providers.dart (导出所有)
```

### 6.2 状态设计

```
✅ 单一数据源
✅ 不可变状态对象
✅ 使用copyWith更新
✅ 分层Provider（基础→派生）
```

### 6.3 错误处理

```dart
class DataNotifier extends AsyncNotifier<Data> {
  @override
  Future<Data> build() async {
    try {
      return await fetchData();
    } catch (e, stack) {
      // 记录错误
      ref.read(loggerProvider).error(e, stack);
      
      // 返回默认值或重新抛出
      rethrow;
    }
  }
}
```

### 6.4 测试策略

```dart
test('Counter increments', () {
  final container = ProviderContainer();
  
  expect(container.read(counterProvider), 0);
  
  container.read(counterProvider.notifier).increment();
  
  expect(container.read(counterProvider), 1);
  
  container.dispose();
});
```

## 7. 核心要点

记住这些关键概念：

1. **Provider类型**：选择合适的Provider类型
   - 简单状态 → StateProvider
   - 复杂逻辑 → NotifierProvider  
   - 异步数据 → AsyncNotifierProvider

2. **Ref使用**：
   - watch - 监听并rebuild
   - read - 只读取
   - listen - 副作用

3. **性能优化**：
   - 使用select精确监听
   - autoDispose自动清理
   - family参数化

4. **外界交互**：
   - ProviderContainer独立使用
   - ProviderScope覆盖配置
   - 与Platform Channel集成

Riverpod是Flutter状态管理的未来，理解其原理能写出更优雅的代码！


# Riverpod 3.x 实战：购物车组件开发

## 1. 需求分析

### 1.1 功能需求

我们要开发一个**购物车组件**，具备以下功能：

```
核心功能：
✅ 展示购物车商品列表
✅ 添加商品到购物车
✅ 调整商品数量
✅ 移除商品
✅ 计算总价
✅ 本地持久化
✅ 加载状态管理
✅ 错误处理
```

### 1.2 复杂度分析

**中等复杂度体现在**：
- 状态组合（多个相关状态）
- 异步操作（持久化、API调用）
- 派生状态（总价计算）
- 副作用（本地存储）
- UI交互（乐观更新）

## 2. 架构设计

### 2.1 状态分层

```
┌────────────────────────────────────────┐
│           UI Layer (Widget)            │
│  - ShoppingCartScreen                  │
│  - CartItemWidget                      │
│  - CartSummary                         │
└────────────────────────────────────────┘
              ↓ ↑ (watch/read)
┌────────────────────────────────────────┐
│        State Layer (Provider)          │
│  - cartProvider (核心状态)              │
│  - cartTotalProvider (派生状态)         │
│  - cartItemCountProvider (派生状态)     │
└────────────────────────────────────────┘
              ↓ ↑
┌────────────────────────────────────────┐
│      Service Layer (Business)          │
│  - CartNotifier (业务逻辑)             │
│  - CartRepository (数据持久化)          │
└────────────────────────────────────────┘
              ↓ ↑
┌────────────────────────────────────────┐
│         Data Layer (Storage)           │
│  - SharedPreferences                   │
│  - API Service                         │
└────────────────────────────────────────┘
```

### 2.2 Provider设计

```dart
// 1. 核心状态 - 购物车商品列表
final cartProvider = AsyncNotifierProvider<CartNotifier, List<CartItem>>(
  () => CartNotifier(),
);

// 2. 派生状态 - 总价
final cartTotalProvider = Provider<double>((ref) {
  final cart = ref.watch(cartProvider);
  return cart.when(
    data: (items) => items.fold(
      0.0,
      (sum, item) => sum + item.product.price * item.quantity,
    ),
    loading: () => 0.0,
    error: (_, __) => 0.0,
  );
});

// 3. 派生状态 - 商品数量
final cartItemCountProvider = Provider<int>((ref) {
  final cart = ref.watch(cartProvider);
  return cart.when(
    data: (items) => items.fold(0, (sum, item) => sum + item.quantity),
    loading: () => 0,
    error: (_, __) => 0,
  );
});

// 4. 数据持久化服务
final cartRepositoryProvider = Provider<CartRepository>((ref) {
  return CartRepository();
});
```

## 3. 数据模型

### 3.1 领域模型

```dart
// 商品模型
class Product {
  final String id;
  final String name;
  final double price;
  final String imageUrl;
  final String? description;

  const Product({
    required this.id,
    required this.name,
    required this.price,
    required this.imageUrl,
    this.description,
  });

  factory Product.fromJson(Map<String, dynamic> json) => Product(
    id: json['id'] as String,
    name: json['name'] as String,
    price: (json['price'] as num).toDouble(),
    imageUrl: json['imageUrl'] as String,
    description: json['description'] as String?,
  );

  Map<String, dynamic> toJson() => {
    'id': id,
    'name': name,
    'price': price,
    'imageUrl': imageUrl,
    'description': description,
  };
}

// 购物车项目模型
class CartItem {
  final Product product;
  final int quantity;
  final DateTime addedAt;

  const CartItem({
    required this.product,
    required this.quantity,
    required this.addedAt,
  });

  CartItem copyWith({
    Product? product,
    int? quantity,
    DateTime? addedAt,
  }) {
    return CartItem(
      product: product ?? this.product,
      quantity: quantity ?? this.quantity,
      addedAt: addedAt ?? this.addedAt,
    );
  }

  factory CartItem.fromJson(Map<String, dynamic> json) => CartItem(
    product: Product.fromJson(json['product'] as Map<String, dynamic>),
    quantity: json['quantity'] as int,
    addedAt: DateTime.parse(json['addedAt'] as String),
  );

  Map<String, dynamic> toJson() => {
    'product': product.toJson(),
    'quantity': quantity,
    'addedAt': addedAt.toIso8601String(),
  };

  double get totalPrice => product.price * quantity;
}
```

## 4. 业务逻辑层

### 4.1 CartNotifier实现

```dart
class CartNotifier extends AsyncNotifier<List<CartItem>> {
  // 依赖注入
  late final CartRepository _repository;

  @override
  Future<List<CartItem>> build() async {
    // 注入依赖
    _repository = ref.read(cartRepositoryProvider);
    
    // 初始化：从本地存储加载
    try {
      final items = await _repository.loadCart();
      return items;
    } catch (e) {
      // 加载失败，返回空购物车
      return [];
    }
  }

  /// 添加商品到购物车
  Future<void> addProduct(Product product, {int quantity = 1}) async {
    // 1. 乐观更新：立即更新UI
    final currentItems = state.valueOrNull ?? [];
    
    // 检查商品是否已存在
    final existingIndex = currentItems.indexWhere(
      (item) => item.product.id == product.id,
    );

    List<CartItem> updatedItems;
    if (existingIndex >= 0) {
      // 商品已存在，增加数量
      updatedItems = [
        ...currentItems.sublist(0, existingIndex),
        currentItems[existingIndex].copyWith(
          quantity: currentItems[existingIndex].quantity + quantity,
        ),
        ...currentItems.sublist(existingIndex + 1),
      ];
    } else {
      // 新商品，添加到列表
      updatedItems = [
        ...currentItems,
        CartItem(
          product: product,
          quantity: quantity,
          addedAt: DateTime.now(),
        ),
      ];
    }

    // 2. 更新状态
    state = AsyncValue.data(updatedItems);

    // 3. 持久化（异步，不阻塞UI）
    try {
      await _repository.saveCart(updatedItems);
    } catch (e) {
      // 持久化失败，回滚状态
      state = AsyncValue.data(currentItems);
      
      // 显示错误提示
      ref.read(snackbarProvider.notifier).showError('添加失败，请重试');
    }
  }

  /// 更新商品数量
  Future<void> updateQuantity(String productId, int quantity) async {
    if (quantity <= 0) {
      // 数量为0，移除商品
      return removeProduct(productId);
    }

    final currentItems = state.valueOrNull ?? [];
    final index = currentItems.indexWhere(
      (item) => item.product.id == productId,
    );

    if (index < 0) return;

    // 乐观更新
    final updatedItems = [
      ...currentItems.sublist(0, index),
      currentItems[index].copyWith(quantity: quantity),
      ...currentItems.sublist(index + 1),
    ];

    state = AsyncValue.data(updatedItems);

    // 持久化
    try {
      await _repository.saveCart(updatedItems);
    } catch (e) {
      // 回滚
      state = AsyncValue.data(currentItems);
      ref.read(snackbarProvider.notifier).showError('更新失败');
    }
  }

  /// 移除商品
  Future<void> removeProduct(String productId) async {
    final currentItems = state.valueOrNull ?? [];
    final updatedItems = currentItems
        .where((item) => item.product.id != productId)
        .toList();

    // 乐观更新
    state = AsyncValue.data(updatedItems);

    // 持久化
    try {
      await _repository.saveCart(updatedItems);
    } catch (e) {
      // 回滚
      state = AsyncValue.data(currentItems);
      ref.read(snackbarProvider.notifier).showError('移除失败');
    }
  }

  /// 清空购物车
  Future<void> clear() async {
    final currentItems = state.valueOrNull ?? [];

    state = const AsyncValue.data([]);

    try {
      await _repository.clearCart();
    } catch (e) {
      state = AsyncValue.data(currentItems);
      ref.read(snackbarProvider.notifier).showError('清空失败');
    }
  }

  /// 刷新购物车（从服务器同步）
  Future<void> refresh() async {
    state = const AsyncValue.loading();

    state = await AsyncValue.guard(() async {
      // 模拟从API加载
      await Future.delayed(const Duration(seconds: 1));
      return await _repository.loadCart();
    });
  }
}
```

### 4.2 CartRepository实现

```dart
class CartRepository {
  static const _storageKey = 'shopping_cart';
  late final SharedPreferences _prefs;

  CartRepository() {
    _initPrefs();
  }

  Future<void> _initPrefs() async {
    _prefs = await SharedPreferences.getInstance();
  }

  /// 加载购物车
  Future<List<CartItem>> loadCart() async {
    await _initPrefs();
    
    final jsonString = _prefs.getString(_storageKey);
    if (jsonString == null) {
      return [];
    }

    try {
      final List<dynamic> jsonList = json.decode(jsonString);
      return jsonList
          .map((item) => CartItem.fromJson(item as Map<String, dynamic>))
          .toList();
    } catch (e) {
      // 解析失败，返回空列表
      return [];
    }
  }

  /// 保存购物车
  Future<void> saveCart(List<CartItem> items) async {
    await _initPrefs();
    
    final jsonString = json.encode(
      items.map((item) => item.toJson()).toList(),
    );
    
    await _prefs.setString(_storageKey, jsonString);
  }

  /// 清空购物车
  Future<void> clearCart() async {
    await _initPrefs();
    await _prefs.remove(_storageKey);
  }

  /// 同步到服务器（可选）
  Future<void> syncToServer(List<CartItem> items) async {
    // 调用API同步购物车
    // await apiService.syncCart(items);
  }
}
```

## 5. UI组件实现

### 5.1 购物车主页面

```dart
class ShoppingCartScreen extends ConsumerWidget {
  const ShoppingCartScreen({Key? key}) : super(key: key);

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final cartAsync = ref.watch(cartProvider);

    return Scaffold(
      appBar: AppBar(
        title: const Text('购物车'),
        actions: [
          // 清空按钮
          IconButton(
            icon: const Icon(Icons.delete_outline),
            onPressed: () => _showClearDialog(context, ref),
          ),
          // 刷新按钮
          IconButton(
            icon: const Icon(Icons.refresh),
            onPressed: () {
              ref.read(cartProvider.notifier).refresh();
            },
          ),
        ],
      ),
      body: cartAsync.when(
        data: (items) => items.isEmpty
            ? _buildEmptyState()
            : _buildCartList(items, ref),
        loading: () => const Center(child: CircularProgressIndicator()),
        error: (error, stack) => _buildErrorState(error, ref),
      ),
      bottomNavigationBar: cartAsync.maybeWhen(
        data: (items) => items.isNotEmpty ? _buildBottomBar(ref) : null,
        orElse: () => null,
      ),
    );
  }

  Widget _buildEmptyState() {
    return Center(
      child: Column(
        mainAxisAlignment: MainAxisAlignment.center,
        children: [
          Icon(
            Icons.shopping_cart_outlined,
            size: 100,
            color: Colors.grey[300],
          ),
          const SizedBox(height: 16),
          Text(
            '购物车是空的',
            style: TextStyle(
              fontSize: 18,
              color: Colors.grey[600],
            ),
          ),
        ],
      ),
    );
  }

  Widget _buildCartList(List<CartItem> items, WidgetRef ref) {
    return ListView.builder(
      itemCount: items.length,
      itemBuilder: (context, index) {
        return CartItemWidget(
          item: items[index],
          onQuantityChanged: (quantity) {
            ref.read(cartProvider.notifier).updateQuantity(
              items[index].product.id,
              quantity,
            );
          },
          onRemove: () {
            ref.read(cartProvider.notifier).removeProduct(
              items[index].product.id,
            );
          },
        );
      },
    );
  }

  Widget _buildErrorState(Object error, WidgetRef ref) {
    return Center(
      child: Column(
        mainAxisAlignment: MainAxisAlignment.center,
        children: [
          const Icon(Icons.error_outline, size: 60, color: Colors.red),
          const SizedBox(height: 16),
          Text('加载失败: $error'),
          const SizedBox(height: 16),
          ElevatedButton(
            onPressed: () => ref.read(cartProvider.notifier).refresh(),
            child: const Text('重试'),
          ),
        ],
      ),
    );
  }

  Widget _buildBottomBar(WidgetRef ref) {
    final total = ref.watch(cartTotalProvider);
    final itemCount = ref.watch(cartItemCountProvider);

    return Container(
      padding: const EdgeInsets.all(16),
      decoration: BoxDecoration(
        color: Colors.white,
        boxShadow: [
          BoxShadow(
            color: Colors.black.withOpacity(0.1),
            blurRadius: 4,
            offset: const Offset(0, -2),
          ),
        ],
      ),
      child: SafeArea(
        child: Row(
          children: [
            Expanded(
              child: Column(
                mainAxisSize: MainAxisSize.min,
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  Text(
                    '共 $itemCount 件商品',
                    style: const TextStyle(
                      fontSize: 14,
                      color: Colors.grey,
                    ),
                  ),
                  const SizedBox(height: 4),
                  Text(
                    '¥${total.toStringAsFixed(2)}',
                    style: const TextStyle(
                      fontSize: 24,
                      fontWeight: FontWeight.bold,
                      color: Colors.red,
                    ),
                  ),
                ],
              ),
            ),
            ElevatedButton(
              onPressed: () => _handleCheckout(ref),
              style: ElevatedButton.styleFrom(
                backgroundColor: Colors.red,
                padding: const EdgeInsets.symmetric(
                  horizontal: 32,
                  vertical: 16,
                ),
              ),
              child: const Text(
                '去结算',
                style: TextStyle(fontSize: 16),
              ),
            ),
          ],
        ),
      ),
    );
  }

  void _showClearDialog(BuildContext context, WidgetRef ref) {
    showDialog(
      context: context,
      builder: (context) => AlertDialog(
        title: const Text('清空购物车'),
        content: const Text('确定要清空购物车吗？'),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(context),
            child: const Text('取消'),
          ),
          TextButton(
            onPressed: () {
              ref.read(cartProvider.notifier).clear();
              Navigator.pop(context);
            },
            child: const Text('确定'),
          ),
        ],
      ),
    );
  }

  void _handleCheckout(WidgetRef ref) {
    // 跳转到结算页面
    // Navigator.push(context, ...);
  }
}
```

### 5.2 购物车项目组件

```dart
class CartItemWidget extends StatelessWidget {
  const CartItemWidget({
    Key? key,
    required this.item,
    required this.onQuantityChanged,
    required this.onRemove,
  }) : super(key: key);

  final CartItem item;
  final ValueChanged<int> onQuantityChanged;
  final VoidCallback onRemove;

  @override
  Widget build(BuildContext context) {
    return Dismissible(
      key: Key(item.product.id),
      direction: DismissDirection.endToStart,
      background: Container(
        alignment: Alignment.centerRight,
        padding: const EdgeInsets.only(right: 20),
        color: Colors.red,
        child: const Icon(Icons.delete, color: Colors.white),
      ),
      confirmDismiss: (direction) => _confirmDelete(context),
      onDismissed: (_) => onRemove(),
      child: Card(
        margin: const EdgeInsets.symmetric(horizontal: 16, vertical: 8),
        child: Padding(
          padding: const EdgeInsets.all(12),
          child: Row(
            children: [
              // 商品图片
              ClipRRect(
                borderRadius: BorderRadius.circular(8),
                child: Image.network(
                  item.product.imageUrl,
                  width: 80,
                  height: 80,
                  fit: BoxFit.cover,
                  errorBuilder: (_, __, ___) => Container(
                    width: 80,
                    height: 80,
                    color: Colors.grey[200],
                    child: const Icon(Icons.image, color: Colors.grey),
                  ),
                ),
              ),
              const SizedBox(width: 12),
              
              // 商品信息
              Expanded(
                child: Column(
                  crossAxisAlignment: CrossAxisAlignment.start,
                  children: [
                    Text(
                      item.product.name,
                      style: const TextStyle(
                        fontSize: 16,
                        fontWeight: FontWeight.w500,
                      ),
                      maxLines: 2,
                      overflow: TextOverflow.ellipsis,
                    ),
                    const SizedBox(height: 8),
                    Text(
                      '¥${item.product.price.toStringAsFixed(2)}',
                      style: const TextStyle(
                        fontSize: 18,
                        color: Colors.red,
                        fontWeight: FontWeight.bold,
                      ),
                    ),
                  ],
                ),
              ),
              
              // 数量控制
              _QuantityControl(
                quantity: item.quantity,
                onChanged: onQuantityChanged,
              ),
            ],
          ),
        ),
      ),
    );
  }

  Future<bool?> _confirmDelete(BuildContext context) {
    return showDialog<bool>(
      context: context,
      builder: (context) => AlertDialog(
        title: const Text('移除商品'),
        content: Text('确定要移除 ${item.product.name} 吗？'),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(context, false),
            child: const Text('取消'),
          ),
          TextButton(
            onPressed: () => Navigator.pop(context, true),
            child: const Text('确定'),
          ),
        ],
      ),
    );
  }
}

// 数量控制组件
class _QuantityControl extends StatelessWidget {
  const _QuantityControl({
    required this.quantity,
    required this.onChanged,
  });

  final int quantity;
  final ValueChanged<int> onChanged;

  @override
  Widget build(BuildContext context) {
    return Container(
      decoration: BoxDecoration(
        border: Border.all(color: Colors.grey[300]!),
        borderRadius: BorderRadius.circular(20),
      ),
      child: Row(
        mainAxisSize: MainAxisSize.min,
        children: [
          _buildButton(
            icon: Icons.remove,
            onPressed: quantity > 1 ? () => onChanged(quantity - 1) : null,
          ),
          Container(
            padding: const EdgeInsets.symmetric(horizontal: 12),
            child: Text(
              '$quantity',
              style: const TextStyle(
                fontSize: 16,
                fontWeight: FontWeight.bold,
              ),
            ),
          ),
          _buildButton(
            icon: Icons.add,
            onPressed: quantity < 99 ? () => onChanged(quantity + 1) : null,
          ),
        ],
      ),
    );
  }

  Widget _buildButton({
    required IconData icon,
    required VoidCallback? onPressed,
  }) {
    return InkWell(
      onTap: onPressed,
      borderRadius: BorderRadius.circular(20),
      child: Padding(
        padding: const EdgeInsets.all(8),
        child: Icon(
          icon,
          size: 20,
          color: onPressed != null ? Colors.black : Colors.grey,
        ),
      ),
    );
  }
}
```

## 6. 测试

### 6.1 单元测试

```dart
void main() {
  group('CartNotifier', () {
    late ProviderContainer container;

    setUp(() {
      container = ProviderContainer(
        overrides: [
          cartRepositoryProvider.overrideWithValue(MockCartRepository()),
        ],
      );
    });

    tearDown(() {
      container.dispose();
    });

    test('初始状态为空购物车', () async {
      final cart = await container.read(cartProvider.future);
      expect(cart, isEmpty);
    });

    test('添加商品', () async {
      final product = Product(
        id: '1',
        name: 'Test Product',
        price: 99.99,
        imageUrl: 'https://example.com/image.jpg',
      );

      await container.read(cartProvider.notifier).addProduct(product);

      final cart = await container.read(cartProvider.future);
      expect(cart.length, 1);
      expect(cart[0].product.id, '1');
      expect(cart[0].quantity, 1);
    });

    test('更新数量', () async {
      final product = Product(
        id: '1',
        name: 'Test Product',
        price: 99.99,
        imageUrl: 'https://example.com/image.jpg',
      );

      await container.read(cartProvider.notifier).addProduct(product);
      await container.read(cartProvider.notifier).updateQuantity('1', 5);

      final cart = await container.read(cartProvider.future);
      expect(cart[0].quantity, 5);
    });

    test('移除商品', () async {
      final product = Product(
        id: '1',
        name: 'Test Product',
        price: 99.99,
        imageUrl: 'https://example.com/image.jpg',
      );

      await container.read(cartProvider.notifier).addProduct(product);
      await container.read(cartProvider.notifier).removeProduct('1');

      final cart = await container.read(cartProvider.future);
      expect(cart, isEmpty);
    });

    test('计算总价', () async {
      final product1 = Product(
        id: '1',
        name: 'Product 1',
        price: 10.0,
        imageUrl: 'url',
      );
      final product2 = Product(
        id: '2',
        name: 'Product 2',
        price: 20.0,
        imageUrl: 'url',
      );

      await container.read(cartProvider.notifier).addProduct(product1, quantity: 2);
      await container.read(cartProvider.notifier).addProduct(product2, quantity: 3);

      final total = container.read(cartTotalProvider);
      expect(total, 80.0); // 10*2 + 20*3 = 80
    });
  });
}
```

### 6.2 Widget测试

```dart
void main() {
  testWidgets('购物车显示商品列表', (tester) async {
    final container = ProviderContainer(
      overrides: [
        cartProvider.overrideWith(() => MockCartNotifier()),
      ],
    );

    await tester.pumpWidget(
      UncontrolledProviderScope(
        container: container,
        child: const MaterialApp(
          home: ShoppingCartScreen(),
        ),
      ),
    );

    expect(find.byType(CartItemWidget), findsWidgets);
  });
}
```

## 7. 优化技巧

### 7.1 性能优化

**使用select精确监听**：
```dart
// ❌ 不好：整个购物车变化都rebuild
final cart = ref.watch(cartProvider);

// ✅ 好：只监听数量变化
final itemCount = ref.watch(
  cartProvider.select((cart) => cart.valueOrNull?.length ?? 0),
);
```

**使用family缓存商品**：
```dart
final productProvider = FutureProvider.family<Product, String>((ref, id) async {
  // 缓存每个商品的数据
  return await apiService.getProduct(id);
});
```

### 7.2 错误处理

**统一的错误处理**：
```dart
Future<void> _executeWithErrorHandling(
  Future<void> Function() action,
  String errorMessage,
) async {
  try {
    await action();
  } catch (e) {
    ref.read(snackbarProvider.notifier).showError(errorMessage);
    // 记录错误日志
    ref.read(loggerProvider).error(e);
  }
}
```

### 7.3 防抖动

```dart
Timer? _debounceTimer;

void _debouncedSave(List<CartItem> items) {
  _debounceTimer?.cancel();
  _debounceTimer = Timer(const Duration(milliseconds: 500), () {
    _repository.saveCart(items);
  });
}
```

## 8. 总结

### 8.1 关键设计点

```
✅ 状态分层清晰
  - UI层只负责展示
  - State层管理状态
  - Service层处理业务
  - Data层处理持久化

✅ 职责单一
  - CartNotifier：业务逻辑
  - CartRepository：数据持久化
  - Widget：UI展示

✅ 乐观更新
  - 立即更新UI
  - 异步持久化
  - 失败时回滚

✅ 错误处理
  - 每个操作都有错误处理
  - 用户友好的提示
  - 状态回滚机制
```

### 8.2 Riverpod最佳实践

```
✅ AsyncNotifier管理异步状态
✅ Provider计算派生状态
✅ 依赖注入service
✅ select精确监听
✅ 完整的错误处理
```

### 8.3 可扩展性

这个购物车组件可以轻松扩展：
- ✅ 添加优惠券功能
- ✅ 添加库存检查
- ✅ 集成支付流程
- ✅ 添加多规格商品
- ✅ 添加收藏功能

这就是一个完整的、生产级的Riverpod业务组件开发实战！


[← 返回状态管理目录](README.md)

# Riverpod 状态管理详解 2 - ConsumerStatefulWidget

上一篇主要介绍了 Riverpod 大体的实现原理，后面的小节我们进入具体的模块来剖析一下细节。

这一篇我们来看看 `Consumer`/`ConsumerWidget` 和 `ConsumerStatefulWidget`，Widget 层的几个类。

## 继承关系

```
ConsumerStatefulWidget (abstract)
    ↑
    └── ConsumerWidget (abstract)
            ↑
            └── Consumer<StateT> (final)
```

最后都是对 `ConsumerStatefulWidget` 的封装。

## ConsumerState 和 ref

在 `ConsumerStatefulWidget` 中自定义了 `ConsumerStatefulElement` 以及 `ConsumerState`。

我们常用的 `ref` 是在 `ConsumerState` 定义的：
```dart
abstract class ConsumerState<WidgetT extends ConsumerStatefulWidget> extends State<WidgetT> {
  late final ref = context as WidgetRef;
}
```

可以看到 `ref` 本质是 `context`，也就是说就是当前的 `ConsumerStatefulElement`。

## ConsumerStatefulElement

接下来我们看看 `ConsumerStatefulElement`。

### 属性列表:
```dart
/// 顶层的状态仓库, 使用InheritedWidget
late ProviderContainer container = ProviderScope.containerOf(this);

/// 依赖项：当前这个组件依赖了哪些 provider，key 是 provider，value 是一个订阅
/// 是什么后边再说，这里只要知道这里保存了自己依赖的 provider
var _dependencies = <ProviderListenable<Object?>, ProviderSubscription<Object?>>{};
Map<ProviderListenable<Object?>, ProviderSubscription<Object?>>? _oldDependencies;

/// listen 的回调：我们在调用 ref.listen 的时候回调函数保存在这里
/// _listeners 中的回调生命周期是 ConsumerStatefulElement 自己管理的，后面会分析
final _listeners = <ProviderSubscription<Object?>>[];

/// 与上边 _listeners 不一样的是这个是手动管理的 listen 的回调
/// 但是我们在后边会了解到，一些 _manualListeners 也是不需要我们手动管理生命周期的
List<ProviderSubscription<Object?>>? _manualListeners;
```

### build：

**代码解读：**

```dart
@override
Widget build() {
  ...

  try {
    // 1. 保存旧的依赖关系
    _oldDependencies = _dependencies;
    
    // 2. 清空所有旧的监听器（这些监听器会在 super.build() 中重新创建）
    for (var i = 0; i < _listeners.length; i++) {
      _listeners[i].close();
    }
    _listeners.clear();
    
    // 3. 清空当前依赖（在 super.build() 执行时会重新填充）
    _dependencies = {};
    
    // 4. 执行真正的 build 方法，在这个过程中会重新调用 watch/listen
    return super.build();
  } finally {
    // 5. 清理不再使用的依赖（在 super.build() 中没有被重新 watch 的）
    for (final dep in _oldDependencies!.values) {
      dep.close();
    }
    _oldDependencies = null;
  }
}
```

**核心机制：**

这里可以看到 `_dependencies` 和 `_oldDependencies` 的关系：

- 每次 `build` 会将 `_dependencies` 赋值给 `_oldDependencies`
- 然后 `_dependencies` 清空
- 在执行完真正的 `super.build()` 后再将 `_oldDependencies` 中依赖的 `provider` 移除

这个操作很自然想到应该是在 `super.build()` 执行过程中所有的监听重新添加到了 `_dependencies` 中，并且从 `_oldDependencies` 中移除，最后将不再使用的 `provider` 关闭。这样一来，就能时刻保持状态依赖关系的同步，防止内存泄漏。

这里还有一个 `_listeners` 大家注意到了吧，每次 `build` 都会清空，`super.build` 调用后会重新加进来，同上，也是为了防止内存泄漏，上次的监听这次 rebuild 后可能不再需要了。

**问题：** 为什么 `_listeners` 只有一个，并没有 `_oldListeners`，而 `_dependencies` 对应会有一个 `_oldDependencies`？

### watch：

**代码解读：**

```dart
@override
StateT watch<StateT>(ProviderListenable<StateT> target) {
  return _dependencies
      .putIfAbsent(target, () {
        // 1. 尝试从旧依赖中获取（如果存在，说明这个 provider 在之前的 build 中也使用了）
        final oldDependency = _oldDependencies?.remove(target);

        if (oldDependency != null) {
          // 2. 如果旧依赖存在，直接复用（避免重复创建监听）
          return oldDependency;
        }

        // 3. 如果旧依赖不存在，创建新的监听
        // 回调函数是 markNeedsBuild，当 provider 变化时会触发组件重建
        final sub = container.listen<StateT>(target, (_, _) => markNeedsBuild());
        return sub;
      })
      .readSafe() as StateT;  // 4. 读取当前值并返回
}
```

**工作流程：**

这里看到每次 `watch`，会在 `_dependencies` 中加入 `provider` 和回调函数（如果 `_dependencies` 中没有这个 `provider` 的情况下）。这个回调函数印证了我们上边说的，所有的监听重新添加到了 `_dependencies` 中，并且从 `_oldDependencies` 中移除。

如果 `_oldDependencies` 中没有，说明是新的需要重新建立监听，这里我们看到了熟悉的 `markNeedsBuild`。

### listen：

**代码解读：**

```dart
@override
void listen<StateT>(
  ProviderListenable<StateT> provider,
  void Function(StateT? previous, StateT value) listener, {
  void Function(Object error, StackTrace stackTrace)? onError,
}) {
  // 1. 创建监听，使用传入的自定义回调函数（而不是 markNeedsBuild）
  final sub = container.listen<StateT>(provider, listener, onError: onError);
  
  // 2. 将订阅添加到 _listeners 中（生命周期与 build 同步）
  _listeners.add(sub);
}
```

**与 watch 的区别：**

和 `watch` 有些类似，只不过这里的回调函数不是 `markNeedsBuild` 而是传入的 `listener`。这意味着：
- `watch` 会自动触发组件重建
- `listen` 只执行自定义回调，不会自动重建组件

### listenManual：

**代码解读：**

```dart
@override
ProviderSubscription<ValueT> listenManual<ValueT>(
  ProviderListenable<ValueT> provider,
  void Function(ValueT? previous, ValueT next) listener, {
  void Function(Object error, StackTrace stackTrace)? onError,
  bool fireImmediately = false,
}) {
  // 1. 初始化手动监听列表（如果不存在）
  final listeners = _manualListeners ??= [];

  // 2. 获取 container
  final container = ProviderScope.containerOf(this, listen: false);

  // 3. 创建监听订阅
  final sub = container.listen<ValueT>(provider, listener, onError: onError, fireImmediately: fireImmediately);

  // 4. 包装 onClose 回调：当订阅关闭时，自动从 _manualListeners 中移除
  final previousOnClose = sub.impl.onClose;
  sub.impl.onClose = () {
    previousOnClose?.call();  // 先执行原来的 onClose
    _manualListeners?.remove(sub);  // 然后从列表中移除
  };

  // 5. 添加到手动监听列表
  listeners.add(sub);

  // 6. 返回订阅对象，允许手动关闭
  return sub;
}
```

**使用场景：**

`listenManual` 返回值是个 `ProviderSubscription`，也就是说你可以手动来进行关闭监听。关闭之后 onClose 回调回来后从 `_manualListeners` 将监听移除掉，但是感觉这一步多余，因为看代码知道 `ProviderSubscription.close` 会判断当前回调是否已经关闭，不会报错崩溃。

**适用场景：**
- 在 `initState` 中创建监听
- 需要手动控制监听的生命周期
- 不需要与 `build` 方法同步的监听

**ProviderSubscription.close 的实现：**

```dart
void close() {
  // 1. 防止重复关闭
  if (_closed) return;

  // 2. 执行关闭回调（如果有）
  onClose?.call();
  
  // 3. 从依赖的 Element 中移除订阅，并标记为已关闭
  _listenedElement.removeDependentSubscription(this, () {
    _closed = true;
  });
}
```

**安全机制：**
- `_closed` 标志防止重复关闭
- 即使多次调用 `close()` 也不会报错

### unmount：

**代码解读：**

```dart
@override
void unmount() {
  super.unmount();

  // 1. 关闭所有依赖的 provider 订阅（防止内存泄漏）
  for (final dependency in _dependencies.values) {
    dependency.close();
  }
  
  // 2. 关闭所有自动管理的监听器
  for (var i = 0; i < _listeners.length; i++) {
    _listeners[i].close();
  }
  
  // 3. 关闭所有手动管理的监听器（框架兜底，即使没有手动关闭也会在这里清理）
  final manualListeners = _manualListeners?.toList();
  if (manualListeners != null) {
    for (final listener in manualListeners) {
      listener.close();
    }
    _manualListeners = null;
  }
}
```

**清理机制：**

当组件从组件树中移除时，会清理所有相关的订阅和监听，确保没有内存泄漏。即使开发者忘记手动关闭 `listenManual` 创建的订阅，框架也会在这里自动清理。


和上边的 `build` 一起分析，我们能 get 到一个使用规范：

- `_listeners` 的生命周期与 `build` 同步，所以 `_listeners` 只应该在 `build` 中使用
- `_manualListeners` 是在 `build` 外部使用的，比如在 `initState` 中，随时可以手动关闭，并且框架兜底关闭
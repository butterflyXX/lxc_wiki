[← 返回状态管理目录](README.md)

# Riverpod 之 Provider 1 - 从 NotifierProvider 入手

上一节已经说明了 UI 层的绑定和关联，其实是 `flutter_riverpod` 框架的内容。从这一节开始，我将深入到 `riverpod` 核心库，去分析一下：

- `provider` 是什么，都有哪些类型的 `provider`
- `provider` 中的 `ref` 是什么
- `ProviderElement` 是什么
- 什么是 `notifier`，都有哪些 `notifier`
- `watch/read(provider)` 返回的是什么
- 永久的 `provider` 和自动销毁的 `provider`

等等...，先列这么多吧。

由于直接使用 `Provider` 不能看出状态变化，因为 `Provider` 是只读状态，所以我们直接从 `NotifierProvider` 入手来学一下。

## Demo 案例

先上一个 demo 案例：

```dart
final testProvider = NotifierProvider.autoDispose<TestNotifier, int>(() => TestNotifier());

class TestNotifier extends Notifier<int> {
  @override
  int build() {
    return 0;
  }

  void add() {
    state++;
  }

  void subtract() {
    state--;
  }
}

class TestWidget extends ConsumerWidget {
  const TestWidget({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    return Scaffold(
      body: Center(
        child: Text(ref.watch(testProvider).toString()),
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: () {
          ref.read(testProvider.notifier).add();
        },
        child: Text('Add'),
      ),
    );
  }
}
```

## NotifierProvider 的继承关系

先看下 `NotifierProvider` 的继承关系：

```
NotifierProvider (final)
with (LegacyProviderMixin<ValueT>)
    ↑
    └── $NotifierProvider (abstract) 通知类型的provider基类
            ↑
            └── $ClassProvider (abstract) 类类型provider基类
                    ↑
                    └── $ProviderBaseImpl (abstract) 所有provider基类
                            ↑
                            └── ProviderBase (sealed) 所有provider基类
                            implements (ProviderListenable<StateT>, Refreshable<StateT>, _ProviderOverride)
                                    ↑
                                    └── ProviderOrFamily (sealed) 所有provider/Family基类
```

## 重要方法梳理

先整体过一下重要的方法。

### NotifierProvider

```dart
/// 创建Notifier
final NotifierT Function() _createNotifier;
NotifierT create() => _createNotifier();
```

### $NotifierProvider

```dart
$NotifierProviderElement<NotifierT, StateT> $createElement(
  $ProviderPointer pointer,
) {
  return $NotifierProviderElement(pointer);
}
```

`$NotifierProvider` 的 `$createElement` 返回的是 `$NotifierProviderElement`，可以看到 `provider` 会创建 `element`，对比 `Widget` 和 `WidgetElement` 是不是会好理解。

### $ClassProvider

```dart
Refreshable<NotifierT> get notifier {
  return ProviderElementProxy<NotifierT, StateT>(
    this,
    (element) =>
        (element as $ClassProviderElement<NotifierT, StateT, ValueT, CreatedT>)
            .classListenable,
  );
}
```

`$ClassProvider` 中定义了一个重要的 getter 方法 `notifier`，也就是我们常用的 `ref.read(testProvider.notifier)`，可以看到创建的是一个代理类 `ProviderElementProxy`。

### $ProviderBaseImpl

```dart
ProviderProviderSubscription<StateT> _addListener(
  Node source,
  void Function(StateT? previous, StateT next) listener, {
  required void Function(Object error, StackTrace stackTrace) onError,
  required void Function()? onDependencyMayHaveChanged,
  required bool weak,
}) {
  /// 获取ProviderElement
  final element = source.readProviderElement(this);

  /// 是否马上挂载element
  if (!weak) element.flush();

  /// 创建订阅
  return ProviderProviderSubscription<StateT>(
    source: source,
    listenedElement: element,
    weak: weak,
    listener: listener,
    onError: onError,
  );
}
```

`_addListener` 是用来真正创建并且监听 `element` 的：

- 创建/获取 `element`
- 挂载 `element`
- 返回订阅

这就是 `NotifierProvider` 及其父类中比较重要的方法，先梳理一下。下面通过断点来一步步看下 UI 组件是如何与状态进行双向绑定的，如何刷新的。

## 从 WidgetRef.watch 开始追踪

我们从 `WidgetRef.watch` 进行断点：

```dart
@override
StateT watch<StateT>(ProviderListenable<StateT> target) {
  ...
  return _dependencies
      .putIfAbsent(target, () {
        final oldDependency = _oldDependencies?.remove(target);

        if (oldDependency != null) {
          return oldDependency;
        }

        final sub = container.listen<StateT>(target, (_, _) => markNeedsBuild());
        ...
        return sub;
      })
      .readSafe() as StateT;
}
```

`ref.watch` 上一小节已经说过了，我们直接来看 `container.listen`。

## container.listen 方法

### 方法实现

```dart
ProviderSubscription<StateT> listen<StateT>(
  ProviderListenable<StateT> provider,
  void Function(StateT? previous, StateT next) listener, {
  bool fireImmediately = false,
  bool weak = false,
  void Function(Object error, StackTrace stackTrace)? onError,
}) {
  // 真正创建订阅
  final sub = provider._addListener(
    this,
    listener,
    weak: weak,
    onError: onError ?? defaultOnError,
    onDependencyMayHaveChanged: null,
  );

  // 是否立即触发一次回调
  _handleFireImmediately(container, sub, fireImmediately: fireImmediately);

  // 将订阅添加到element中
  sub.impl._listenedElement.addDependentSubscription(sub.impl);

  return sub;
}
```

### _addListener 方法

这里 `_addListener` 被调用了：

```dart
/// 获取ProviderElement
final element = source.readProviderElement(this);

/// 是否马上挂载element
if (!weak) element.flush();

/// 创建订阅
return ProviderProviderSubscription<StateT>(
  source: source,
  listenedElement: element,
  weak: weak,
  listener: listener,
  onError: onError,
);
```

这里的 `source` 我们可以简单理解成状态仓库 `ProviderContainer`。

由于 `ProviderContainer` 内部属性嵌套，`ProviderContainer` 持有 `ProviderPointerManager` 来管理状态节点，`ProviderPointerManager` 又持有 `ProviderDirectory` 来保存状态节点，所以这里我们简化一下，不管是 `ProviderPointerManager` 还是 `ProviderDirectory` 的方法都看做是 `ProviderContainer` 的。

### readProviderElement 和 mount

`readProviderElement` 最终会调用到 `mount`：

```dart
$ProviderPointer mount(
  ProviderBase<Object?> origin, {
  required ProviderContainer currentContainer,
}) {
  // 更新并插入状态节点
  final pointer = upsertPointer(origin, currentContainer: currentContainer);

  // 如果节点没有绑定状态元素,需要创建状态元素,并且进行绑定
  if (pointer.element == null) {
    ProviderElement? element;
    ...
    element = origin.$createElement(pointer);
    ...
    pointer.element = element;
  }

  return pointer;
}
```

这里可以简单理解 `pointer` 是 `element`：

```dart
class $ProviderPointer implements _PointerBase {
  /// provider
  final ProviderBase<Object?> origin;
  /// element
  ProviderElement? element;

  /// container
  final ProviderContainer targetContainer;
}

final HashMap<ProviderBase<Object?>, $ProviderPointer> pointers;
```

这个是 `pointer` 大概的数据结构，以及在 `ProviderContainer` 中用来保存 `provider` 和 `pointer` 的数据类型，是一个 `HashMap`。

如果更新并插入 `pointer` 后发现还没有绑定 `element`，会进行绑定，调用 `provider` 的 `$createElement`，`NotifierProvider` 返回的就是 `$NotifierProviderElement`，上边有提到。

### $NotifierProviderElement 的继承关系

梳理一下 `$NotifierProviderElement` 的继承关系：

```
$NotifierProviderElement (class)
with (SyncProviderElement<ValueT>)
    ↑
    └── $ClassProviderElement (abstract) 类类型providerElement基类
    with (ElementWithFuture)
            ↑
            └── ProviderElement(abstract) 所有providerElement基类
```

`ProviderElement` 需要传一个 `$ProviderPointer`：

```dart
$NotifierProviderElement(pointer)
```

`$NotifierProviderElement` 没有什么东西，我们来看它的父类 `$ClassProviderElement`：

```dart
abstract class $ClassProviderElement<NotifierT> extends ProviderElement<StateT, ValueT> with ElementWithFuture {
  $ClassProviderElement(super.pointer)
    // provider 是从 pointer中获取的
    : provider = pointer.origin as $ClassProvider<NotifierT, StateT, ValueT, CreatedT>;

  /// provider
  $ClassProvider<NotifierT, StateT, ValueT, CreatedT> provider;

  /// 我们平时写的Notifier监听器
  final classListenable = $Observable<NotifierT>();

  WhenComplete create(
    $Ref<StateT, ValueT> ref,
  ) {
    // 创建Notifier
    final result =
        classListenable.result ??= $Result.guard(() {
          final notifier = provider.create();
          if (notifier._element != null) {
            throw StateError(alreadyInitializedError);
          }

          notifier._element = this;
          return notifier;
        });

    switch (result) {
      case $ResultData():
        try {
          switch (_runNotifierBuildOverride) {
            case final override?:
              handleCreate(ref, () => override(ref, result.value));
            case null:
              // result.value 就是创建的Notifier
              // 调用notifier的runBuild,初始化状态
              result.value.runBuild();
          }
        } catch (err, stack) {
          handleError(ref, err, stack);
        }
      case $ResultError():
        handleError(ref, result.error, result.stackTrace);
    }

    return null;
  }

  void handleCreate(Ref ref, CreatedT Function() created);
  void handleError(Ref ref, Object error, StackTrace stackTrace);

  // 是否通知UI组件或者其他provider 执行rebuild
  bool updateShouldNotify(StateT previous, StateT next) {
    return classListenable.result?.value?.updateShouldNotify(previous, next) ??
        super.updateShouldNotify(previous, next);
  }
}
```

可以看到在 `$ClassProviderElement` 的 `create` 方法简化一下，其实干了 2 件事：

- `provider.create()`，上边也提到了实际是创建 Notifier 实例
- `result.value.runBuild()`，执行 Notifier 实例的 `rebuild` 方法

`ProviderElement` 有些庞大，哪里用到哪里在做分析。

### 总结：readProviderElement 的执行流程

以上都是执行 `_addListener` 中 `final element = source.readProviderElement(this)` 的相关调用，总结一下：

1. 创建 `ProviderPointer`：保存在 `ProviderContainer`
2. 创建 `ProviderElement`：保存在 `ProviderPointer`
3. 返回 `ProviderElement`

## element.flush() 方法

拿到 `element` 后，我们返回头继续执行 `_addListener` 中下一个方法 `element.flush()`，这里调用了 `element` 的 `mount`：

```dart
void mount() {
  /// 创建ref,实际是持有了element,简单理解ref就是element
  final ref = this.ref = $Ref(this, isFirstBuild: true, isReload: false);

  /// 初始值,是null
  final initialState = value;

  try {
    /// 创建状态
    buildState(ref);

    /// 通知监听者
    _notifyListeners(
      value,
      initialState,
      isFirstBuild: true,
      checkUpdateShouldNotify: false,
    );
  }
}

@internal
void buildState($Ref<StateT, ValueT> ref) {
  ...
  try {
    // 调用provider的create方法
    final whenComplete = create(ref) ?? (cb) => cb();
    ...
  }
}
```

这里会创建 `Ref`，分析代码可以先简单理解是当前的 `ProviderElement`。对比一下上一节中的 `WidgetRef`，在 `riverpod` 中虽然移除了对 `BuildContext` 的依赖，但是转头依赖了 `Ref` 这样一个东西，代表当前的自己。在 `ProviderElement` 中就代表当前的 `ProviderElement`，在 `WidgetElement` 中就代表当前的 `WidgetElement`，我觉得理解他是身份证很形象。

`buildState(ref)` 会真正开始创建 `State`，调用了 `create`，`create` 实现在 `$ClassProviderElement` 中，上边已经做了分析创建 `Notifier` 实例，执行 `Notifier.rebuild` 方法进行初始化。

`_notifyListeners` 目前没啥用，刚刚初始化哪来的监听者？但是 `_notifyListeners` 方法很重要，是状态同步给监听者的关键方法，我们后边做分析。

### 总结：element.flush() 的执行流程

以上是执行 `_addListener` 中 `element.flush()` 的相关方法调用，简单总结就是：

1. 创建 `Notifier`
2. 调用 `Notifier.rebuild`

## ProviderProviderSubscription

继续 `_addListener`，执行完 `element.flush()` 后，将 `ProviderElement` 和一个回调函数，包装成一个订阅对象返回。

返回了 `ProviderProviderSubscription`，`ProviderProviderSubscription` 的继承关系：

```
ProviderProviderSubscription (final)
    ↑
    └── ProviderSubscriptionImpl (sealed)
            ↑
            └── ProviderSubscription (sealed)
```

以上就是 `_addListener` 方法执行过程中的方法调用，简单一句话总结：**创建真实的状态，放入 `ProviderContainer` 中**。

## container.listen 的完整流程

`_addListener` 执行完后，返回 `listen` 方法，继续执行 `_handleFireImmediately(container, sub, fireImmediately: fireImmediately)`，这个方法是是否立即执行一次回调。

继续执行 `sub.impl._listenedElement.addDependentSubscription(sub.impl)`，简单描述一下就是 element 将当前这个订阅添加到 `dependents` 中：

```dart
void addDependentSubscription(ProviderSubscriptionImpl<Object?> sub) {
  _onChangeSubscription(sub, () {
    final dependents = this.dependents ??= [];
    dependents.add(sub);
  });
}

List<ProviderSubscriptionImpl<Object?>>? dependents; // 监听者,实际就是订阅回调
```

## 总结：container.listen 整体实现

以上就是 `container.listen` 整体实现，平铺展开总结如下：

### _addListener 部分：

1. 创建 `ProviderPointer`：保存在 `ProviderContainer`
2. 创建 `ProviderElement`：保存在 `ProviderPointer`
3. 返回 `ProviderElement`
4. 创建 `Notifier`
5. 调用 `Notifier.rebuild`
6. 创建订阅对象返回

### _handleFireImmediately 部分：

7. 是否立即执行一次回调

### addDependentSubscription 部分：

8. 将订阅 `sub` 添加到 `ProviderElement` 的 `dependents` 中

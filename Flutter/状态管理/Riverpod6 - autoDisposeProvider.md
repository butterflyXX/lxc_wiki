[← 返回状态管理目录](README.md)

# Riverpod 6 - autoDisposeProvider 生命周期同步机制

之前小节已经对 `read`/`watch` 的实现原理进行了代码级别分析，这一篇我们来分析 `Riverpod` 中页面级别的状态如何与页面生命周期同步，也就是 `autoDispose` 的实现原理。

## 祖传demo：
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

## autoDispose 调用链

```dart
static const autoDispose = AutoDisposeNotifierProviderBuilder();

NotifierProvider<NotifierT, StateT>
  call<NotifierT extends Notifier<StateT>, StateT>(
    NotifierT Function() create, {
    String? name,
    Iterable<ProviderOrFamily>? dependencies,
    Retry? retry,
  }) {
    return NotifierProvider<NotifierT, StateT>(
      create,
      name: name,
      isAutoDispose: true,
      dependencies: dependencies,
      retry: retry,
    );
  }
```

在 `dart` 中，`call` 方法可以直接通过实例名调用。  
`NotifierProvider.autoDispose<TestNotifier, int>(() => TestNotifier())` 本质调用的就是 `call` 方法，返回的是一个 `NotifierProvider`。唯一与普通 `NotifierProvider` 不同的是，`isAutoDispose` 被设置成了 `true`；如果不设置，默认是 `false`。

```dart
 NotifierProvider(
    this._createNotifier, {
    super.name,
    super.dependencies,
    super.isAutoDispose = false,
    super.retry,
  }) : super(
         $allTransitiveDependencies: computeAllTransitiveDependencies(
           dependencies,
         ),
         from: null,
         argument: null,
       );
```

## mayNeedDispose：是否需要销毁

上一节提到了 `mayNeedDispose()`，这个方法用来判断 provider 要不要销毁：

```dart
void mayNeedDispose() {
    if (provider.isAutoDispose) {
      final links = ref?._keepAliveLinks;

      if (!isActive && (links == null || links.isEmpty)) {
        container.scheduler.scheduleProviderDispose(this);
      }
    }
  }
```

有几个重要属性：

```dart
/// 是否活跃,是监听者数量 - 不活跃监听者数量
bool get isActive => (listenerCount - pausedActiveSubscriptionCount) > 0;

/// 监听者数量
int get listenerCount => dependents?.length ?? 0;

/// 不活跃监听者数量
var pausedActiveSubscriptionCount = 0;
```

`ref?._keepAliveLinks`：

```dart
KeepAliveLink keepAlive() {
    _throwIfInvalidUsage();

    final links = _keepAliveLinks ??= [];

    late KeepAliveLink link;
    link = KeepAliveLink._(() {
      if (links.remove(link)) {
        if (links.isEmpty) _element.mayNeedDispose();
      }
    });
    links.add(link);

    return link;
  }
```

这是一个用来自由控制 `provider` 生命周期的方法。  
比如我们可以这样写：

```dart
import 'package:flutter_riverpod/misc.dart';

KeepAliveLink? aliveLink;
class TestNotifier extends Notifier<int> {
  @override
  int build() {
    aliveLink = ref.keepAlive();
    return 0;
  }

  void add() {
    state++;
  }

  void subtract() {
    state--;
  }
}
```

这个时候，即使 `watch` 的页面销毁了，`provider` 也不会销毁，只有你手动调用 `aliveLink?.close()` 才会销毁。

通过 `mayNeedDispose` 及其相关属性的分析可以看到，`Riverpod` 采用了引用计数的方式维护 `provider` 生命周期。引用计数为 0 时，就会进入销毁流程。

下面通过断点来看一下 `mayNeedDispose` 的调用时机。

## 阶段 1：进入页面

首先在 `watch` 阶段就会调用。
![image.png](https://upload-images.jianshu.io/upload_images/5976114-41fb3b4edab9e076.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到调用了 `readSafe`。由于 `watch` 时（之前讲过）`dependents` 会 `+1`，所以 `watch` 之后的 `read` 不会导致 `autoDisposeProvider` 销毁。

## 阶段 2：退出页面

![image.png](https://upload-images.jianshu.io/upload_images/5976114-e33b781014395fe8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

退出页面时，`ConsumerStatefulElement` 会调用 `unmount`，所有监听订阅执行 `close`。

```dart
void close() {
    if (_closed) return;

    onClose?.call();
    _listenedElement.removeDependentSubscription(this, () {
      _closed = true;
    });
  }
```

会调用到 `providerElement` 的 `removeDependentSubscription` 方法：

```dart
void removeDependentSubscription(
    ProviderSubscription sub,
    void Function() apply,
  ) {
    _assertContainsDependent(sub);

    _onChangeSubscription(sub, () {
      apply();
      ...

      if (sub.weak) {
        weakDependents.remove(sub);
      } else {
        dependents?.remove(sub);
      }

      ...
    });
  }

void _onChangeSubscription(ProviderSubscription sub, void Function() apply) {
    ...
    final previousListenerCount = listenerCount;
    
    apply();
    

    ...

    if (listenerCount < previousListenerCount) {
      ...
      mayNeedDispose();
    } else if (listenerCount > previousListenerCount) {
      ...
    }
  }
```

`_onChangeSubscription` 先执行 `apply`：  
- 如果是弱引用，从 `weakDependents` 移除。  
- 如果是强引用，从 `dependents` 移除。  

执行完 `apply` 后，会比较执行前后的监听者数量。如果监听数量小于之前，就会走 `mayNeedDispose`。此时没有 `_keepAliveLinks` 且 `dependents` 为空，就会进入销毁流程。

如果符合销毁条件，会调用 `container.scheduler.scheduleProviderDispose(this)`。`ProviderScheduler` 负责管理 `container` 中保存状态的销毁。

```dart
void scheduleProviderDispose(ProviderElement element) {
    ...

    _stateToDispose.add(element);
    _scheduleTask();
  }
```

可以看到，会将需要销毁的 `Element` 放到 `_stateToDispose` 中，然后开启 `_scheduleTask`，最终调用 `ProviderScheduler` 的 `_task`：

```dart
void _task() {
    ...
    _performDispose();
    ...
    _stateToDispose.clear();
    ...
  }
```

```dart
void _performDispose() {
    for (var i = 0; i < _stateToDispose.length; i++) {
      final element = _stateToDispose[i];
      final links = element.ref?._keepAliveLinks;

      if ((links != null && links.isNotEmpty) ||
          element.container._disposed ||
          element.hasNonWeakListeners) {
        continue;
      }

      if (element.weakDependents.isEmpty) {
        element.container._disposeProvider(element.origin);
      } else {
        // Don't delete the pointer if there are some "weak" listeners active.
        element.clearState();
      }
    }
  }
```

`_performDispose` 主要做了几件事：

- 判断 `element` 是否处于被监听状态；如果是，`continue`。
- 判断是否有弱监听；如果没有，执行销毁操作；如果有，只清除状态，不销毁 `pointerElement` 实例。

```dart
void _disposeProvider(ProviderBase<Object?> provider) {
    final pointer = _pointerManager.remove(provider);
    // The provider is already disposed, so we don't need to do anything
    if (pointer == null) return;

    ...

    pointer.element?.dispose();
    pointer.element = null;
  }

$ProviderPointer? remove(ProviderBase<Object?> provider) {
    final directory = readDirectory(provider);
    if (directory == null) return null;

    final pointer = directory.pointers[provider];
    ...
    directory.pointers.remove(provider);
    ...
    return pointer;
  }
```

`_disposeProvider` 干了两件事：

- 从 `container` 中移除状态节点。
- `providerElement` 执行 `dispose`。

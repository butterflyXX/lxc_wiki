[← 返回状态管理目录](README.md)

# Riverpod 5 - notifier.state 状态变更与页面刷新

之前小节已经对 `read`/`watch` 展开说明了，这一小节，我们来看看状态变更如何实现页面的刷新。

## Demo 案例

祖传 demo：

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

## state 的 getter 和 setter

`state++` 是两步：

- get state
- set state

我们来看看 `set state`：

```dart
StateT get state {
  final ref = $ref;
  ref._throwIfInvalidUsage();

  return ref._element.readSelf().valueOrRawException;
}

set state(StateT newState) {
  final ref = $ref;
  ...
  ref._element.setValueFromState(newState);
}
```

可以看到不管是 get 还是 set，都是调用的 `ref._element`，所以本质，真正的状态还是保存在 `element` 中。

## setValueFromState 方法

```dart
void setValueFromState(ValueT state) => value = AsyncData(state);

set value(AsyncValue<ValueT> value) {
  if (kDebugMode) _debugDidSetState = true;

  final previousResult = this.value;
  final result = _value = value;

  if (_didBuild) {
    _notifyListeners(result, previousResult);
  }
}
```

最终调用到 `ProviderElement` 的 `_notifyListeners` 方法。

## _notifyListeners 方法

```dart
void _notifyListeners(
  AsyncValue<ValueT> newStateValue,
  AsyncValue<ValueT>? previousStateValue, {
  bool checkUpdateShouldNotify = true,
  bool isFirstBuild = false,
}) {
  // 新值
  final newState = resultForValue(newStateValue)!;

  // 旧值
  final previousStateResult =
      previousStateValue != null ? resultForValue(previousStateValue) : null;

  final previousState = previousStateResult?.value;

  // 判断是否要通知监听者
  if (checkUpdateShouldNotify) {
    switch ((previousStateResult, newState)) {
      case ((null, _)):
      case (($ResultError(), _)):
      case ((_, $ResultError())):
        break;
      case (($ResultData() && final prev, $ResultData() && final next)):
        if (!updateShouldNotify(prev.value, next.value)) return;
    }
  }

  // 监听者
  final listeners = [...weakDependents, if (!isFirstBuild) ...?dependents];
  switch (newState) {
    // 新值如果是正常值
    case final $ResultData<StateT> newState:
      for (var i = 0; i < listeners.length; i++) {
        // 订阅对象 - ProviderSubscription
        final listener = listeners[i];
        if (listener.closed) continue;

        container.runBinaryGuarded(
          // 回调函数
          listener.providerSub._notifyData,

          // 老值
          previousState,

          // 新值
          newState.value,
        );
      }
    // 新值如果是错误值
    case final $ResultError<StateT> newState:
      for (var i = 0; i < listeners.length; i++) {
        final listener = listeners[i];
        if (listener.closed) continue;

        container.runBinaryGuarded(
          listener.providerSub._notifyError,
          newState.error,
          newState.stackTrace,
        );
      }
  }
  ...
}

ProviderProviderSubscription<Object?> get providerSub {
  final that = impl;
  switch (that) {
    case final ProviderProviderSubscription<Object?> sub:
      return sub;
    case final ExternalProviderSubscription<Object?, Object?> sub:
      return sub._source;
  }
}
```

我们可以看到 `providerSub`，如果是 `ProviderProviderSubscription` 返回本身，如果是 `ExternalProviderSubscription`，返回 `sub._source` 其实就是上一节中说的真实的 `ProviderProviderSubscription` 订阅。

关于 `_notifyData` 上一节提到了，就是执行回调。

## 不同场景下的回调

在 UI 中 `ref.watch(testProvider)` 回调就是执行 `WidgetElement.markNeedsBuild`。

在 Provider 中 `ref.watch(testProvider)` 执行的是 `ProviderElement.invalidateSelf`：

```dart
void invalidateSelf({required bool asReload}) {
  if (asReload) _didChangeDependency = true;
  if (_mustRecomputeState) return;

  // 标记当前状态为需要重新创建的
  _mustRecomputeState = true;

  // 触发ref.onDispose
  runOnDispose();

  // autoDispose的Element是否需要销毁
  mayNeedDispose();

  // 将当前Element放入待刷新队列,等待刷新
  container.scheduler.scheduleProviderRefresh(this);

  // 将状态同步给上下游
  visitChildren((element) {
    element._markDependencyMayHaveChanged();
    element.visitListenables(
      (notifier) => notifier.notifyDependencyMayHaveChanged(),
    );
  });
  visitListenables((notifier) => notifier.notifyDependencyMayHaveChanged());
}
```

## 总结

简单一句话总结：如果在 `Notifier` 的 `build` 中监听另一个 `provider`，会导致 `Notifier` 的重新创建。

关于重新创建，支持两种：

- **被动重建**：`_mustRecomputeState`
- **主动重建**：`container.scheduler.scheduleProviderRefresh(this)`

都是异步的。

其实 `Provider` 中 `ref.watch(testProvider)`，也就是 `ProviderElement` 监听 `ProviderElement` 是很复杂的，作者为了良好的性能做了很多"懒处理"，能省则省。如果了解 `WidgetElement` 的刷新机制，这里就比较好理解了。

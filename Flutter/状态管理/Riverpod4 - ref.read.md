[← 返回状态管理目录](README.md)

# Riverpod 4 - ref.read 详解

上一小节主要是分析了 `watch` 都做了那些事情，这一小节，我们来进一步了解 `watch` 后，页面在什么时候进行刷新，还是使用上一节的案例：

## Demo 案例

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

之前已经了解到 `watch` 会将回调函数也就是组件的 `markNeedsBuild` 方法，绑定到 `ProviderElement`。

现在我们来看看 `ref.read(testProvider.notifier).add()` 都做了哪些事情。

## testProvider.notifier

```dart
Refreshable<NotifierT> get notifier {
  return ProviderElementProxy<NotifierT, StateT>(
    // provider
    this,

    // notifier
    (element) => (element as $ClassProviderElement<NotifierT, StateT, ValueT, CreatedT>).classListenable,
  );
}

ProviderElementProxy (class) with (_ProviderRefreshable)
_ProviderRefreshable (mixin) implements (Refreshable)
Refreshable implements (ProviderListenable)
```

`testProvider.notifier` 返回的是 `ProviderElementProxy`，通过源码我们了解到就是 `provider` 与 `notifier` 的一个包装对象，并且实现了 `ProviderListenable` 的一系列方法，猜想后续会将 `ProviderElementProxy` 当做 `provider` 来使用。

## ref.read 方法

### WidgetRef.read

```dart
@override
StateT read<StateT>(ProviderListenable<StateT> provider) {
  _assertNotDisposed();
  return ProviderScope.containerOf(this, listen: false).read(provider);
}
```

### ProviderContainer.read

```dart
StateT read<StateT>(ProviderListenable<StateT> provider) {
  final sub = listen(provider, (_, _) {});

  try {
    return sub.readSafe().valueOrProviderException;
  } finally {
    sub.close();
  }
}
```

我们可以看到 `read` 的参数是 `ProviderListenable` 类型，并且名称是 `provider`。

我们可以看到先创建了一个空回调的 `sub`，这一步并没有直接通过 `provider` 去查，是为了避免 `ProviderElement` 没有的情况，而 `listen` 会在没有的时候创建一个。

但是这里需要注意 `provider` 本质是 `ProviderElementProxy`，而 `listen` 中，最终创建 `ProviderSubscription` 是 `provider._addListener` 来完成的，所有我们来看看 `ProviderElementProxy._addListener` 都做了什么。

## ProviderElementProxy._addListener

```dart
ProviderSubscriptionImpl<OutT> _addListener(
  Node source,
  void Function(OutT? previous, OutT next) listener, {
  required void Function(Object error, StackTrace stackTrace) onError,
  required void Function()? onDependencyMayHaveChanged,
  required bool weak,
}) {
  // 创建/查找ProviderElement,上一节已经说过了,这里不展开说明
  final element = source.readProviderElement(provider);

  // 调用真正的provider._addListener,返回一个ProviderProviderSubscription,是个空回调
  final innerSub = provider._addListener(
    source,
    (prev, next) {},
    weak: weak,
    onDependencyMayHaveChanged: onDependencyMayHaveChanged,
    onError: (err, stack) {},
  );

  // 获取notifier,之前已经创建了
  final notifier = _lense(element);

  // 这里出现新的ProviderSubscription子类
  late ExternalProviderSubscription<InT, OutT> sub;
  final removeListener = notifier.addListener(
    (prev, next) => sub._notifyData(prev, next),
    onError: onError,
    onDependencyMayHaveChanged: onDependencyMayHaveChanged,
  );

  return sub = ExternalProviderSubscription<InT, OutT>.fromSub(
    innerSubscription: innerSub,
    onClose: removeListener,
    listener: listener,
    onError: onError,
    read: () {
      final element = source.readProviderElement(provider);
      element.flush();
      element.mayNeedDispose();

      return _lense(element).requireResult;
    },
  );
}
```

其实这里的 `notifier` 是 `element.classListenable`，类型是 `$Observable`，`$Observable` 继承自 `_ValueListenable`。

`_ValueListenable` 实际是管理 `notifier` 以及 `notifier` 的监听者的，`addListener` 是 `_ValueListenable` 的方法。

## notifier.addListener

```dart
void Function() addListener(
  void Function(ValueT?, ValueT) onChange, {
  required OnError? onError,
  required void Function()? onDependencyMayHaveChanged,
) {
  ...
  // onChange就是sub._notifyData
  final listener = _Listener._(onChange, onError, onDependencyMayHaveChanged);
  ...

  // 将监听保存到数组中
  _listeners[_count++] = listener;

  // 返回移除回调
  return () => _removeListener(listener);
}

// 监听者数量
var _count = 0;

// 监听者回调
List<_Listener<ValueT>?> _listeners = _emptyListeners();
```

## sub._notifyData

```dart
void _notifyData(OutT? prev, OutT next) {
  ...
  // 执行回调
  _listenedElement.container.runBinaryGuarded(_listener, prev, next);
}
```

`addListener` 就是将回调函数保存在了 `$Observable` 中，并且将移除的回调返回。

`ExternalProviderSubscription` 会将 `innerSub`/`removeListener` 传进去，还有个 `read` 回调，赋值给了 `ExternalProviderSubscription` 的 `_read`。

最终将 `ExternalProviderSubscription` 返回。

## readSafe 方法

接下来我们来看 `sub.readSafe().valueOrProviderException`：

```dart
$Result<StateT> readSafe() {
  ...
  final that = impl;
  that._listenedElement.mayNeedDispose();
  that._listenedElement.flush();

  return that._callRead();
}
```

上一节最后也提到了 `readSafe` 方法，是用来获取状态的，针对 `ref.read(testProvider.notifier)`，看看 `_callRead` 干了什么。

`readSafe` 是 `ProviderSubscription` 的一个扩展方法，这里的 `ProviderSubscription` 是 `ExternalProviderSubscription`：

```dart
$Result<OutT> _callRead() => _read();
```

调用的 `_read()`，上边提到过 `_read` 回调函数，实际是 `return _lense(element).requireResult`，`requireResult` 是 `$Observable` 一个 getter 方法，返回 `_result`，是个 `$Result` 类型，包装了真实的 `Notifier` 实例，所以 `sub.readSafe()` 返回了 `$Result`，`$Result.valueOrProviderException` 返回了 `Notifier` 实例：

```dart
ValueT get valueOrProviderException => value;
```

最后执行 `sub.close`，将订阅关闭，因为 `read` 只是用来获取数据，并不是真正创建监听，用完就销毁。

因为返回的是真实的 `Notifier` 实例，我们就可以调用 `Notifier` 的实例方法，比如 `add`/`subtract`。

## ref.read(testProvider) 的情况

以上介绍了 `ref.read(testProvider.notifier)`，如果是直接 `ref.read(testProvider)` 会是怎么样的呢？

```dart
StateT read<StateT>(ProviderListenable<StateT> provider) {
  // 获取订阅
  final sub = listen(provider, (_, _) {});

  try {
    // 获取状态
    return sub.readSafe().valueOrProviderException;
  } finally {
    sub.close();
  }
}
```

这里的 `provider` 是 `NotifierProvider`，而 `NotifierProvider._addListener` 上一节已经介绍过，返回的是 `ProviderProviderSubscription`。

`ProviderProviderSubscription.readSafe`，也会调用到 `_callRead`，但是 `ProviderProviderSubscription._callRead` 是 `_listenedElement.readSelf()`：

```dart
$Result<StateT> readSelf() {
  flush();

  final state = resultForValue(value);
  ...
  return state;
}

$Result<ValueT>? resultForValue(AsyncValue<ValueT> value) {
  switch (value) {
    case AsyncData():
      return $ResultData(value.value);
    case AsyncLoading(:final error?, :final stackTrace?) when value.retrying:
    case AsyncError(:final error, :final stackTrace):
      return $ResultError(error, stackTrace);
    case AsyncLoading():
      return null;
  }
}
```

真正的状态并不是 `value`，而是 `value.value`，在这个案例中就是一个 `int` 类型的值，`resultForValue` 返回 `$ResultData`，`$ResultData.valueOrProviderException` 又返回了 `int` value。

## 总结

总结：

- `ref.read(testProvider.notifier)` 返回的是 `notifier`
- `ref.read(testProvider)` 返回的是 `int` value

铺垫了这么多，可能会慢慢了解到作者的设计思路，`testProvider.notifier` 和 `testProvider` 都是 `provider`，只不过读出来的东西是不一样的。

日常使用中，常用的方式：

- `ref.read(testProvider.notifier)`
- `ref.read(testProvider)`
- `ref.watch(testProvider)`
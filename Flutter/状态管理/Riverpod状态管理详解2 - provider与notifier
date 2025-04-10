这篇主要说下provider与notifier的关系,用StateProvider作为切入点,来探索一下
有这样一个stateProvider
```dart
final counterStateProvider = StateProvider<int>((ref)=>0)
```
一般监听用`ref.watch(counterStateProvider)`,这个返回的直接就是int类型的数字
而我们想改变数据的时候用`ref.read(counterStateProvider.notifier).state++`
这里多了两个属性`notifier` 和 `state`,
下来看下`notifier`,它是StateProvider的属性,是`ProviderElementProxy`对象
```dart
ProviderElementProxy<T, StateController<T>> _notifier<T>(
  _StateProviderBase<T> that,
) {
  return ProviderElementProxy<T, StateController<T>>(
    that,
    (element) {
      return (element as StateProviderElement<T>)._controllerNotifier;
    },
  );
}

class ProviderElementProxy {
  const ProviderElementProxy(this._origin, this._lense);
}
```
通过查看源码发现能够read和watch的对象必须是`ProviderListenable`对象,ref不管是调用watch还是read,最终都会调用到`ProviderListenable`read方法,`ProviderElementProxy`就是混入`ProviderListenable`,那我们看下`ProviderElementProxy`的read方法是如何实现的
```dart
Output read(Node node) {
  final element = node.readProviderElement(_origin);
  ...
  return _lense(element).value;
}
```
这里可以看到read就是简单的获取provider对应的providerElement,然后通过回调函数,获取到`providerElement._controllerNotifier`,然后返回它的`value`属性,下面看下`StateProviderElement`的创建过程
```dart
@override
void create({required bool didChangeDependency}) {
  final provider = this.provider as _StateProviderBase<T>;
  final initialState = provider._create(this);

  final controller = StateController(initialState);
  _controllerNotifier.result = Result.data(controller);

  _removeListener = controller.addListener(
    fireImmediately: true,
    (state) {
      _stateNotifier.result = _controllerNotifier.result;
      setState(state);
    },
  );
}
```
会创建`StateController`对象,并且将`StateController`对象赋值给了`_controllerNotifier`,然后给`StateController`绑定上了`setState(state)`,也就是说`StateController`是有能力更新状态的,具体就是通过`StateController`的`state setter`进行更新的
```dart
RemoveListener addListener(
  Listener<T> listener, {
  bool fireImmediately = true,
}) {
  final listenerEntry = _ListenerEntry(listener);
  _listeners.add(listenerEntry);

  return () {
    if (listenerEntry.list != null) {
      listenerEntry.unlink();
    }
  };
}

set state(T value) {
  _state = value;

  /// only notify listeners when should
  if (!updateShouldNotify(previousState, value)) {
    return;
  }

  for (final listenerEntry in _listeners) {
    listenerEntry.listener(value);
  }
}
```
这里addListener对回调函数进行了一层包装`_ListenerEntry(listener)`,所以state中listenerEntry并不是回调函数,`listenerEntry.listener`才是
简单总结,对于StateProvider会多一个`_controllerNotifier`,`_controllerNotifier`又包含`StateController`,如果我们获取到了这个`StateController`对象,那么我们就能够通过它的state的setter方法修改状态了,而notifier就恰恰提供了获取`StateController`对象的能力
我们可以看到notifier的类名叫`ProviderElementProxy`就能够理解到notifier并不是真正意义上的Provider,而只是Provider来管理状态的代理,所以不管是`ref.watch(counterStateProvider)`还是`ref.watch(counterStateProvider.notifier).state`获取到的都是同一个状态
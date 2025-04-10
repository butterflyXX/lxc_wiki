`AutoDispose`就是会自动销毁的`provider`,这样说是不严谨的,应该是自动销毁的`providerElement`
之前文章说了实际管理状态是通过`providerElement`,这里通过`AutoDisposeStateProvider`看下内部的继承关系
```dart
class AutoDisposeStateProviderElement<T> extends StateProviderElement<T>
    with AutoDisposeProviderElementMixin<T>
    implements AutoDisposeStateProviderRef<T> {
class StateProviderElement<T> extends ProviderElementBase<T>
    implements StateProviderRef<T> {
abstract class ProviderElementBase<StateT> implements Ref<StateT>, Node {
```
可以看到`AutoDisposeStateProviderElement`最终也是继承自`ProviderElementBase`
这里要说的一个思路,如果要做到页面退出状态销毁,首先的一点是要从`ProviderContainer`移除,没错,就是第一篇开头说的顶层的状态存储容器,只有从这里删除掉,才能在之后`watch`的时候重新去创建新的,所以我们从`ProviderContainer`入手
状态都保存在`_stateReaders`属性中,看看`_stateReaders`什么时候`remove`,找到了`_disposeProvider`方法
```dart
void _disposeProvider(ProviderBase<Object?> provider) {
  final reader = _getOrNull(provider);
  // The provider is already disposed, so we don't need to do anything
  if (reader == null) return;

  reader._element?.dispose();

  if (reader.isDynamicallyCreated) {
    // Since the StateReader is implicitly created, we don't keep it
    // on provider dispose, to avoid memory leak

    void removeStateReaderFrom(ProviderContainer container) {
      /// Checking if the reader is the same instance is important,
      /// as it is possible that the provider was overridden.
      if (container._stateReaders[provider] == reader) {
        container._stateReaders.remove(provider);
      }
      container._children.forEach(removeStateReaderFrom);
    }

    removeStateReaderFrom(this);
  } else {
    reader._element = null;
  }
}
```
它是在`_performDispose`中调用的
```dart
void _performDispose() {
  for (var i = 0; i < _stateToDispose.length; i++) {
    final element = _stateToDispose[i];

    final links = element._keepAliveLinks;
    
    if (element.maintainState ||
        (links != null && links.isNotEmpty) ||
        element.hasListeners ||
        element._container._disposed) {
      continue;
    }
    element._container._disposeProvider(element._origin);
  }
}
```
这里遍历了`_stateToDispose`,说明要销毁的状态都在这里面,我们看下这个数组在什么时候执行的添加操作,定位到`scheduleProviderDispose`
```dart
void scheduleProviderDispose(
  AutoDisposeProviderElementMixin<Object?> element,
) {
  _stateToDispose.add(element);
  _scheduleTask();
}
```
`scheduleProviderDispose`在`mayNeedDispose`中被调用,这个方法是在`ProviderElementBase`中定义的,但是是个空函数,只有在`AutoDisposeProviderElementMixin`被重新实现了
```dart
void mayNeedDispose() {
  final links = _keepAliveLinks;

  if (!maintainState && !hasListeners && (links == null || links.isEmpty)) {
    _container.scheduler.scheduleProviderDispose(this);
  }
}
```
这里看到,在条件成熟的时候会执行`_container.scheduler.scheduleProviderDispose(this)`
这个时候我们在`mayNeedDispose`打断点,就会发现这样的调用栈
![image.png](https://upload-images.jianshu.io/upload_images/5976114-8b6c3804ea850159.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
熟悉的`ConsumerStatefulElement`和`unmount`方法,这个是在`widgetElement`从`Element tree`移除时调用的
```dart
void unmount() {
  /// Calling `super.unmount()` will call `dispose` on the state
  /// And [ListenManual] subscriptions should be closed after `dispose`
  super.unmount();

  for (final dependency in _dependencies.values) {
    dependency.close();
  }
}
```
`_dependencies`第一篇中说过,是当前`WidgetElement`依赖的`ProviderElement`,准确的说是`ProviderSubscription`,`ProviderSubscription`内部绑定了`ProviderElement`,`ProviderSubscription`的子类`_ProviderStateSubscription`实现的`close`方法
```dart
void close() {
  if (!closed) {
    listenedElement._dependents?.remove(this);
    listenedElement._onRemoveListener();
  }

  super.close();
}
```
`_onRemoveListener`
```dart
void _onRemoveListener() {
  _onRemoveListeners?.forEach(runGuarded);
  if (!hasListeners) {
    _didCancelOnce = true;
    _onCancelListeners?.forEach(runGuarded);
  }
  mayNeedDispose();
}
```
到这里,关键思路已经明确了,最后总结一下
`AutoDisposeProviderElement` 混入了 `AutoDisposeProviderElementMixin`,这个类中重写了`mayNeedDispose`方法
当`ConsumerElement`从树中移除时,会执行`AutoDisposeProviderElement`的`_onRemoveListener`方法,由于`AutoDisposeProviderElement`实现了`mayNeedDispose`,所以会判断是否将状态从顶层状态容器`_container`中移除,从而实现了状态自动的生命周期管理
这里在深入探究一下,如何判断是否要移除,销毁这个状态
```dart
void mayNeedDispose() {
  final links = _keepAliveLinks;

  // ignore: deprecated_member_use_from_same_package
  if (!maintainState && !hasListeners && (links == null || links.isEmpty)) {
    _container.scheduler.scheduleProviderDispose(this);
  }
}
```
`hasListeners`:
```dart
bool get hasListeners =>
    (_dependents?.isNotEmpty ?? false) || _providerDependents.isNotEmpty;
```
`_dependents`如果不为空说明有`WidgetElement`使用这个状态
`_providerDependents`如果不为空,说明有其他`ProviderElement`在依赖这个状态
只要有一个不为空,就认为当前的状态还在使用
`maintainState`: 这个被`_keepAliveLinks`替代了,默认就是`false`
`_keepAliveLinks`: 这个属性用来额外的控制状态的生命周期,比如
```dart
late KeepAliveLink alive;
final personStateProvider = AutoDisposeStateProvider<Person>((ref) {
  alive = ref.keepAlive();
  return Person("name", 0);
});
```
这个时候如果`watch`当前状态的组件销毁了,状态还是会保留的,我们看下`keepAlive`源码
```dart
KeepAliveLink keepAlive() {
  final links = _keepAliveLinks ??= [];

  late KeepAliveLink link;
  link = KeepAliveLink._(() {
    if (links.remove(link)) {
      if (links.isEmpty) mayNeedDispose();
    }
  });
  links.add(link);

  return link;
}
```
会将创建的`link`加入到`_keepAliveLinks`,会导致组件移除时调用`mayNeedDispose`,判断`_keepAliveLinks`并不会将状态销毁,这里其实是给用户一种手动管理状态生命周期的方式,比如A->B->C,我在C页面编辑一些内容,但是当我返回B的时候,依然要保持C的状态,等到再次进入C页面依然是之前退出时候的状态,但是当我返回A页面的时候,才真正的销毁C的状态,此时这个`link`就很有用了,可以做到进退自如的状态管理 同理处理
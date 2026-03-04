[← 返回状态管理目录](README.md)

上一篇组要介绍了Riverpod大体的实现原理,后面的小节我们进入具体的模块来剖析一下细节
这一篇我们来看看`Consumer`/`ConsumerWidget` 和 `ConsumerStatefulWidget`,Widget层的几个类
继承关系
ConsumerStatefulWidget (abstract)
    ↑
    └── ConsumerWidget (abstract)
            ↑
            └── Consumer<StateT> (final)

最后都是对`ConsumerStatefulWidget`的封装

在`ConsumerStatefulWidget`中自定义了`ConsumerStatefulElement`以及`ConsumerState`
我们常用的`ref`是在`ConsumerState`定义的
```dart
abstract class ConsumerState<WidgetT extends ConsumerStatefulWidget> extends State<WidgetT> {
  late final ref = context as WidgetRef;
}
```
可以看到`ref`本质是`context`,也就是说就是当前的`ConsumerStatefulElement`
接下来我们看看`ConsumerStatefulElement`
先看看它的属性列表
```dart
/// 顶层的状态仓库
late ProviderContainer container = ProviderScope.containerOf(this);

/// 依赖项当前这个组件依赖了哪些provider,key是provider,value是一个订阅,是什么后边再说,这里只要知道这里保存了自己以来的provider
var _dependencies = <ProviderListenable<Object?>, ProviderSubscription<Object?>>{};
Map<ProviderListenable<Object?>, ProviderSubscription<Object?>>? _oldDependencies;

/// listen的回调,我们在调用ref.listen的时候回调函数保存在这里,_listeners中的回调生命周期是ConsumerStatefulElement自己管理的后面会分析
final _listeners = <ProviderSubscription<Object?>>[];

/// 与上边_listeners不一样的是这个是手动管理的listen的回调,但是我们在后边会了解到,一些_manualListeners也是不需要我们手动管理生命周期的
List<ProviderSubscription<Object?>>? _manualListeners;
```
从`build`方法入手
```dart
@override
  Widget build() {

    ...

    try {
      _oldDependencies = _dependencies;
      for (var i = 0; i < _listeners.length; i++) {
        _listeners[i].close();
      }
      _listeners.clear();
      _dependencies = {};
      return super.build();
    } finally {
      for (final dep in _oldDependencies!.values) {
        dep.close();
      }
      _oldDependencies = null;
    }
  }
```
这里可以看到`_dependencies`和`_oldDependencies`的关系,每次`build`会将`_dependencies`赋值给`_oldDependencies`,然后`_dependencies`清空,在执行完真正的`super.build()`后再将`_oldDependencies`中依赖的`provider`移除
这个操作很自然想到应该是在`super.build()`执行过程所有的监听重新添加到了`_dependencies`中,并且从`_oldDependencies`中移除,最后将不再使用的`provider`关闭,这样一来,就能时刻保持状态依赖关系的同步,防止内存泄漏

这里还有一个`_listeners`大家注意到了吧,每次`build`都会清空,`super.build`调用后会重新加进来,同上,也是为了防止内存泄漏,上次的监听这次rebuild后可能不在需要了

这里先抛出一个问题为什么`_listeners`只有一个,并没有`_oldListeners`,而`_dependencies`对应会有一个`_oldDependencies`?

接下来我们看重头戏
```dart
@override
StateT watch<StateT>(ProviderListenable<StateT> target) {
    return _dependencies
            .putIfAbsent(target, () {
              final oldDependency = _oldDependencies?.remove(target);

              if (oldDependency != null) {
                return oldDependency;
              }

              final sub = container.listen<StateT>(target, (_, _) => markNeedsBuild());
              return sub;
            })
            .readSafe()
        as StateT;
  }
```
这里看到每次`watch`,会在`_dependencies`中加入`provider`和回调函数(如果`_dependencies`中没有这个`provider`的情况下),这个回调函数印证了我们上边说的,所有的监听重新添加到了`_dependencies`中,并且从`_oldDependencies`中移除
如果`_oldDependencies`中没有,说明是新的需要重新建立监听,这里我们看到了熟悉的`markNeedsBuild`

再看下`listen`具体实现
```dart
@override
  void listen<StateT>(
    ProviderListenable<StateT> provider,
    void Function(StateT? previous, StateT value) listener, {
    void Function(Object error, StackTrace stackTrace)? onError,
  }) {
    final sub = container.listen<StateT>(provider, listener, onError: onError);
    _listeners.add(sub);
  }
```
和`watch`有些类似,只不过这里的回调函数不是`markNeedsBuild`而是传入的`listener`

然后是`listenManual`
```dart
@override
  ProviderSubscription<ValueT> listenManual<ValueT>(
    ProviderListenable<ValueT> provider,
    void Function(ValueT? previous, ValueT next) listener, {
    void Function(Object error, StackTrace stackTrace)? onError,
    bool fireImmediately = false,
  }) {
    final listeners = _manualListeners ??= [];

    final container = ProviderScope.containerOf(this, listen: false);

    final sub = container.listen<ValueT>(provider, listener, onError: onError, fireImmediately: fireImmediately);

    // 监听close回调,从_manualListeners移除
    final previousOnClose = sub.impl.onClose;
    sub.impl.onClose = () {
      previousOnClose?.call();
      _manualListeners?.remove(sub);
    };

    listeners.add(sub);

    return sub;
  }
```
`listenManual`返回值是个`ProviderSubscription`,也就是说你可以手动来进行关闭监听,关闭之后onClose回到回来后从`_manualListeners`将监听移除掉,但是感觉这一步多余,因为看代码知道`ProviderSubscription.close`会判断当前回调是否已经关闭,不会报错崩溃
```dart
void close() {
    if (_closed) return;

    onClose?.call();
    _listenedElement.removeDependentSubscription(this, () {
      _closed = true;
    });
  }
```


下面我们来看`unmount`
```dart
@override
  void unmount() {
    super.unmount();

    for (final dependency in _dependencies.values) {
      dependency.close();
    }
    for (var i = 0; i < _listeners.length; i++) {
      _listeners[i].close();
    }
    final manualListeners = _manualListeners?.toList();
    if (manualListeners != null) {
      for (final listener in manualListeners) {
        listener.close();
      }
      _manualListeners = null;
    }
  }
```
从组件树移除的时候,会清空`_dependencies`,`_listeners`,`_manualListeners`,上边说了`_manualListeners`算是一个需要我们手动关系生命周期的队列,但是这里作者也帮我们清理掉了,和上边的`build`一起分析,我们能get到一个使用规范: `_listeners`是在build中使用的,而`_manualListeners`是在build外部使用的,比如在initstate中


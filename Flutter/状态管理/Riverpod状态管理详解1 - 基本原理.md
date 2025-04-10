```Riverpod```数据共享也是使用了```InheritedWidget```,在项目中,runapp外层要嵌套一个```ProviderScope```
```
void main() {
  runApp(
    ProviderScope(
      child: MyApp(),
    ),
  );
}
```
而```ProviderScope```是个```StatefulWidget```,在State build方法中,使用了```UncontrolledProviderScope```
![image.png](https://upload-images.jianshu.io/upload_images/5976114-3e79f3e2b6ec1fb1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)```
UncontrolledProviderScope```就是一个```InheritedWidget```
![image.png](https://upload-images.jianshu.io/upload_images/5976114-6494ff2f82543f21.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
在我们使用```riverpod```时候,获取的状态都保存在这个```UncontrolledProviderScope```中,具体是```ProviderContainer```对象当中,这里能够感受到的一个最大的优势就是不再依赖上下文,因为在任何地方获取的都是这个顶层的状态,像之前如果路由进行了跳转,路由1中的状态是没办法在路由2中获取到的,当然有方法去解决,但是会带来更多的代码和结构上的不合理.

![image.png](https://upload-images.jianshu.io/upload_images/5976114-34cdf106839f9e45.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```riverpod```提供的是顶层的```InheritedWidget```来管理所有的状态,很容易想到```ProviderContainer```中应该是有个Map来存储所有的状态,所有状态保存在顶层,下层组件想要使用状态,通过顶层的InheritedWidget获取```ProviderContainer```来使用
```riverpod```里边一个很特别的对象就是```ref```,在```provider```(指的riverpod中的状态)中有```ref```,在```Consumer```中也有一个```ref```,但是这两个```ref```是不一样的类型,这里也要说一下```riverpod```中的```provider```会有对应的```element```,所以可以简单的理解
```provider``` 中的```ref```是```providerElement```
```Consumer``` 中的```ref```是```widgetElement```,也就是说```context```可以被强制转化成```ref```
关于```providerElement```可以类比widget与element的关系,provider只是一个配置类,providerElement才是真实的状态管理类,只有providerElement创建才会产生真正的状态,所以在```Riverpod```中经常会看见类似这样的代码
```
final counterStateProvider = StateProvider<int>((ref)=>0)
```
```StateProvider```可能会被提前创建出来,但是真正的状态被创建,会在真正使用的时候
**所以不必担心全局创建的provider**
这里还是要做一个类比
```InheritedElement```与```WidgetElement```的关系与```ProviderElement```与```ConsumerStatefulElement```
```InheritedElement```中有```_dependents```,```WidgetElement```中有```_dependencies```
```ProviderElement```中有```_dependents```,```ConsumerStatefulElement```中有```_dependencies```
他们的含义是一样的```_dependents```代表哪些组件注册了刷新回调,```_dependencies```代表组件在哪些状态中注册了回调,有点绕,但是要理解
上边说了在顶层有一个保存所有状态的```InheritedWidget```,但是具体如何操作组件刷新的呢?
类比```Provider```状态管理,```Riverpod```也提供了两个方法```read``` 和```watch```,含义是一样的```read```是用来读数据,```watch```不仅用来读也用来注册刷新回调
在```ConsumerWidget```中使用的```ref```我们上面说到是```widget```的```element```,我们执行```ref.watch(countProvider)```的时候
```
@override
Res watch<Res>(ProviderListenable<Res> target) {
  _assertNotDisposed();
  return _dependencies.putIfAbsent(target, () {
    final oldDependency = _oldDependencies?.remove(target);

    if (oldDependency != null) {
      return oldDependency;
    }

    return _container.listen<Res>(
      target,
      (_, __) => markNeedsBuild(),
    );
  }).read() as Res;
}
```
```_container```就是顶层的```ProviderContainer```
```_dependencies```用来存放```_container```监听之后的```ProviderSubscription```,类比```StreamSubscription```,如果当前的```element```从element tree移除后,可以移除掉在```_container```中注册的刷新回调
```
@override
ProviderSubscription<State> listen<State>(
  ProviderListenable<State> provider,
  void Function(State? previous, State next) listener, {
  bool fireImmediately = false,
  void Function(Object error, StackTrace stackTrace)? onError,
}) {
  return provider.addListener(
    this,
    listener,
    fireImmediately: fireImmediately,
    onError: onError,
    onDependencyMayHaveChanged: null,
  );
}
```
最终走到```Provider```的```addListener``` 方法 ```node.readProviderElement(this) ```本质就是找到```ProviderElement```,这里其实也发现了```ProviderElement```与```WidgetElement```的一一对应的关系是```_ProviderStateSubscription```来管理的
```
_ProviderStateSubscription(
  super.source, {
  required this.listenedElement,
  required this.listener,
  required this.onError,
}) {
  final dependents = listenedElement._dependents ??= [];
  dependents.add(this);
}
```
构造函数可以看到```_dependents```添加了```Subscription```,也就是添加了组件的刷新回调,```provider```中的状态变化的时候可以遍历```_dependents```来刷新UI组件,实际也是这样做的
```
void _notifyListeners(
    Result<StateT> newState,
    Result<StateT>? previousStateResult, {
    bool checkUpdateShouldNotify = true,
  }) {
  
  ...
  
    final listeners = _dependents?.toList(growable: false);
    newState.map(
      data: (newState) {
        if (listeners != null) {
          for (var i = 0; i < listeners.length; i++) {
            final listener = listeners[i];
            if (listener is _ProviderStateSubscription) {
              Zone.current.runBinaryGuarded(
                listener.listener,
                previousState,
                newState.state,
              );
            }
          }
        }
      },
      error: (newState) {
        if (listeners != null) {
          for (var i = 0; i < listeners.length; i++) {
            final listener = listeners[i];
            if (listener is _ProviderStateSubscription<StateT>) {
              Zone.current.runBinaryGuarded(
                listener.onError,
                newState.error,
                newState.stackTrace,
              );
            }
          }
        }
      },
    );

    ...
  }
```
用```StateProvider```举例,在我们调用```ref.read(countProvider.notifier).state++``` 的时候会执行```listenerEntry.listener(value) ```
```
set state(T value) {
  assert(_debugIsMounted(), '');
  final previousState = _state;
  _state = value;

  final errors = <Object>[];
  final stackTraces = <StackTrace?>[];
  for (final listenerEntry in _listeners) {
    try {
      listenerEntry.listener(value);
    } catch (error, stackTrace) {
      errors.add(error);
      stackTraces.add(stackTrace);

      if (onError != null) {
        onError!(error, stackTrace);
      } else {
        Zone.current.handleUncaughtError(error, stackTrace);
      }
    }
  }
  if (errors.isNotEmpty) {
    throw StateNotifierListenerError._(errors, stackTraces, this);
  }
}
```
最终会调用到```ProviderElement```的```setState```,然后调用```_notifyListeners```
```
void setState(StateT newState) {
  assert(
    () {
      _debugDidSetState = true;
      return true;
    }(),
    '',
  );
  final previousResult = getState();
  final result = _state = ResultData(newState);

  if (_didBuild) {
    _notifyListeners(result, previousResult);
  }
}
```
而```setState```是在```ProviderElement```创建的时候就进行了注册
```
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
```Riverpod```实际做的对系统```InheritedWidget```的优化,让状态从组件数中抽离出来,
```InheritedWidget```时代,状态和组件本身就是一个东西(```InheritedElement```就是个特殊的```WidgetElement```),自己管理自己的状态,也就导致他严重依赖组件树的结构,如果两个处于不同分支下的状态想要相互调用是不可以的,但是```Riverpod```对他们进行了拆分,状态单独出来,并且通过索引保存在最顶层,这样既保持了状态和组件的1对1的关系,又实现了不同分支下的状态之间的相互调用

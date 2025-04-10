Riverpod 数据共享也是使用了InheritedWidget,在项目中,runapp外层要嵌套一个ProviderScope
void main() {
  runApp(
    ProviderScope(
      child: MyApp(),
    ),
  );
}
而ProviderScope是个StatefulWidget,在State build方法中,使用了UncontrolledProviderScope
[图片]
UncontrolledProviderScope就是一个InheritedWidget
[图片]
在我们使用riverpod时候,其实获取的provider都保存在这个UncontrolledProviderScope中,也就是ProviderContainer对象当中
最外层了共享架构先说到这里,这里能够感受到的一个最大的优势就是不再依赖上下文,因为在任何地方获取的都是这个顶层的状态,像之前如果路由进行了跳转,路由1中的状态是没办法在路由2中获取到的,当然有方法去解决,但是会带来更多的代码和结构上的不合理,riverpod的解决方法和getx基本类似,只不过getx是提供了单例类来管理所有的状态,而riverpod提供的是顶层的InheritedWidget来管理所有的状态,很容易想到ProviderContainer中应该是有个Map来存储所有的状态,所有状态保存在顶层,下层组件想要使用状态,通过顶层的InheritedWidget获取ProviderContainer来使用
暂时无法在昆仑万维文档外展示此内容
riverpod里边一个很特别的对象就是 ref ,在provider中也有ref,在Consumer中也有一个ref,但是这两个ref是不一样的类型,这里也要说一下riverpod中的provider会有对应的element,所以可以简单的理解
provider 中的 ref 是 providerElement
Consumer 中的 ref 是 widgetElement(由于ConsumerWidget的element是自定义的,所以不能将普通的element强转成ref)
也可以类比widget与element的关系,provider只是一个配置类,providerElement才是真实的状态管理类,只有providerElement创建才会产生真正的状态
所以不必担心全局创建的provider

这里还是要做一个类比
InheritedElement与WidgetElement的关系与ProviderElement与ConsumerStatefulElement
InheritedElement中有_dependents,WidgetElement中有_dependencies
ProviderElement中有_dependents,ConsumerStatefulElement中有_dependencies
他们的含义是一样的_dependents代表哪些组件注册了刷新回调,_dependencies代表组件在哪些状态中注册了回调,有点绕,但是要理解
上边说了在顶层有一个保存所有状态的InheritedWidget,但是具体如何操作组件刷新的呢?
类比Provider状态管理,Riverpod也提供了两个方法 read 和 watch,含义是一样的read是用来读数据,watch不仅用来读也用来注册刷新回调
在ConsumerWidget中使用的ref我们上面说到是widget的element,我们执行ref.watch(countProvider)的时候
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
_container就是顶层的ProviderContainer
_dependencies用来存放_container监听之后的ProviderSubscription,类比StreamSubscription,如果当前的element从element tree移除后,可以移除掉在_container中注册的刷新回调
/// ProviderContainer
@override
ProviderSubscription<State> listen<State>(
  ProviderListenable<State> provider,
  void Function(State? previous, State next) listener, {
  bool fireImmediately = false,
  void Function(Object error, StackTrace stackTrace)? onError,
}) {
  // TODO test always flushed provider
  return provider.addListener(
    this,
    listener,
    fireImmediately: fireImmediately,
    onError: onError,
    onDependencyMayHaveChanged: null,
  );
}
/// Provider
@override
ProviderSubscription<StateT> addListener(
  Node node,
  void Function(StateT? previous, StateT next) listener, {
  required void Function(Object error, StackTrace stackTrace)? onError,
  required void Function()? onDependencyMayHaveChanged,
  required bool fireImmediately,
}) {

  final element = node.readProviderElement(this);
  ...
  return _ProviderStateSubscription<StateT>(
    node, //ProviderContainer
    listenedElement: element, // ProviderElement
    listener: (prev, next) => listener(prev as StateT?, next as StateT), // 组件的刷新回调
    onError: onError,
  );
}
最终走到Provider 的 addListener 方法 node.readProviderElement(this) 本质就是找到ProviderElement,这里其实也发现了ProviderElement与WidgetElement的一一对应的关系是_ProviderStateSubscription来管理的
_ProviderStateSubscription(
  super.source, {
  required this.listenedElement,
  required this.listener,
  required this.onError,
}) {
  final dependents = listenedElement._dependents ??= [];
  dependents.add(this);
}
构造函数可以看到_dependents添加了Subscription,也就是添加了组件的刷新回调,provider中的状态变化的时候可以遍历_dependents来刷新UI组件,实际也是这样做的
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
用StateProvider举例,在我们调用ref.read(countProvider.notifier).state++ 的时候会执行listenerEntry.listener(value) 
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
最终会调用到ProviderElement的setState,然后调用_notifyListeners
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
而setState是在ProviderElement创建的时候就进行了注册
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
riverpod实际做的对系统InheritedWidget的优化,让状态从组件数中抽离出来,但是实现思路还是沿用了InheritedWidget的大部分思想
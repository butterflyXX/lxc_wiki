在使用riverpod时候,如果我们的state是个对象,比如`Person`,有两个属性`name`和`age`,我们只修改了`age`,如果只使用`ref.watch(personStateProvider)`,会造成仅仅使用`name`的组件也被刷新
```dart
final personStateProvider = StateProvider<Person>((ref) {
  return Person("name", 0);
});

class Person {
  String name;
  int age;

  Person(this.name, this.age);

  copy({String? name, int? age}) {
    return Person(name ?? this.name, age ?? this.age);
  }
}
```
这个时候一般使用`ref.watch(personStateProvider.select((p) => p.name))`来进一步降低刷新颗粒度
上文说过,放入`watch`中的对象必须是`ProviderListenable`子类
`select`是在`AlwaysAliveProviderBase`中定义的这个类被混入到`StateProvider`中
```dart
class StateProvider<T> extends _StateProviderBase<T>
    with AlwaysAliveProviderBase<T>
    
mixin AlwaysAliveProviderBase<State> on ProviderBase<State>
    implements
        AlwaysAliveProviderListenable<State>,
        AlwaysAliveRefreshable<State> {
  @override
  AlwaysAliveProviderListenable<Selected> select<Selected>(
    Selected Function(State value) selector,
  ) {
    return _AlwaysAliveProviderSelector<State, Selected>(
      provider: this,
      selector: selector,
    );
  }
}
```
这里`select`返回了`_AlwaysAliveProviderSelector`对象
```dart
class _AlwaysAliveProviderSelector extends _ProviderSelector
```
`_AlwaysAliveProviderSelector`继承自`_ProviderSelector`,在`_ProviderSelector`中实现了`addListener`方法
```dart
_SelectorSubscription<Input, Output> addListener(
  Node node,
  void Function(Output? previous, Output next) listener, {
  required void Function(Object error, StackTrace stackTrace)? onError,
  required void Function()? onDependencyMayHaveChanged,
  required bool fireImmediately,
}) {
  onError ??= Zone.current.handleUncaughtError;

  late Result<Output> lastSelectedValue;

  final sub = node.listen<Input>(
    provider,
    (prev, input) {
      _selectOnChange(
        newState: input,
        lastSelectedValue: lastSelectedValue,
        listener: listener,
        onError: onError!,
        onChange: (newState) => lastSelectedValue = newState,
      );
    },
    onError: onError,
  );

  lastSelectedValue = _select(Result.guard(sub.read));

  return _SelectorSubscription(
    node,
    sub,
    () {
      return lastSelectedValue.map(
        data: (data) => data.state,
        error: (error) => throwErrorWithCombinedStackTrace(
          error.error,
          error.stackTrace,
        ),
      );
    },
  );
}
```
这个方法中主要有两部分组成
- 到`stateProvider`中注册回调:在`state`改变之后执行`_selectOnChange`,这个时候只要是`person`类变了,就会回调,但是`listener`(`widget setState`)会不会执行,就要看`_selectOnChange`的处理逻辑了
- 对`stateProvider`中注册回调的封装,`_SelectorSubscription`,这个对象最后返回到了`widget element`中,保存在了`_dependencies`中
下面我们来看下`_selectOnChange`是如何处理的
```dart
void _selectOnChange({
  required Input newState,
  required Result<Output> lastSelectedValue,
  required void Function(Object error, StackTrace stackTrace) onError,
  required void Function(Output? prev, Output next) listener,
  required void Function(Result<Output> newState) onChange,
}) {

  // 这里使用select的回调函数,获取到具体要监听的值,name或者是age,并且把他包装成Result对象
  final newSelectedValue = _select(Result.data(newState));
  
  // 判断旧值和新值是否相同,这里比较的就是具体的name或者age了
  if (!lastSelectedValue.hasState ||
      !newSelectedValue.hasState ||
      lastSelectedValue.requireState != newSelectedValue.requireState) {

    // 不同,就会进行刷新,listener就是从widget element层传进来的 markNeedsBuild
    onChange(newSelectedValue);
    newSelectedValue.map(
      data: (data) {
        listener(
          lastSelectedValue.stateOrNull,
          data.state,
        );
      },
      error: (error) => onError(error.error, error.stackTrace),
    );
  }
}
```
最后我们来看下完整的状态变化流程
```dart
Consumer(builder: (context, ref, child) {
  print("name");
  final name = ref.watch(personStateProvider.select((p) => p.name));
  return Text("name: $name");
}),
Consumer(builder: (context, ref, child) {
  print("age");
  final age = ref.watch(personStateProvider.select((p) => p.age.toString()));
  return Text("age: $age");
}),

final b = ref.read(personStateProvider.notifier);
b.state = b.state.copy(age: b.state.age + 1);
```
1.`state`被重新赋值,对`age`进行了+1,`name`没有变化
2.由于`state`变化,导致`personProviderElement`执行`setState`,进而执行`_notifyListeners`
```dart
_notifyListeners(){
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
  );
}
```
3.`listener.listener`执行的是`_selectOnChange`
4.`_selectOnChange`通过`select((p) => p.age.toString())`,获取到新的`age`,并且包装成`Result`
```dart
final newSelectedValue = _select(Result.data(newState));
```
5.对新旧值进行比较,如果不同,执行`listener`,这里的`listener`就是`markNeedsBuild`
```dart
listener(
  // TODO test from error
  lastSelectedValue.stateOrNull,
  data.state,
);
```
`select`本质是`providerElement`与`WidgetElement`中间加了一层,`providerElement._dependents`并不是直接添加的`WidgetElement.markNeedsBuild`,而是`_AlwaysAliveProviderSelector._selectOnChange`,在`_AlwaysAliveProviderSelector._selectOnChange`中判断是否要执行`WidgetElement.markNeedsBuild`
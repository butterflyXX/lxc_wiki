当我们的状态是一个`int`类型时候,我们使用`StateProvider`,是非常好用的
```dart
final counter = StateProvider((ref)=>0);
refRead(counter.notifier).state++;
```
但是,如果我们的状态是个对象,或者要进行很复杂的逻辑处理`StateProvider`就不是很友好了,因为他只提供了`state`这个属性来处理状态,所有的逻辑处理都需要我们自己实现,然后知道最终的状态,然后才能对`state`进行赋值,举个例子,如果我们状态是一个`Person`
```dart
class Person {
  String name;
  int age;

  Person(this.name, this.age);
}
```
我们想要修改名字,如果用`StateProvider`我们只能这样处理,有种面向过程开发的感觉
```dart
final person = StateProvider((ref)=>Person("",0));
refRead(person.notifier).state = Person("123", 0);
```
能不能只改变名字的时候,不去直接操作`state`,提供个方法啥的,比如
```dart
refRead(person.notifier).changeName("123");
```
###### 这篇文章就是解决这种问题的方案 - `NotifierProvider`
记得之前讲过`StateProvider`内部的实现逻辑,有`notifier`它的`value`属性是个`StateController`对象,`refRead(counter.notifier)`本质返回的就是这个对象,它提供了`state`的`setter`和`getter`,其实`NotifierProvider`本质也是这样的,只不过`StateProvider`的`notifier`是它内部实现的,`NotifierProvider`的`notifier`是我们外部定义的,所以我们通过`StateProvider`来学习`NotifierProviderImpl`
往下看的前提,先要看下 **Riverpod状态管理详解2**
`NotifierProvider`只是一个别名,本质是`NotifierProviderImpl`
对比一下`NotifierProviderImpl`和`StateProvider`的`_notifier`属性,基本是一样的
```dart
// StateProvider
ProviderElementProxy<T, StateController<T>> _notifier<T>(_StateProviderBase<T> that) {
  return ProviderElementProxy<T, StateController<T>>(
    that,
    (element) {
      return (element as StateProviderElement<T>)._controllerNotifier;
    },
  );
}

// NotifierProviderImpl:
ProviderElementProxy<T, NotifierT> _notifier<NotifierT extends NotifierBase<T>, T>(NotifierProviderBase<NotifierT, T> that) {
  return ProviderElementProxy<T, NotifierT>(
    that,
    (element) {
      return (element as NotifierProviderElement<NotifierT, T>)._notifierNotifier;
    },
  );
}
```
最终,通过`read`调用获取到`ProxyElementValueNotifier`的 `value`属性,也就是`StateController`或者`NotifierT`
```dart
Output read(Node node) {
  final element = node.readProviderElement(_origin);
  element.flush();
  element.mayNeedDispose();
  return _lense(element).value;
}
```
调用原理都是一样的,区别就在于`element`创建的时候`StateProvider`创建了`StateController`,而`NotifierProviderImpl`创建了`NotifierT`,我们看下`create`方法
```dart
// StateProvider:
void create() {
  final provider = this.provider as _StateProviderBase<T>;
  final initialState = provider._create(this);
  final controller = StateController(initialState);
  _controllerNotifier.result = Result.data(controller);
  ...
}

// NotifierProviderImpl:
void create() {
  final provider = this.provider as NotifierProviderBase<NotifierT, T>;
  final notifierResult = (_notifierNotifier.result ??= provider._createNotifier().._setElement(this));
  final notifier = notifierResult.requireState;
  setState(provider.runNotifierBuild(notifier));
}
```
到此内部的调用原理就清楚了,下面重点来看下`Notifier`我们是怎么在外部创建的,上面看到调用了`provider`的`_createNotifier`,是从外部传入的
```dart
final NotifierT Function() _createNotifier;

NotifierProviderImpl(
  super._createNotifier, {
  super.name,
  super.dependencies,
}) : super(
        allTransitiveDependencies:
            computeAllTransitiveDependencies(dependencies),
      );
```
`NotifierT` 又必须要继承`Notifier`,所以我们的`notifier`可以这样定义
```dart
class PersonNotifier extends Notifier<Person> {
  @override
  Person build() {
    return Person("", 0);
  }
}
```
必须要实现`build`,这个也可以从`create`方法中看到,是为了创建初始状态
```dart
T runNotifierBuild(NotifierBase<T> notifier) {
  return (notifier as Notifier<T>).build();
}
```
这样我们的`PersonNotifier`就创建好了
```dart
final person = NotifierProviderImpl<PersonNotifier, Person>(()=>PersonNotifier());
```
我们可以在`PersonNotifier`中处理复杂的逻辑
比如我们上面说的只修改名字,我们可以在`PersonNotifier`中添加方法`changeName`
```dart
class PersonNotifier extends Notifier<Person> {
  @override
  Person build() {
    return Person("", 0);
  }

  changeName(String newName) {
    state = state.copy(name: newName);
  }
}
```
外部直接调用`changeName`就可以了
```dart
refRead(person.notifier).changeName("123");
```
这一小节可以总结一句话:
**`NotifierProvider`和`StateProvider`是一样的,只不过是将管理状态的能力放权给了调用者,用来处理更加复杂的逻辑操作**
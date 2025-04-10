在使用Riverpod时候,我们常常会对比`Widget`来说,因为他们很像都有自己的`Element`,但是从今天要讲的内容来说,他们有一个很重要的不同点是:
**一个`Widget`可以对应多个`WidgetElement`,而一个`provider`只能对应一个`ProviderElement`**
`Provider`和他的`Element`是一对一的关系,这里说的不严谨,有`Provider`不一定要有`Element`,但是一个`Provider`只能对应一个`Element`,理解这一点很重要,这里我们先从代码角度看下为什么会出现这种1v1的关系
我们说到所有的状态都保存在顶层的`ProviderContainer`,看下存储的数据结构
```dart
final Map<ProviderBase<Object?>, _StateReader> _stateReaders;
```
从表面看`ProviderBase`就是`Provider`,`_StateReader`可以简单理解成`ProviderElement`,所以`Provider`实例作为key,就不会有多个`ProviderElement`被创建
但是很多场景下,我们的`Provider`处理这种1vN的情况,比如我们从商品详情1,跳转到了商品详情2,有跳转到了商品详情3,每一个商品提供一个`Provider`实例,从代码角度是可行的,在跳转页面的时候创建`Provider`
```dart
class ProductPage extends ConsumerWidget {
  final int id;
  ProductPage({required this.id,super.key});
  late final productProvider = AutoDisposeStateProvider((_)=>Product(id));
  ...
}
```
这种方式很好的解决了上面的问题,`Provider`的创建是在商品页面进入的时候,一般这种页面状态生命周期都要和页面同步,所以使用`AutoDisposeStateProvider`
`ProviderFamily`本质就是可以创建多个`Provider`
```dart
final userProviderFamily = AutoDisposeStateProvider.family<Person, int>((ref, userId) {
  return Person(userId);
});
final test = userProviderFamily(id);
```
这样可以如果id不同会创建新的`Provider`
```dart
static const family = AutoDisposeStateProviderFamily.new;
```
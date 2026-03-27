[← 返回状态管理目录](README.md)

# Riverpod 7 - Family

之前的小节我们了解了 `Provider` 的实现原理，这一小节我们来看看 `Family`。

在 `Riverpod` 中，`Family` 是一类 `Provider` 的集合。下面从一个简单的 demo 入手。

## Demo

```dart
final testProvider = NotifierProvider.family<TestNotifier, int, int>(TestNotifier.new);

class TestNotifier extends Notifier<int> {
  int arg;
  TestNotifier(this.arg);
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

class MeetingNotesListPage extends ConsumerWidget {
  const MeetingNotesListPage({super.key});
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    return Scaffold(
      appBar: AppBar(title: Text('Test')),
      body: ListView.builder(
        itemBuilder: (context, index) {
          return ListTile(
            title: Text(ref.watch(testProvider(index)).toString()),
            onTap: () {
              ref.read(testProvider(index).notifier).add();
            },
          );
        },
        itemCount: 10,
      ),
    );
  }
}
```

通过这个 demo 可以了解到 `Family` 的一个重要作用：对于「结构相同、仅参数不同」的状态，可以通过 `Family` 创建很多个独立实例。

## 实用场景

我们开发一个购物 app：从苹果详情页跳到香蕉详情页，两个页面状态类型完全一样，但具体状态不同。如果只用单个 `provider`，就得定义 `appleProvider`、`bananaProvider`；商品一多，手动建 `provider` 不现实。

这时 `Family` 就派上用场：定义一类 `Provider`，用 id 区分不同实例。例如苹果 id 为 1、香蕉 id 为 2，则 `testProvider(1)` 与 `testProvider(2)` 各管各的状态。

## 实现思路概览

弄清 `Family` 的作用之后，我们从源码看它如何实现。

之前小节里，`testProvider` 是一个 `Provider` 实例，全局共用这一个实例。本小节里，`testProvider` 不再是单个 `Provider` 实例，而是 `Family` 实例，同样也是全局共用这一个 `Family` 实例。

在 `ProviderContainer` 中，`final HashMap<ProviderBase<Object?>, $ProviderPointer> pointers` 保存真实的状态实例，key 是 `ProviderBase`（即具体的 `Provider` 实例）。Map 取 value 时靠 key 的 `hashCode` 与 `==`。我们来看看 `Provider` 如何实现这两个方法。

## LegacyProviderMixin：hashCode 与 ==

所有的 `Provider` 都会混入 `LegacyProviderMixin`：

```dart
base mixin LegacyProviderMixin<StateT> on $ProviderBaseImpl<StateT> {
  @override
  int get hashCode {
    if (from == null) return super.hashCode;

    return from.hashCode ^ argument.hashCode;
  }

  @override
  bool operator ==(Object other) {
    if (from == null) return identical(other, this);

    return other.runtimeType == runtimeType &&
      other is $ProviderBaseImpl<StateT> &&
      other.from == from &&
      other.argument == argument;
  }
}
```

可以看到，两个方法都通过 `from` 做了区分：

- 普通 `Provider` 实例：`from == null`，按普通对象处理。
- 属于某个 `Family` 的 `Provider` 实例：不再关心「是不是同一个 Provider 对象」，而是看是否属于同一个 **family**（`other.from == from`），以及 **`argument`（也就是 id）是否相同**。

因此 `testProvider(1)` 与 `testProvider(2)` 类型相同，但在 `pointers` 里对应两个 key，各自有独立的 `ProviderElement`。

## `NotifierProvider.family` 做了什么

下面分析 `final testProvider = NotifierProvider.family<TestNotifier, int, int>(TestNotifier.new)`。

这里有个语法糖：`TestNotifier.new` 指向 `TestNotifier` 的构造函数，相当于 `(arg) => TestNotifier(arg)`。  
翻译成更展开的写法可以理解为：

`final testProvider = NotifierProvider.family<TestNotifier, int, int>.call((arg) => TestNotifier(arg));`

`NotifierProvider.family` 是一个 `NotifierProviderFamilyBuilder` 实例；`call` 返回 `NotifierProviderFamily`。所以 `testProvider` 就是一个 `NotifierProviderFamily` 实例：

```dart
static const family = NotifierProviderFamilyBuilder();

final class NotifierProviderFamilyBuilder {
  const NotifierProviderFamilyBuilder();

  /// {@macro riverpod.autoDispose}
  NotifierProviderFamily<NotifierT, StateT, ArgT> call<NotifierT extends Notifier<StateT>, StateT, ArgT>(
    NotifierT Function(ArgT arg) create, {
      String? name,
      Iterable<ProviderOrFamily>? dependencies,
      Retry? retry,
      bool isAutoDispose = false,
    }) {
    return NotifierProviderFamily<NotifierT, StateT, ArgT>.internal(
      create,
      name: name,
      isAutoDispose: isAutoDispose,
      dependencies: dependencies,
      retry: retry,
    );
  }

  ...
}
```

## 继承关系

```
NotifierProviderFamily (final)
    ↑
    └── ClassFamily (final) 类类型 Family 基类
            ↑
            └── Family (base) Family 基类
                    ↑
                    └── ProviderOrFamily (sealed) 所有 provider / Family 基类
```

最终也继承自 `ProviderOrFamily`。之前小节里，`Provider` 最终也是继承自 `ProviderOrFamily`，因为 `ProviderOrFamily` 提供了 `isAutoDispose`：

- 对 `Provider`：`isAutoDispose` 表示该 `Provider` 对应的 `ProviderElement` 是否可以自动销毁。
- 对 `Family`：`isAutoDispose` 表示属于当前 `Family` 的那些 `ProviderElement` 是否可以自动销毁。

## ClassFamily.call

`ClassFamily` 提供了一个重要方法：

```dart
ProviderT call(ArgT argument) {
  return _providerFactory(
    () => _createFn(argument),
    name: name,
    isAutoDispose: isAutoDispose,
    from: this,
    retry: retry,
    argument: argument,
    dependencies: null,
    $allTransitiveDependencies: null,
  );
}
```

这也是 `call` 方法，可以直接用 `Family` 实例调用；执行 `testProvider(1)` 时走的就是这里。  
返回类型是 `ProviderT`，说明返回的是具体的 `Provider` 实例。  
`_providerFactory`、`_createFn` 都是外部传入的参数，下面从构造函数把调用链串起来。

### ClassFamily 构造函数

```dart
final ClassProviderFactory<NotifierT, ProviderT, CreatedT, ArgT> _providerFactory;

final NotifierT Function(ArgT arg) _createFn;

ClassFamily(
  this._createFn, {
    required ClassProviderFactory<NotifierT, ProviderT, CreatedT, ArgT> providerFactory,
    required super.name,
    required super.dependencies,
    required super.$allTransitiveDependencies,
    required super.isAutoDispose,
    required super.retry,
  }) : _providerFactory = providerFactory;
```

### NotifierProviderFamily 构造函数

```dart
NotifierProviderFamily.internal(
  super._createFn, {
    super.name,
    super.dependencies,
    super.isAutoDispose = false,
    super.retry,
  }) : super(
    providerFactory: NotifierProvider.internal,
    $allTransitiveDependencies: computeAllTransitiveDependencies(
      dependencies,
    ),
  );
```

以及上面的 `NotifierProviderFamilyBuilder.call`：

```dart
NotifierProviderFamily<NotifierT, StateT, ArgT> call<NotifierT extends Notifier<StateT>, StateT, ArgT>(
  NotifierT Function(ArgT arg) create, {
    String? name,
    Iterable<ProviderOrFamily>? dependencies,
    Retry? retry,
    bool isAutoDispose = false,
  },
) {
  return NotifierProviderFamily<NotifierT, StateT, ArgT>.internal(
    create,
    name: name,
    isAutoDispose: isAutoDispose,
    dependencies: dependencies,
    retry: retry,
  );
}
```

## 小结

- `_providerFactory`：`NotifierProvider.internal`
- `_createFn`：`TestNotifier.new`

因此 `ClassFamily.call(argument)`（也就是 `testProvider(1)`）会**新建**一个 `NotifierProvider` 实例，并设置：

- `from`：当前的 `NotifierProviderFamily` 实例  
- `argument`：例如 `1`  
- `isAutoDispose`：`NotifierProviderFamily` 中的 `isAutoDispose` 值  

每次调用 `ClassFamily.call` 都会得到新的 `Provider` 对象；但当 `from` 与 `argument` 与已有 key 在 `hashCode` / `==` 意义下相同时，在 `pointers` 里仍会命中同一个 `ProviderElement` 实例。

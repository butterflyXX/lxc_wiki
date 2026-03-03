# Flutter 状态管理 - Riverpod 系列教程

本目录包含 Riverpod 状态管理的完整系列教程，从基础原理到高级用法。

## 📚 文档索引

### 1. [Riverpod 状态管理详解 1 - 基本原理](Riverpod状态管理详解1 - 基本原理.md)

**核心内容：**
- ProviderScope 和 UncontrolledProviderScope 的关系
- ProviderContainer 的状态存储机制
- InheritedWidget 在 Riverpod 中的应用
- ref 的两种类型：ProviderElement 和 WidgetElement
- read 和 watch 方法的区别和实现原理
- 依赖关系的管理（_dependents 和 _dependencies）

**适合人群：** 初学者，想了解 Riverpod 底层原理的开发者

---

### 2. [Riverpod 状态管理详解 2 - provider 与 notifier](Riverpod状态管理详解2 - provider与notifier.md)

**核心内容：**
- StateProvider 的实现原理
- ProviderElementProxy 的作用机制
- notifier 属性的工作原理
- state 属性的访问方式
- ProviderListenable 接口的作用

**适合人群：** 已掌握基础，想深入了解 Provider 机制的开发者

---

### 3. [Riverpod 状态管理详解 3 - provider 与 select](Riverpod状态管理详解3 - provider与select.md)

**核心内容：**
- select 机制的工作原理
- 如何通过 select 优化性能
- 细粒度更新的实现方式

**适合人群：** 需要优化应用性能的开发者

---

### 4. [Riverpod 状态管理详解 4 - AutoDisposeProvider](Riverpod状态管理详解4 - AutoDisposeProvider.md)

**核心内容：**
- AutoDisposeProvider 的使用场景
- 自动释放机制的原理
- 何时使用 AutoDisposeProvider

**适合人群：** 需要管理资源释放的开发者

---

### 5. [Riverpod 状态管理详解 5 - NotifierProvider](Riverpod状态管理详解5 - NotifierProvider.md)

**核心内容：**
- NotifierProvider 的详细解析
- 复杂状态管理的最佳实践
- Notifier 类的使用方式

**适合人群：** 需要管理复杂状态的开发者

---

### 6. [Riverpod 状态管理详解 6 - ProviderFamily](Riverpod状态管理详解6 - ProviderFamily.md)

**核心内容：**
- ProviderFamily 的使用场景
- 参数化 Provider 的实现
- 如何根据参数创建不同的 Provider 实例

**适合人群：** 需要动态创建 Provider 的开发者

---

## 🎯 学习路径建议

1. **入门阶段**：从文档 1 开始，理解 Riverpod 的基本原理
2. **进阶阶段**：阅读文档 2-3，掌握 Provider 的核心机制
3. **高级阶段**：学习文档 4-6，掌握高级用法和最佳实践

---

## 🔗 相关链接

- [返回 Flutter 目录](../README.md)
- [返回知识库首页](../../README.md)
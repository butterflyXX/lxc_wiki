[← 返回状态管理目录](README.md)

`Riverpod`架构设计方面非常优秀,整体是一种POP架构(面相协议开发),举个例子不管是什么类型的`Provider`,`watch`/`read`对他的定义都是`ProviderListenable`,`ProviderListenable`是协议类,如果不了解POP设计模式,看懂`Riverpod`是有压力的
`Riverpod`中比较重要的概念都进行了深度的拆分,使用了继承(`extends`)/混入(`with`)/协议(`implements`),学习完`Riverpod`的源码,相信你对这些概念会有更深入的理解
`Riverpod`中包含以下几个重要概念,由于`Riverpod`还支持覆盖,比如`ProviderScope`嵌套`ProviderScope`使用,会涉及到很多`Override`操作,对整体框架学习影响不大,本小节先跳过
- Provider
- ProviderElement
- ProviderSubscription

## Provider
![Provider 架构图](/resource/svg/Riverpod-Provider架构图.svg)
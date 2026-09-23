# 📱 移动端

> Android 客户端开发方向，语言以 Kotlin 为主（Google 官方 Kotlin-first），覆盖 Kotlin 语言、JVM/Java 生态与 Jetpack 体系，并延伸 Flutter 跨端开发。

## 学习要点

- Android 体系：四大组件与生命周期、Handler/Looper 消息机制、Binder IPC、View 绘制与事件分发、进程/线程与 ANR、内存与性能优化。
- 语言基础：空安全、`val/var`、扩展函数、密封类、作用域函数。
- Kotlin 与 JVM/Java：字节码编译、互操作注解（`@JvmStatic` 等）、混编迁移。
- 协程：`suspend`、`Dispatchers`、结构化并发、Flow。
- Android 使用：Jetpack Compose、ViewModel/Lifecycle、Retrofit + 协程、Android KTX。
- Flutter：三棵树（Widget/Element/RenderObject）、渲染管线（Build/Layout/Paint/Compositing/Rasterize）、性能优化。
- 面试：语言特性辨析、协程原理、Android 生命周期与异步方案对比、Flutter 渲染原理。

## 文档

- [Android 基础](Android基础.md)：系统架构、四大组件与生命周期、Handler 与 Binder、View 绘制与事件分发、进程线程与内存、存储网络、Jetpack/Compose、性能优化。
- [Android 面试题](Android面试题.md)：组件/消息机制/IPC/View/线程 ANR/内存性能/架构 Compose 高频题与开放题。
- [Kotlin 语言基础](Kotlin语言基础.md)：Kotlin 是什么、与 JVM/Java 的关系、Android 中的使用、核心语法速览。
- [Kotlin 面试题](Kotlin面试题.md)：基础语法 / 空安全 / 协程 / Android 使用 / JVM 互操作 / 进阶 高频题与要点。
- [Kotlin 语法糖对照](Kotlin语法糖.md)：Android 常用语法糖（空安全、data class、扩展函数、作用域函数、协程 KTX 等）与手写 Java 等价物逐条对照。
- [Flutter 渲染树与渲染流程](Flutter/README.md)：三棵树职责与 diff 复用、一帧渲染管线、布局/绘制边界与性能优化、高频面试题。

## 推荐资料

- [Kotlin 官方文档](https://kotlinlang.org/docs/home.html)
- [Android Developers 官方文档](https://developer.android.com/)
- 《第一行代码 Android（第 3 版）》（Kotlin 版）
- [Kotlin 中文站](https://www.kotlincn.net/)

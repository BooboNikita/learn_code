# 📱 移动端

> Android 客户端开发方向，语言以 Kotlin 为主（Google 官方 Kotlin-first），覆盖 Kotlin 语言、JVM/Java 生态与 Jetpack 体系，并延伸 Flutter、React Native 跨端开发。

## 学习要点

- Android 体系：四大组件与生命周期、Handler/Looper 消息机制、Binder IPC、View 绘制与事件分发、进程/线程与 ANR、内存与性能优化。
- 语言基础：空安全、`val/var`、扩展函数、密封类、作用域函数。
- Kotlin 与 JVM/Java：字节码编译、互操作注解（`@JvmStatic` 等）、混编迁移。
- 协程：`suspend`、`Dispatchers`、结构化并发、Flow。
- Android 使用：Jetpack Compose、ViewModel/Lifecycle、Retrofit + 协程、Android KTX。
- Flutter：三棵树（Widget/Element/RenderObject）、渲染管线（Build/Layout/Paint/Compositing/Rasterize）、性能优化。
- React Native：原生渲染原理、Bridge 与 JSI、新架构四件套（Fabric/TurboModules/Codegen）、Yoga 与 Flexbox 差异、列表与启动优化、原生互操作与热更新。
- 面试：语言特性辨析、协程原理、Android 生命周期与异步方案对比、Flutter 渲染原理、RN 架构演进与性能定位。

## 文档

- [Android 基础](Android基础.md)：系统架构、四大组件与生命周期、Handler 与 Binder、View 绘制与事件分发、进程线程与内存、存储网络、Jetpack/Compose、性能优化。
- [Android 面试题](Android面试题.md)：组件/消息机制/IPC/View/线程 ANR/内存性能/架构 Compose 高频题与开放题。
- [Kotlin 语言基础](Kotlin语言基础.md)：Kotlin 是什么、与 JVM/Java 的关系、Android 中的使用、核心语法速览。
- [Kotlin 面试题](Kotlin面试题.md)：基础语法 / 空安全 / 协程 / Android 使用 / JVM 互操作 / 进阶 高频题与要点。
- [Kotlin 语法糖对照](Kotlin语法糖.md)：Android 常用语法糖（空安全、data class、扩展函数、作用域函数、协程 KTX 等）与手写 Java 等价物逐条对照。
- [Flutter 渲染树与渲染流程](Flutter/README.md)：三棵树职责与 diff 复用、一帧渲染管线、布局/绘制边界与性能优化、高频面试题。
- [React Native 基础](ReactNative基础.md)：原生渲染定位与跨端对比、Bridge 到新架构（JSI/TurboModules/Fabric/Codegen）、线程模型与渲染流程、Yoga 与 Flexbox 差异、组件与列表优化、原生互操作、动画、工程化与版本演进。
- [React Native 面试题](ReactNative面试题.md)：定位对比 / 架构与通信 / 渲染与布局 / 组件与状态 / 原生互操作 / 动画 / 工程化与热更新 / 开放题 高频题与要点。

## 推荐资料

- [Kotlin 官方文档](https://kotlinlang.org/docs/home.html)
- [Android Developers 官方文档](https://developer.android.com/)
- 《第一行代码 Android（第 3 版）》（Kotlin 版）
- [Kotlin 中文站](https://www.kotlincn.net/)
- [React Native 官方文档](https://reactnative.dev/docs/getting-started)
- [React Native Blog（版本演进最可靠来源）](https://reactnative.dev/blog)
- [React Native Directory（第三方库的新架构适配状态）](https://reactnative.directory/)

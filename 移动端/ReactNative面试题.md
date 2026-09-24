# React Native 面试题

> React Native 高频面试题整理：定位与对比 → 架构演进（Bridge/JSI/新架构）→ 线程与渲染 → 组件与样式 → 列表与性能 → 原生互操作 → 动画 → 工程化与热更新 → 开放题。每题为「考察点 + 回答要点」，配合 [React Native 基础](ReactNative基础.md) 使用。

## 一、定位与技术选型

### 1.1 React Native 的原理是什么？它和 H5 + WebView 有什么本质区别？

- **考察点**：是否理解"原生渲染"这一核心，而非把它当成套壳 WebView。
- **要点**：
  - RN 用 JS/TS 描述 UI，但**最终渲染的是平台原生控件**（Android `View`、iOS `UIView`），不是浏览器 DOM。
  - H5 方案是 WebView 里跑网页，渲染由浏览器内核完成，与原生之间隔着 WebView 通信（JSBridge），交互与性能上限较低。
  - 因此 RN 的触摸、滚动、手势、原生控件外观都天然贴近原生；跨端视觉一致性不如自绘方案（如 Flutter）。
  - RN 的跨线程通信是**框架内部机制**（Bridge/JSI），不需要业务自己写 JSBridge。
  - 一句话：**"RN 不画 UI，它指挥原生控件画 UI"**。

### 1.2 React Native 和 Flutter 怎么选？

- **考察点**：架构级取舍，而非简单的好/坏。
- **要点**：
  - **渲染**：Flutter 自绘（Skia/Impeller），跨端一致；RN 用原生控件，更"像原生"但双端有差异。
  - **语言/生态**：Flutter 用 Dart，生态相对封闭但内置组件丰富；RN 用 JS/TS，可直接复用 Web 生态与大量 JS 库，社区规模更大。
  - **包体积**：RN 需带 JS 引擎，Flutter 需带渲染引擎 + Dart runtime，通常 Flutter 更大。
  - **动态能力**：RN 可热更新 JS Bundle（受平台政策约束）；Flutter 动态化能力弱。
  - **性能**：重动画/重绘场景 Flutter 更可控；RN 在 0.76+ 新架构后，列表滑动与手势的抖动问题大幅改善。
  - **团队**：前端团队转 RN 成本低；Flutter 需要重新学 Dart 与 Flutter 组件体系。

### 1.3 React Native 为什么不全用原生开发？它的适用边界在哪？

- **考察点**：工程判断力。
- **要点**：
  - 收益：一套代码覆盖双端业务层、前端人才复用、迭代与热更新快、UI 与业务逻辑复用。
  - 代价：需要维护原生侧（原生模块、构建、上架）、版本升级与第三方库适配成本。
  - 适合：业务型 / 内容型 / CRUD 为主的中大型 App；不适合：重度依赖自绘、超低延迟渲染（游戏、复杂图表、AR）、极致包体积要求的场景。

## 二、架构与通信机制

### 2.1 React Native 用了几条线程？各自做什么？

- **考察点**：线程模型，是理解 RN 性能问题的地基。
- **要点**：
  - **JS 线程**：执行 JS Bundle、React 渲染（Reconciler）、业务逻辑与状态更新。**JS 是单线程的**。
  - **UI / 主线程（原生）**：创建与更新原生控件、手势与触摸分发、绘制。原生模块默认也跑在这里。
  - **Shadow / 后台线程**：老架构下负责 Yoga 布局计算与影子树维护；新架构把影子树与布局挪到 **C++ 共享层**。
  - **后果**：JS 线程被一段耗时逻辑占住，会同时卡住 UI 更新和手势响应——所以重活要"下发原生"（`useNativeDriver`、worklet、原生模块）。

### 2.2 老架构的 Bridge 机制是怎样的？有什么问题？

- **考察点**：能否说清 Bridge 的三个特性及其代价。
- **要点**：
  - 特性：**异步 + 序列化（JSON）+ 批量投递**。所有 JS ↔ Native 的通信都编码成消息放进队列，由桥定时批量搬运。
  - 问题：
    - **异步**：JS 拿不到同步结果，布局测量只能回调（`measure` 回调式 API），催生了 `setNativeProps` 这类逃生舱。
    - **序列化开销**：大数据跨桥要编解码，是主要瓶颈之一。
    - **队列阻塞**：JS 一忙，UI 更新排队 → 掉帧、白屏、手势无响应。
    - **启动昂贵**：所有 NativeModule 启动时集中初始化。
  - 结论：Bridge 是旧架构性能瓶颈的根源，也是 2018 年启动架构重写的动因。

### 2.3 新架构包含哪些部分？各自解决什么问题？

- **考察点**：新架构四件套是否讲得清、能对号入座。
- **要点**：
  - **JSI**：JS 引擎与 C++ 之间的通用抽象层，**引擎无关**（Hermes/JSC/V8 都能接）。让 JS 能**持有原生对象引用并同步调用**，取代 JSON 异步队列——这是一切的基础。
  - **TurboModules**：基于 JSI 的原生模块体系，**按需懒加载 + Codegen 类型安全**，启动不再初始化所有模块。
  - **Fabric**：新渲染器，**C++ 共享影子树**，同步布局与挂载，支持 React 18/19 的并发渲染（Suspense、Transition、`useTransition`）。
  - **Codegen**：从 TS/Flow 类型定义自动生成 JS ↔ 原生胶水代码，接口在编译期校验，去掉手写样板。
  - **Bridgeless**：以上全开、旧桥彻底移除的最终形态。

### 2.4 JSI 和 Bridge 的本质区别是什么？为什么说 JSI 是关键？

- **考察点**：能否从"同步/异步""引用传递/值传递"两个维度讲透。
- **要点**：
  - Bridge：传的是**序列化后的值**，跨线程异步队列，调用与返回天然分离。
  - JSI：JS 直接持有 **C++ HostObject 的引用**，可以**同步**调用方法、同步读取属性（如常量）。
  - 关键性：同步能力让 **Fabric 的同步布局测量**、**TurboModule 的同步方法/常量**、**Reanimated 的 worklet** 都成为可能，从根上消除了"布局往返延迟"和"跨桥串行排队"。
  - 代价：**旧原生模块不迁移就会失效**（只能走 interop 兼容层），且直接持有对象引用要小心内存与线程安全。

### 2.5 React Native 的新架构从哪个版本开始默认开启？还能退回旧架构吗？

- **考察点**：对近期版本节奏的了解（区分"默认"和"唯一"）。
- **要点**：
  - **0.76（2024.10）**：新架构**默认开启**，仍可通过 `newArchEnabled=false`（Android）/ `RCT_NEW_ARCH_ENABLED=0`（iOS）退回。
  - **0.81**：**最后一个支持旧架构的版本**，同时提供迁移告警与辅助。
  - **0.82（2025.10）**：新架构成为**唯一**架构，回退开关**失效**；同时把 React 升级到 19.1.1。
  - **0.84**：**Hermes V1 成为默认引擎**；iOS 默认使用预编译 `.xcframework`。
  - 迁移建议：**不要跨版本硬跳**。先升到 0.81 打开新架构跑通，再升 0.82+。

### 2.6 新架构下 JS 和原生有哪几种通信方式？

- **考察点**：互操作方式的全景掌握。
- **要点**：
  - **Native Module / TurboModule**：JS 调原生（相机、蓝牙、SDK）。
  - **Native UI Component / Fabric Component**：把原生视图暴露成 JSX 组件（地图、视频播放器）。
  - **Events**：原生通过 `NativeEventEmitter` / `DeviceEventEmitter` 向 JS 上报事件。
  - **JSI HostObject**：双向且**同步**，JS 直接持有 C++ 对象引用。
  - 补充：旧桥下还有 `UIManager.dispatchViewManagerCommand`（命令式操作原生视图），新架构下行为已变化，需逐个用例回归。

## 三、渲染与布局

### 3.1 一次状态更新到屏幕显示，RN 内部发生了什么？

- **考察点**：能否完整串起 Reconciler → Shadow Tree → Yoga → 挂载。
- **要点**：
  - ① 状态变化 → ② **React Reconciler（ReactFabric）**在 JS 线程做 diff → ③ 生成/更新 **React Shadow Tree（C++）** → ④ **Yoga** 单遍计算布局 → ⑤ Commit 提交挂载指令 → ⑥ **UI 线程同步创建/更新原生控件** → 上屏。
  - 新架构与老架构的差别集中在 ③④⑤：老架构要经 Bridge 排队与序列化，新架构在 C++ 内同步完成。

### 3.2 Yoga 是什么？为什么 RN 的布局比浏览器快？

- **考察点**：是否理解"单遍布局"这一关键差异。
- **要点**：
  - Yoga 是 C/C++ 实现的 **Flexbox 子集布局引擎**，跨平台同一份算法，保证三端布局一致。
  - 以 `YGNode` 组成布局树，`YGNodeCalculateLayout` **一次自顶向下算完**，没有浏览器 CSS 的多轮 reflow / 复杂尺寸传播。
  - RN 0.74 起内置 **Yoga 3.0**，`gap`、`flex` 等语义与 Web 更对齐。

### 3.3 RN 的 Flexbox 和 Web 的 Flexbox 有哪些关键差异？

- **考察点**：实际写代码时最容易踩的坑。
- **要点**：
  - `flexDirection` 默认是 **`column`**（Web 默认 `row`）。
  - **没有 CSS 单位**，无单位数字即 dp（密度无关像素），不支持 em/rem。
  - **文本必须包在 `<Text>` 里**，`View` 不能直接放字符串。
  - **样式不继承**（只有嵌套 `<Text>` 内部继承字体相关属性）。
  - 定位只有 `relative`（默认）与 `absolute`，没有 `fixed`/`sticky`。
  - 阴影：iOS 用 `shadow*` 系列，**Android 只能用 `elevation`**。
  - 没有媒体查询，用 `useWindowDimensions()` / `Dimensions`。

### 3.4 RN 里 `flex: 1` 不生效是什么原因？

- **考察点**：布局约束理解与排查思路。
- **要点**：
  - `flex: 1` 需要有"可分配的空间"，父容器高度必须是确定的（被自身父级约束住）。
  - Android 上常见问题是 `flex: 1` 链路上溯不到根（中间某一层没有 `flex: 1` 或固定高度）→ 高度塌成 0，表现为"列表空白但能滚动"或"文本不显示"。
  - 排查：临时给容器加 `backgroundColor` 看区域是否存在；逐层确认高度来源；列表需 `flex: 1` 才能撑满。

## 四、组件、状态与渲染优化

### 4.1 `ScrollView` 和 `FlatList` 的区别？长列表为什么必须用 FlatList？

- **考察点**：虚拟列表机制。
- **要点**：
  - `ScrollView` **一次性渲染全部子元素**，内存与首屏开销随内容线性增长，只适合少量静态内容。
  - `FlatList` 基于 `VirtualizedList`，**按渲染窗口懒渲染 + 回收**，只保留可视区上下若干屏的 cell。
  - `SectionList` 是带分组的 FlatList；`FlashList` 用**单元回收复用**，超长列表更优。
  - 注意：**不要嵌套同方向的滚动容器**（VirtualizedList 嵌套会告警并失去虚拟化收益）。

### 4.2 FlatList 有哪些性能优化手段？

- **考察点**：是否有真实优化经验。
- **要点**：
  - `keyExtractor` 保证 key 稳定；`getItemLayout` 固定行高时**跳过测量**。
  - `initialNumToRender`（首屏量）、`maxToRenderPerBatch` + `updateCellsBatchingPeriod`（每批量与间隔）、`windowSize`（渲染窗口）。
  - `removeClippedSubviews`（Android 收益明显，注意副作用）。
  - **cell 组件用 `React.memo`，回调用 `useCallback`，避免内联函数/内联对象**——否则每次父组件渲染都会让整列重渲染，memo 直接失效。
  - 图片给固定宽高并做缓存；避免 cell 内部维护复杂状态或嵌套 FlatList。
  - **根因导向**：卡顿多半来自"cell 太重 / key 不稳定导致整列重建 / 每帧 setState"，而不是列表参数没调好。

### 4.3 RN 中如何做渲染优化？`memo`、`useMemo`、`useCallback` 怎么用？

- **考察点**：是否理解浅比较成本与适用边界。
- **要点**：
  - `React.memo`：组件级浅比较 props，相同则跳过重渲染。
  - `useMemo`：缓存**计算结果**（昂贵计算、需要稳定引用的对象）。
  - `useCallback`：缓存**函数引用**，主要为了让子组件的 `memo` 生效。
  - 三者都不是免费的（自身有比较与依赖数组成本）——**先定位瓶颈再优化**，不要无脑包裹。
  - 更有效的往往是把大组件**拆小**，让状态更新的影响面缩小。

### 4.4 RN 的状态管理怎么选？

- **考察点**：能否清晰区分"服务端状态"与"客户端状态"。
- **要点**：
  - 局部 UI 状态 → `useState` / `useReducer`；跨层低频数据（主题、登录态）→ `Context`。
  - 客户端全局状态 → **Zustand / Jotai / MobX**（轻量按需订阅）或 **Redux Toolkit**（大型应用、生态与调试成熟）。
  - **服务端状态 → React Query / SWR**（缓存、重试、失效、乐观更新），不要塞进全局 store。
  - 持久化 → MMKV（mmap + protobuf，优于 AsyncStorage）。

### 4.5 `useEffect` 和类组件的生命周期怎么对应？常见坑是什么？

- **考察点**：Hooks 心智模型。
- **要点**：
  - `useEffect(fn, [])` ≈ `componentDidMount`；`useEffect(fn, [dep])` ≈ `componentDidUpdate`（dep 变化时）；返回的清理函数 ≈ `componentWillUnmount`。
  - 必须在清理函数中**取消订阅、清定时器、中断请求**，否则内存泄漏 / 卸载后 setState。
  - 坑：依赖数组遗漏导致闭包捕获旧值（用函数式 `setState` 或把依赖补全）；对象/函数依赖每次都变导致 effect 反复执行（用 `useCallback`/`useMemo` 稳定）。

### 4.6 `ErrorBoundary` 能捕获哪些错误？不能捕获哪些？

- **考察点**：错误处理边界。
- **要点**：
  - 能捕获：子树**渲染期间**的同步错误（类组件 + `componentDidCatch` / `getDerivedStateFromError`）。
  - 不能捕获：**事件回调**里的错误、异步代码（Promise/定时器）的错误、自身抛出的错误、原生崩溃。
  - 异步与全局错误需要 `ErrorUtils.setGlobalHandler` + 原生崩溃上报（Sentry/Bugly）。

## 五、原生互操作

### 5.1 怎么实现一个原生模块？新架构下写法有什么变化？

- **考察点**：是否有写过原生侧代码。
- **要点**：
  - 新架构流程：写 **TS spec**（继承 `TurboModule`，`TurboModuleRegistry.getEnforcing`）→ 在 `package.json` 的 `codegenConfig` 声明 → Codegen 生成胶水代码 → 原生侧实现（iOS 用 `RCT_EXPORT_MODULE`/`RCTTurboModule`，Android 继承生成的 `NativeXxxSpec`）。
  - 同步能力：TS spec 中**不返回 Promise 的方法即为同步方法**，`readonly constants` 可同步读取——这是 JSI 带来的新能力。
  - 老写法：iOS `RCT_EXPORT_METHOD`、Android `ReactContextBaseJavaModule` + `@ReactMethod`，JS 侧 `NativeModules.XXX`；新架构下通过 **interop 兼容层**仍可运行，但常数与同步方法行为有差异，属技术债。

### 5.2 怎么把原生视图暴露给 JS？

- **考察点**：Native UI Component 的实现路径。
- **要点**：
  - 新架构用 `codegenNativeComponent<Props>('MyView')` 声明，原生侧提供 Component Descriptor（iOS）/ Fabric ViewManager（Android）。
  - 老架构是 `requireNativeComponent` + `RCTViewManager`（iOS）/ `SimpleViewManager`（Android）。
  - **旧 `RCTViewManager` 在 Fabric 下默认不渲染**，必须走 interop 层，且 prop 更新可能滞后一帧——这是新架构迁移中最容易"炸"的一类库（相机、图表、地图）。

### 5.3 JS 侧的大数据怎么传给原生？有什么限制？

- **考察点**：老架构的工程细节。
- **要点**：
  - 老架构下所有数据都要过 Bridge，**序列化 + 单次事务容量**是硬约束（Binder 事务约 1MB 量级）。
  - 大数据/二进制走曲线方案：写临时文件传路径、原生侧使用共享内存、图片/视频用 URI 而非 base64。
  - 新架构下 JSI 可以传引用，但**也不要把它当共享内存数据库用**——跨语言传大对象的成本依然存在，设计上应尽量"传句柄，不传内容"。

## 六、动画

### 6.1 RN 有哪几种动画方案？`useNativeDriver` 的原理和限制是什么？

- **考察点**：性能意识。
- **要点**：
  - `Animated`（内置）、`LayoutAnimation`（布局变化）、**Reanimated 3+**（worklet 跑 UI 线程，复杂手势首选）、`Gesture Handler` 配合作手势。
  - `useNativeDriver: true` 会把动画计算**下发到原生 UI 线程执行**，使动画不再依赖 JS 线程，卡顿显著减少。
  - 限制：只支持 **`transform` / `opacity`** 等非布局属性；驱动 `width/height/top/left` 在提交前就会抛错。0.85.1 起引入共享动画后端，布局属性也能走原生驱动（实验特性）。
  - 反模式：动画过程中每帧 `setState`（每帧触发 React 渲染），应直接用动画值驱动 `style`。

### 6.2 Reanimated 和 Animated 的本质区别是什么？

- **考察点**：是否理解 worklet 与 UI 线程。
- **要点**：
  - `Animated` 的动画逻辑仍在 JS 侧计算（除非 `useNativeDriver` 生效，但能力受限于非布局属性）。
  - Reanimated 通过 **worklet（在 UI 运行时执行的 JS 片段）** 把动画函数直接跑在 UI 线程，JS 线程被占满也不掉帧，且能与手势事件**同步跟手**。
  - 代价：需要 Babel 插件、worklet 与 JS 线程的变量共享有规则（需用 `runOnJS` / `runOnUI` 跨越）。

## 七、工程化、调试与发布

### 7.1 Metro 是什么？和 Webpack 有什么区别？

- **考察点**：构建体系理解。
- **要点**：
  - Metro 是 RN 的打包器，产物是**单个 JS Bundle**（`index.android.bundle` / `main.jsbundle`），支持 Babel transform、按需构建与 Fast Refresh。
  - 与 Webpack 的差别：定位不同，Metro 针对 RN 的"单 bundle + 原生运行时"设计，不做 code splitting（Web 需要），更关注增量构建速度与 bundle 体积。
  - 关键配置：`inlineRequires`（把 import 变成**惰性 require**，明显改善启动耗时）。

### 7.2 Hermes 是什么？为什么 RN 要用它？

- **考察点**：引擎层面的理解。
- **要点**：
  - Meta 为 RN 定制的 JS 引擎，**构建期 AOT 编译成字节码**（`hermesc`），运行时省去 parse/compile。
  - 收益：启动更快、内存更低、bundle 更小；配合 bytecode 缓存与预加载。
  - 代价：**不带 JIT**，纯 JS 的 CPU 密集计算不如 JSC/V8 → 复杂计算应该交给原生或 worklet。
  - 版本节奏：0.70 起 Android 默认；**0.84 起 Hermes V1 成为双端默认引擎**（字节码与集成面兼容，无需改代码）。

### 7.3 RN 的启动流程是怎样的？白屏怎么排查？

- **考察点**：能否给出可执行的排查路径。
- **要点**：
  - 冷启动：进程启动 → RN 容器初始化 → 加载 JS Bundle → 执行 JS（注册根组件）→ 首屏渲染 → 异步补齐数据/图片。
  - 白屏原因：① bundle 加载失败（路径/版本不匹配）② JS 在首屏前抛错（兜底 ErrorBoundary 都还没生效）③ 大量同步初始化阻塞 ④ 首屏强依赖网络/字体/图片。
  - 优化：`inlineRequires`、Hermes 字节码、**TurboModule 懒加载**（0.76+ 默认收益）、延迟初始化（启动后空闲再跑）、Splash + 骨架屏、减少启动期同步 IO。

### 7.4 RN 的热更新原理是什么？有什么限制？

- **考察点**：动态化能力与平台政策的平衡。
- **要点**：
  - 原理：**下发新的 JS Bundle 与资源**（CodePush / EAS Update / 自建），应用启动或指定时机加载新 bundle 覆盖旧版本。
  - 限制：**只能更新 JS 与 JS 侧资源，不能改原生代码**；涉及原生依赖变更时必须走应用商店发版。
  - 政策风险：iOS 审核对"动态下发改变主要功能"有限制，需遵守规则、避免绕过审核。
  - 工程稳健性：灰度分批、版本绑定（bundle 关联原生版本号）、回滚机制、签名校验、失败兜底用内置 bundle。

### 7.5 RN 怎么调试和定位性能问题？

- **考察点**：工具链熟练度。
- **要点**：
  - 官方 **React Native DevTools**（基于 CDP，替代已归档的 Flipper）+ React DevTools 看组件树与 Profiler。
  - Perf Monitor / Performance 面板看帧率、JS 线程与内存。
  - **二分法定位**：先判断是 **JS 线程**耗时（重渲染、重计算、大 JSON）还是 **UI 线程**耗时（视图层级过深、阴影、大图、原生绘制）。
  - 原生侧互补：Android 用 Systrace/Perfetto，iOS 用 Xcode Instruments。

### 7.6 如何减小 RN 应用的包体积？

- **考察点**：发布侧优化经验。
- **要点**：
  - Android 开启 R8/ProGuard 与资源压缩，用 `abiFilters` 裁剪不需要的 so 架构。
  - 图片转 WebP、字体子集化、移除未使用资源。
  - 按需引入第三方 SDK（很多库默认带全量埋点/地图）。
  - 升级到新架构：可移除大量旧桥代码，官方也提到这有助于精简引擎体积。

## 八、开放题

### 8.1 一个 RN 页面滑动卡顿，你会怎么定位和优化？

- **考察点**：系统性排查思路。
- **要点**：
  - ① 用 Perf Monitor / Performance 判断卡在 **JS 线程**还是 **UI 线程**。
  - ② JS 线程忙 → 查 cell 是否重渲染（memo/key/内联回调）、是否有滚动中 setState、是否有大计算。
  - ③ UI 线程忙 → 查视图层级、阴影与圆角、大图解码、嵌套滚动容器。
  - ④ 列表本身：`getItemLayout`、`windowSize`、`removeClippedSubviews`，超长列表换 FlashList。
  - ⑤ 动画/手势：改 `useNativeDriver` 或 Reanimated worklet。
  - ⑥ 复测并量化（帧率、JS 耗时），确认优化有效而不是"感觉快了"。

### 8.2 让你把公司一个 0.72 版本的 RN 老项目升级到最新版，你怎么做？

- **考察点**：风险意识与迁移策略。
- **要点**：
  - ① **先锁版本**：不要在可预发布的 canary 上做架构迁移。
  - ② **先升到 0.81**（最后一个支持旧架构的版本），逐个小版本升级、跑通构建与回归，避免跨版本跳级导致错误难以归因。
  - ③ **依赖审计**：把每个原生依赖分三类——已适配新架构 / 只能靠 interop / 完全不兼容（等维护者、换 fork 或自研）。这一步决定工期。
  - ④ 在 0.81 上打开新架构（`newArchEnabled=true` / `RCT_NEW_ARCH_ENABLED=1`），**保留回退能力**的前提下修问题（`setNativeProps`、`UIManager.dispatchViewManagerCommand`、手势时序、旧 ViewManager）。
  - ⑤ 稳定后升 0.82+（此时回退能力消失，但已不依赖旧架构）。
  - ⑥ Hermes V1 作为**独立决策**：先在灰度渠道验证启动耗时、TTI、崩溃率对比基线，再放量。
  - ⑦ 全程用测试覆盖线程与内存回归。

### 8.3 RN 项目里哪些逻辑不该放在 JS 侧？

- **考察点**：边界设计能力。
- **要点**：
  - 大数据量的序列化/解析、加密解密、音视频处理、图像压缩、复杂数学与图表计算。
  - 高频且必须低延迟的交互：手势跟手、复杂动画（用 worklet / 原生驱动）。
  - 需要常驻后台的任务：定位、推送、下载、蓝牙连接（原生能力更可靠）。
  - 判断标准：**是不是受 JS 单线程与无 JIT 的限制**，是不是需要平台 API/常驻能力。

## 九、小结

- 面试主线通常是：**为什么是原生渲染 → Bridge 的三个特性与代价 → 新架构四件套（JSI/TurboModules/Fabric/Codegen）→ 线程与渲染流程 → 列表与渲染优化 → 原生互操作 → 启动与热更新**。
- 最容易拉开差距的两点：**能讲清"同步"为什么是新架构的关键**，以及**能给出可执行的性能定位路径**（先分清 JS 线程还是 UI 线程）。
- 版本类问题注意区分"**默认**"与"**唯一**"：0.76 默认新架构，0.82 起只剩新架构。

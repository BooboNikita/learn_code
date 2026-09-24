# React Native 基础

> React Native（RN）是 Meta 开源的跨端框架：用 React 的声明式范式写业务，用 JavaScript/TypeScript 描述 UI，最终**渲染成平台原生控件**（不是 WebView）。本文按「定位 → 架构 → 渲染 → 组件 → 原生互操作 → 工程化」梳理速查要点，配合 [React Native 面试题](ReactNative面试题.md) 使用。

## 一、React Native 是什么

### 1.1 定位与设计理念

- 一句话：**Learn once, write anywhere**（注意不是 Write once, run anywhere）。
- 三层结构：
  - **JS 层**：React 组件、业务逻辑，跑在 JS 引擎（Hermes）里。
  - **中间层**：C++ 核心（Yoga、Shadow Tree、JSI），负责布局计算与通信。
  - **原生层**：Android/iOS 原生控件（`android.view.View` / `UIView`），负责最终绘制。
- 核心理念：**声明式 UI（UI = f(state)）+ 原生渲染 + 平台能力复用**。
- JSX 是语法糖，最终由 Babel 转成 `React.createElement` 调用，与 React 完全同源。

### 1.2 与主流方案对比

| 维度 | React Native | H5 + WebView | Flutter | 纯原生 |
| --- | --- | --- | --- | --- |
| 渲染载体 | **平台原生控件** | 浏览器内核（DOM） | 自绘引擎（Skia/Impeller） | 原生控件 |
| 渲染线程 | 原生负责绘制，JS 只描述 | WebView 合成 | 自建管线（UI + Raster） | 平台 |
| 语言 | JS / TS（+ 少量原生） | JS / TS | Dart | Kotlin / Swift |
| 一致性 | 各平台控件**不完全一致** | 一致（浏览器) | **高度一致** | 各自原生 |
| 包体积 | 中（带 JS 引擎） | 小（带内核则大） | 大（带引擎） | 小 |
| 生态/热更新 | 生态最大，可热更新 | 可热更新 | 生态较小 | 不可热更 |
| 典型短板 | 桥接/异步带来的抖动（新架构已大幅改善） | 性能与体验上限低 | Dart 生态与包体积 | 双端成本高 |

> 关键辨析：**RN 不"画"UI，它"指挥"原生控件**；Flutter 才是自己画每一个像素。所以 RN 的视觉与手势天然贴合平台，但跨端一致性不如 Flutter。

### 1.3 技术栈全景

- 语言：JavaScript / **TypeScript（0.71 起官方模板默认 TS）**。
- UI：React（函数组件 + Hooks 为主）。
- 布局：**Yoga**（C++ 实现的 Flexbox 子集）+ `StyleSheet`。
- 打包：**Metro**（不是 Webpack）。
- 引擎：**Hermes**（RN 专用 JS 引擎，构建期编译为字节码）。
- 导航：React Navigation（JS 实现）/ react-native-navigation（原生导航）。
- 原生工程：Android（Gradle + Kotlin/Java）、iOS（CocoaPods + Objective-C/Swift）。

## 二、架构演进：从 Bridge 到新架构

理解 RN 的一切性能与"卡顿"话题，都绕不开这次架构重写。

### 2.1 老架构（Bridge，2015–2024）

```
┌─────────── JS 线程 ───────────┐
│  React 代码（Reconciler）      │
│  生成 UI 操作指令              │
└───────────────┬───────────────┘
                │ ① JSON 序列化
                ▼
┌──────────── Bridge（异步队列，批量投递）────────────┐
└───────────────┬───────────────────────┬────────────┘
                │                       │
                ▼                       ▼
        ┌───────────────┐      ┌──────────────────┐
        │ Shadow 线程    │      │ Native/UI 线程    │
        │ Yoga 计算布局  │─────▶│ 创建/更新原生控件  │
        └───────────────┘      └──────────────────┘
```

- **三线程**：JS 线程（跑 React 与业务）、Shadow 线程（跑 Yoga 布局）、Native/UI 主线程（创建与更新原生控件、处理手势与绘制）。
- **通信只能走 Bridge**：所有跨线程数据都被**序列化成 JSON 消息**放进异步队列，批量、定时投递。
- 副作用（后来成为新架构要解决的问题）：
  - **异步**：JS 与原生无法同步取值 → `measure` 只能回调，`setNativeProps` 变成"逃生舱"。
  - **序列化开销**：大数组/大对象过桥要编解码，是主要瓶颈。
  - **队列阻塞**：JS 线程一忙，UI 更新就排队 → 掉帧、白屏、手势无响应。
  - **启动昂贵**：所有 NativeModule 在启动时集中初始化。

### 2.2 新架构四件套

| 模块 | 定位 | 解决什么 |
| --- | --- | --- |
| **JSI**（JavaScript Interface） | JS 引擎与 C++ 之间的**通用抽象层**，引擎无关（Hermes/JSC/V8 都能接） | 让 JS 能**持有原生对象引用并同步调用**，取代 JSON 异步队列 |
| **TurboModules** | 新一代原生模块体系，基于 JSI，**按需懒加载 + 类型安全** | 启动时不再初始化所有模块；调用不再序列化 |
| **Fabric** | 新一代渲染器，**C++ 共享影子树**，同步布局与挂载 | 消除布局往返延迟，支持并发渲染（React 18/19 的 Concurrent、Suspense、Transition） |
| **Codegen** | 从 TS/Flow 类型定义**生成** JS ↔ 原生胶水代码（C++/Java/ObjC） | 接口编译期校验，去掉手写样板与运行时反射 |

- **Bridgeless（无桥模式）**：以上全部启用、把旧 Bridge **彻底移除**的最终形态。0.82 起成为**唯一**架构。

```
JS（Hermes 字节码）
   │  JSI：直接 C++ 调用 / 持有 HostObject（同步）
   ▼
React Common（C++）
   ├── Fabric 渲染器 ── React Shadow Tree（C++，JS 与原生共享）
   ├── Yoga 布局引擎
   └── TurboModules ──── 原生模块实现（Kotlin/Swift）
   ▼
原生 UI 线程：同步创建/挂载原生控件 → 上屏
```

### 2.3 新老架构对比

| 维度 | 老架构 | 新架构 |
| --- | --- | --- |
| 通信方式 | Bridge，异步 + JSON 序列化 | JSI，同步 C++ 调用 |
| 影子树位置 | 原生侧（Java/ObjC） | **C++ 共享层**（JS 与原生都能访问） |
| 布局计算 | Shadow 线程异步 | C++ 同步（可用并发渲染调度） |
| 原生模块 | 启动时全量初始化 | TurboModule 懒加载 |
| 类型安全 | 运行时反射，易错 | Codegen 编译期校验 |
| 并发特性 | 不支持 | 支持 React 18/19 Concurrent / Suspense |
| 兼容性 | —— | 旧模块/旧视图需迁移，否则走 interop 兼容层 |

### 2.4 版本演进要点

| 版本 | 关键变化 |
| --- | --- |
| 0.59 | 支持 Hooks |
| 0.60 | 迁移 AndroidX、引入 **autolinking**（自动链接原生依赖） |
| 0.62 | 默认集成 Flipper 调试 |
| 0.64 | iOS 可选启用 Hermes |
| 0.68 | **新架构 opt-in**（Fabric / TurboModules 可试用） |
| 0.70 | Hermes 成为 Android 默认引擎 |
| 0.71 | 官方模板默认 **TypeScript** |
| 0.74 | 内置 **Yoga 3.0**（与 Web 的 Flexbox 语义更对齐） |
| **0.76**（2024.10） | **新架构默认开启**（仍可用 `newArchEnabled=false` 回退） |
| 0.81 | **最后一个支持旧架构的版本**；iOS 预编译构建开始试验 |
| **0.82**（2025.10） | **新架构成为唯一架构**，回退开关失效；Hermes V1 实验性 opt-in；升级到 React 19.1.1 |
| 0.84 | **Hermes V1 成为默认引擎**；iOS 默认下载预编译 `.xcframework`（编译时间大幅下降） |
| 0.85 | 移除旧桥最后的核心实现（桥彻底"死"了） |
| 0.86（2026.06） | 持续演进；新架构 + Hermes V1 成为生态基线 |

> 记忆锚点：**0.76 默认新架构 → 0.81 最后能回退 → 0.82 只能新架构**。做版本升级时，**别跨版本硬跳**，先落到 0.81 打开新架构跑通，再升 0.82+。

## 三、线程模型

### 3.1 三条线程的分工

| 线程 | 职责 | 不该做什么 |
| --- | --- | --- |
| JS 线程 | 执行 JS Bundle、React 渲染（Reconciler）、业务逻辑、状态更新 | 重计算、大 JSON 解析、同步阻塞 |
| UI / 主线程（原生） | 创建/更新原生控件、手势与触摸分发、绘制、系统回调 | —— |
| Shadow / 后台线程 | Yoga 布局计算、影子树维护（新架构下移至 C++ 共享层） | —— |

- **JS 是单线程的**：所有组件渲染与业务逻辑排队执行，一段耗时 JS 会**同时**卡住 UI 更新与手势响应。
- 因此重活要"切线程"或"交给原生"：
  - 交给原生：`Animated` + `useNativeDriver`、Reanimated worklet、原生模块里跑。
  - 交给别的 JS 运行时：`react-native-worklets` / 多 JS 线程方案、WebView 里跑、Worker 变通方案。
- 新架构下，布局与挂载不再往返 JS 线程，**卡顿的主要来源收敛为"JS 线程执行时间 + 原生渲染耗时"两项**。

### 3.2 与 Android/iOS 线程的对应关系

- RN 的"UI 线程"就是 Android 的 main thread / iOS 的 main thread，**原生模块默认也在这个线程**——这一点和 Android 原生开发一致，也是排查 ANR（Android 无响应）的切入点。

## 四、渲染流程

### 4.1 一次更新的完整链路

```
① setState / 状态变化
        ▼
② React Reconciler（ReactFabric）在 JS 线程做 diff
        ▼
③ 生成/更新 React Shadow Tree（C++，节点是 ReactShadowNode）
        ▼
④ Yoga 在 C++ 中单遍计算布局（flexbox 子集）
        ▼
⑤ Commit：新树提交 + 挂载指令（Mounting）
        ▼
⑥ UI 线程同步创建/更新原生控件 → 交给系统绘制上屏
```

- 新架构的关键点：**影子树在 C++**，JS 侧与原生侧看到的是同一份布局结果，布局可以在提交前同步拿到。
- 老架构等价流程由 Bridge 承担 ③④⑤ 的排队与序列化，所以才有明显延迟。

### 4.2 Yoga：为什么布局快

- **Flexbox 子集**，用 C/C++ 实现，跨平台同一份算法 → 三端布局一致。
- **单遍布局**：以 `YGNode` 组成布局树，`YGNodeCalculateLayout` 一次自顶向下算完，不像浏览器 CSS 存在多轮 reflow / 复杂尺寸传播。
- 对齐 Web 的改动在 Yoga 3.0（RN 0.74）落地，`gap`、`flex` 语义更接近 CSS。

### 4.3 Flexbox：RN 与 Web 的差异（高频考点）

| 项 | Web | React Native |
| --- | --- | --- |
| `flexDirection` 默认值 | `row` | **`column`** |
| `flex: 1` 含义 | grow 1 / shrink 1 / **basis 0%** | grow 1 / shrink 1 / **basis 0%**（行为接近） |
| `display` | 有 `block/inline/flex/grid` | 只有 flex 语义，`display: 'none'` 用于隐藏 |
| 单位 | px / em / rem / % | **无单位数字 = dp**（密度无关像素），不支持 % 之外的单位 |
| 文本 | 任意元素可含文本 | **文本必须包在 `<Text>` 里**（`View` 不能直接放字符串） |
| 样式继承 | 大多数属性可继承 | **不可继承**（仅嵌套 `<Text>` 内部继承字体相关） |
| 定位 | `static/relative/absolute/fixed/sticky` | 只有 `relative`（默认）与 `absolute` |
| 媒体查询 | 支持 | 无，用 `useWindowDimensions` / `Dimensions` |
| 阴影 | `box-shadow` | iOS `shadowColor/Offset/Opacity/Radius`；Android 用 `elevation` |
| 圆角/边框 | `border-radius` | `borderRadius`，不支持不同边不同色（部分） |

### 4.4 样式系统

- `StyleSheet.create({...})`：返回**注册后的 ID**，避免每次渲染新建对象，也便于静态校验。
- 样式属性是 **camelCase**（`backgroundColor`），值是数字或字符串，不是 CSS 字符串。
- 数组写法即"优先级合并"：`style={[styles.a, active && styles.b, {color: 'red'}]}`，后者覆盖前者。
- `StyleSheet.flatten` / `StyleSheet.compose` 用于合并；`StyleSheet.hairlineWidth` 是最细（1 物理像素）线宽。
- 主题方案：Context + 自定义 hook，或 `react-native-unistyles` / NativeWind（Tailwind 风格）等。

### 4.5 PixelRatio 与单位

- `dp`（Android）/ `pt`（iOS）是逻辑单位，`PixelRatio.get()` 拿到设备像素比，`PixelRatio.roundToNearestPixel()` 把尺寸对齐到物理像素以避免毛边。
- 设计稿按 750 / 375 宽出图时，通常按 375 基准换算或封装 `scale` 工具。

## 五、核心组件与 API

### 5.1 基础组件对应关系

| RN 组件 | 类比（Web / 原生） | 说明 |
| --- | --- | --- |
| `View` | `div` / `ViewGroup` | 布局容器，**不能直接放文本** |
| `Text` | `span` / `TextView` | 唯一承载文本的组件，支持 `numberOfLines`、`onPress` |
| `Image` | `img` / `ImageView` | `source={{uri}}` 或 `require('./a.png')`；支持 `resizeMode` |
| `ScrollView` | `overflow: scroll` | **一次性渲染全部子元素**，只适合少量内容 |
| `FlatList` | 虚拟列表 | 长列表首选，基于 `VirtualizedList` 懒渲染 |
| `SectionList` | 分组列表 | 带分组的 FlatList |
| `TextInput` | `input` | 受控/非受控、`keyboardType`、`returnKeyType` |
| `Pressable` | 按钮基元 | 新推荐，支持 `pressed` 状态回调 |
| `TouchableOpacity/Highlight` | 旧按钮 | 旧 API，仍广泛存在 |
| `Modal` | 弹窗 | 原生弹层 |
| `SafeAreaView` | —— | 适配刘海/安全区 |
| `KeyboardAvoidingView` | —— | 键盘遮挡处理 |
| `RefreshControl` | 下拉刷新 | 配 `FlatList.refreshing` |
| `ActivityIndicator` | loading | 原生菊花 |
| `Switch` / `Slider` | 开关 / 滑块 | —— |

### 5.2 常用 API / 模块

- `Dimensions`、`useWindowDimensions()`、`Platform.OS` / `Platform.select()`、`PixelRatio`。
- `AppState`（前后台）、`Linking`（唤起外部 App / 深链）、`PermissionsAndroid`、`Alert`、`Vibration`。
- `Keyboard`、`LayoutAnimation`、`InteractionManager`（等动画/手势结束再做重活）。
- `NativeModules`、`NativeEventEmitter` / `DeviceEventEmitter`（原生事件）。
- `Animated`、`PanResponder`（手势）。
- `StyleSheet`、`AccessibilityInfo`、`AppRegistry`。

### 5.3 组件与生命周期

- **函数组件 + Hooks 是现代写法**：`useState` / `useEffect` / `useMemo` / `useCallback` / `useRef` / 自定义 Hook。
- `useEffect` 的清理函数对应老架构 `componentWillUnmount`：**取消订阅、清定时器、中断请求**，否则泄漏。
- `useMemo` / `useCallback` / `React.memo` 是列表与子树渲染优化的主力，但**不要滥用**（本身有比较成本）。
- `ErrorBoundary`（类组件 + `componentDidCatch`）捕获子树渲染错误，做兜底 UI；**无法捕获事件回调与异步错误**（要全局 `ErrorUtils.setGlobalHandler`）。

## 六、列表与性能优化

### 6.1 列表选型

| 组件 | 机制 | 适用 |
| --- | --- | --- |
| `ScrollView` | 全量渲染 | 少量静态内容（表单、详情页） |
| `FlatList` | VirtualizedList，按窗口懒渲染 + 回收 | 通用长列表（**首选**） |
| `SectionList` | 分组虚拟列表 | 通讯录、分类列表 |
| `FlashList`（Shopify） | 单元**回收复用**（recycle） | 超长列表、复杂 cell，性能更好 |

### 6.2 FlatList 优化清单

| 手段 | 作用 |
| --- | --- |
| `keyExtractor` | 稳定 key，避免重复渲染与状态错位 |
| `getItemLayout` | 固定行高时**跳过测量**，滚动与定位更快 |
| `initialNumToRender` | 首屏渲染数量，影响首屏速度 |
| `maxToRenderPerBatch` / `updateCellsBatchingPeriod` | 每批渲染量与批次间隔 |
| `windowSize` | 渲染窗口（可视区上下各若干屏） |
| `removeClippedSubviews` | 移除屏幕外视图（Android 收益明显，注意副作用） |
| `React.memo` + `useCallback` | 避免 cell 因父组件重渲染而重渲染 |
| **避免内联函数/内联对象** | `renderItem={({item}) => ...}` 每次新建，破坏 memo |
| 图片固定宽高 + 缓存 | 减少布局抖动与网络请求 |
| `onEndReachedThreshold` | 无限滚动触底阈值调优 |

- **列表卡顿的典型根因**：cell 太重（嵌套 FlatList、大图无尺寸）、key 不稳定导致整列重建、`renderItem` 频繁新建、每次滚动都 setState。

## 七、状态管理与导航

### 7.1 状态管理选型

| 方案 | 特点 | 场景 |
| --- | --- | --- |
| `useState` / `useReducer` | 组件内状态 | 局部 UI 状态 |
| **Context** | React 内置，跨层传递 | 主题、语言、登录态等**低频**变化 |
| **Zustand / Jotai / MobX** | 轻量、按需订阅 | 中小型应用（Zustand 目前最流行） |
| **Redux Toolkit** | 规范化、可预测、中间件生态成熟 | 大型应用、复杂异步与调试需求 |
| **React Query / SWR** | 服务端状态（缓存、重试、失效、乐观更新） | **把"服务端数据"从全局状态里剥离** |
| MMKV / AsyncStorage | 本地持久化 | 缓存与偏好设置（MMKV 性能更好） |

- 实践建议：**服务端状态交给 React Query，客户端状态交给 Zustand/Redux**，别把两者混在一个 store 里。

### 7.2 导航

- **React Navigation**：纯 JS 实现，生态最全，含 `Stack / Tab / Drawer`，配 `react-native-screens`（原生屏幕优化）与 `react-native-safe-area-context`。
  - 深层链接（deep link）通过 `linking` 配置映射到路由。
  - 原生栈 `@react-navigation/native-stack` 基于原生导航容器，转场更接近原生。
- **react-native-navigation（Wix）**：原生导航实现，性能好但集成成本高。
- 页面间传参：路由 params（**只传可序列化数据**，别传函数/大对象），函数用 Context/事件解耦。

## 八、原生互操作（RN ↔ 原生）

### 8.1 四类通信方式

| 方式 | 方向 | 说明 |
| --- | --- | --- |
| **Native Module**（TurboModule） | JS → 原生 | JS 调用原生能力（相机、蓝牙、SDK） |
| **Native UI Component**（Fabric Component） | 原生 → JS | 把原生视图暴露成 JSX 组件（地图、播放器） |
| **Events** | 原生 → JS | `NativeEventEmitter` / `DeviceEventEmitter` 上报事件与数据 |
| **JSI HostObject** | 双向，**同步** | 新架构能力：JS 直接持有 C++ 对象引用并同步调用 |

### 8.2 新架构下的写法（Codegen spec）

```ts
// NativeDeviceInfo.ts —— 由 Codegen 生成胶水代码，原生侧实现
import type { TurboModule } from 'react-native';
import { TurboModuleRegistry } from 'react-native';

export interface Spec extends TurboModule {
  getBatteryLevel(): Promise<number>;      // 异步：Promise
  setBrightness(level: number): void;      // 同步：不返回 Promise 即同步调用
  readonly constants: { model: string };   // 常量：可同步读取
}

export default TurboModuleRegistry.getEnforcing<Spec>('DeviceInfo');
```

```js
// package.json 中声明，构建时触发 Codegen
{
  "codegenConfig": {
    "name": "RNDeviceInfoSpec",
    "type": "modules",
    "jsSrcsDir": "src"
  }
}
```

- 老写法（新旧兼容通过 **interop 兼容层**）：iOS `RCT_EXPORT_MODULE` + `RCT_EXPORT_METHOD`，Android `ReactContextBaseJavaModule` + `@ReactMethod`，JS 侧 `NativeModules.XXX`。
- **常量与同步方法**在旧桥下是异步兼容层最易出问题的地方，迁移时重点回归。

### 8.3 原生视图组件（Fabric Component）

```ts
import { codegenNativeComponent } from 'react-native';
export default codegenNativeComponent<Props>('MyNativeView');
```

- 需要原生侧提供 **Component Descriptor**（iOS）/ **ViewManager + Delegate**（Android，新架构用 `Fabric ViewManager`）。
- 旧 `RCTViewManager` 在 Fabric 下**默认不生效**，需要 interop 层（只是迁移过渡，不要长期依赖）。

## 九、动画

| 方案 | 机制 | 适用 |
| --- | --- | --- |
| `Animated`（RN 内置） | `Animated.Value` + `timing/spring/decay` | 常规动画 |
| `Animated` + `useNativeDriver: true` | 动画计算**下发到原生 UI 线程**，绕过 JS 线程 | 只支持 `transform` / `opacity` 等**非布局属性**，性能最好 |
| `LayoutAnimation` | 一次性布局变化动画 | 列表增删、展开收起 |
| **Reanimated 3+** | worklet 直接跑在 **UI 线程**，可与手势同步 | 复杂手势、跟手动画、布局动画（现代首选） |
| `react-native-gesture-handler` | 原生手势系统 | 与 Reanimated 搭配 |
| `moti` / `react-native-animatable` | 上层封装 | 快速实现 |

- **`useNativeDriver` 的边界**：不能驱动 `width/height/top/left` 等布局属性（会在提交前就抛错）。0.85.1 起引入共享动画后端，布局属性也能走原生驱动（仍属实验特性）。
- 动画交互中避免频繁 `setState`（每帧触发 React 渲染），优先使用动画值直接驱动 `style`。

## 十、工程化、调试与部署

### 10.1 Metro（打包器）

- RN 专用打包器，特点：**单 bundle 全量打包**、按需构建、Babel transform、HMR（热替换）。
- 关键配置（`metro.config.js`）：
  - `inlineRequires`（把 import 变为**惰性 require**，明显改善启动耗时）。
  - `resolver` / `transformer` 定制、`watchFolders`（monorepo）。
- 产物：`index.android.bundle`（Android）/ `main.jsbundle`（iOS）。

### 10.2 Bundle 加载与 Hermes

- **开发环境**：从 Metro Dev Server（`localhost:8081`）拉 bundle，配合 Fast Refresh。
- **生产环境**：bundle 打进 App 资源，运行时解压到应用私有目录，由 Hermes 加载。
- **Hermes**：Meta 为 RN 定制的 JS 引擎，**构建期 AOT 编译为字节码**，省去运行时 parse/compile：
  - 启动更快、内存占用更低、包更小。
  - **不带 JIT**（CPU 密集的纯 JS 计算不如 JSC/V8），这也是"复杂计算交给原生"的原因之一。
  - 0.70 起 Android 默认，iOS 逐步默认；**0.84 起 Hermes V1 成为默认引擎**（字节码与集成面兼容，无需改代码）。

### 10.3 调试工具

| 工具 | 用途 |
| --- | --- |
| **React Native DevTools** | 官方推荐（基于 CDP，替代已归档的 Flipper），支持断点、Console、React 组件树检查 |
| React DevTools | 检查组件树、props、状态、Profiler |
| Perf Monitor / Performance 面板 | 帧率、JS 线程耗时、内存 |
| Systrace / Perfetto（Android） | 原生侧线程与渲染耗时 |
| Xcode Instruments（iOS） | 卡顿、内存、能耗 |
| `console.log` / LogBox | 快速排查（LogBox 会聚合同类告警） |
| Detox / Maestro | E2E 自动化测试 |
| Jest + RN Testing Library | 单元与组件测试 |

### 10.4 打包发布与热更新

- **autolinking**（0.60+）：原生依赖自动链接，无需手动 `react-native link`。
- Android：Gradle 产物 AAB/APK，注意 Hermes 字节码编译任务（`hermesc`）与新架构的 CMake 编译时间（0.84 起 iOS 用预编译 `.xcframework`，编译时间大幅缩短）。
- iOS：CocoaPods 安装与管理。
- **Expo vs RN CLI**：
  - Expo（托管）：开箱即用、EAS Build 云构建、更新方便，原生定制通过 **Config Plugin / prebuild** 落地。
  - RN CLI（裸工程）：完全掌控 Android/iOS 工程，适合深度定制。
- **热更新**：本质是**下发新的 JS Bundle**（CodePush / EAS Update / 自建服务）→ 应用启动或指定时机加载新 bundle。
  - 限制：**只能更新 JS 与资源，不能改原生代码**；iOS 审核对"动态下发改变主要功能"有约束，需遵守平台政策。
  - 稳健做法：灰度、版本回滚、签名校验、与原生版本（`versionName`/`buildNumber`）绑定。

## 十一、性能与启动优化

| 方向 | 手段 |
| --- | --- |
| JS 线程 | 减少重渲染（memo / useCallback / 拆组件）、避免大 JSON 解析、把重计算放原生或 worklet、`InteractionManager` 延后重活 |
| 渲染 / 布局 | 减少 View 嵌套层级、避免频繁改变布局属性、动画走 `useNativeDriver` |
| 列表 | 见第六节；超长列表换 FlashList |
| 资源 | 图片按尺寸加载 + 缓存（FastImage / expo-image）、字体子集化、WebP |
| 启动（冷启动） | `inlineRequires`、Hermes 字节码、**TurboModule 懒加载**（0.76+ 默认收益）、延迟初始化、减少启动期同步 IO、Splash + 骨架屏 |
| 包体积 | R8/ProGuard（Android）、资源裁剪、`abiFilters` 裁剪 so、按需引入 SDK、升级到新架构（可移除大量旧桥代码） |
| 内存 | 清理 `useEffect` 订阅与定时器、大图解码尺寸控制、避免缓存无上限、关注原生侧内存（图片/Video） |

- **白屏排查顺序**：① 日志里 bundle 是否加载成功 → ② JS 是否在首屏前抛错（可加全局错误兜底）→ ③ 是否被大量同步初始化阻塞 → ④ 首屏是否依赖网络/字体/图片。
- **卡顿定位**：先看 Perf Monitor 的 **JS 线程 vs UI 线程**耗时，谁高查谁；JS 高 → 重渲染/重计算；UI 高 → 视图层级、阴影、大图、原生绘制。

## 十二、常见坑与排查

| 现象 | 常见原因 |
| --- | --- |
| 启动白屏 | bundle 加载失败 / JS 早期异常 / 同步初始化过多 |
| 列表空白但可滚动 | 未设高度（`flex: 1` 缺失）或 `getItemLayout` 行高与实际不符 |
| 文本不显示 | 文本没包在 `<Text>` 里；或 `View` 缺少 `flex` 约束导致高度为 0 |
| `flex: 1` 不生效 | 父容器无确定高度（Android 需 `flex: 1` 链路上溯到根） |
| Android 阴影无效 | Android 用 `elevation`，iOS 才用 `shadow*` 系列 |
| 键盘遮挡输入框 | `KeyboardAvoidingView` / `react-native-keyboard-controller` |
| 状态更新不触发渲染 | 直接修改对象/数组（未返回新引用）；闭包捕获旧值（用函数式 `setState`） |
| 内存泄漏 / 警告 | `useEffect` 未清理订阅、定时器、监听器；组件卸载后 setState |
| 新架构下第三方库报错 | 该库未适配（旧 `RCTViewManager` / `UIManager.dispatchViewManagerCommand`）→ 查库的 issue 与适配版本 |
| 升级后编译失败 | 新旧架构混用 / 依赖锁版本不一致 → 先升到 0.81 开新架构跑通再往上 |

## 十三、推荐资料

- [React Native 官方文档](https://reactnative.dev/docs/getting-started)
- [新架构介绍（About the New Architecture）](https://reactnative.dev/architecture/landing-page)
- [React Native Blog（版本发布说明，了解演进最可靠）](https://reactnative.dev/blog)
- [React Native Directory（库的新架构适配状态）](https://reactnative.directory/)
- [React Navigation 文档](https://reactnavigation.org/)
- [Hermes 引擎仓库](https://github.com/facebook/hermes)
- 《React Native 跨平台移动应用开发》系列资料 / Expo 官方文档

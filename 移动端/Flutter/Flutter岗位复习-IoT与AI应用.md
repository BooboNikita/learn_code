# Flutter 岗位复习 · IoT / AI 应用方向

> 面向岗位：阶跃星辰「Flutter 开发工程师（IoT / AI 应用方向）」（全职 · 深圳南山 / 上海徐汇）。
> 这份文档不是 Flutter 教程，而是**把 JD 的每一条职责反推成面试考点**，再按优先级给你可背诵、可展开的答案骨架。
> 渲染原理的深度内容见 [Flutter 渲染树与渲染流程](README.md)，本文只做面试口径速查，不重复展开。

## 〇、先看 JD 在说什么

### 0.1 一句话画像

这个岗位**不是普通业务 App 开发**。它要做的是一条完整链路：

```
硬件设备（耳机/手表/录音笔）
   │ BLE / 私有协议
   ▼
Flutter App（设备中心、录音、播放）
   │ 音频流 + 设备事件
   ▼
云端 Agent（ASR → LLM → TTS）
   │ SSE 流式 token
   ▼
聊天 / 记忆 / 任务界面
```

所以面试官真正想确认三件事：

1. **你能把"原生 SDK + 流式数据 + 长列表"这三样拼成一个不卡的 App**（性能与架构）。
2. **你懂平台边界**：什么时候用 Flutter，什么时候必须写 Kotlin/Swift，两者怎么协作（插件与线程）。
3. **你能上双端也能上桌面**：不是"跑起来"，是"组件和接口一开始就按多端设计"。

### 0.2 JD 逐条拆解 → 考点映射

| JD 条目 | 面试官的真实考察点 | 可能的问法 | 对应章节 |
| ------- | ----------------- | ---------- | -------- |
| 设备中心 | BLE 状态机、权限矩阵、断连重连、后台存活 | 「设备连不上，你怎么逐层排查？」 | 五 |
| 录音 / 转写 / 播放 | 采集参数、音频会话、流式上传、边收边播 | 「后台录音被系统杀了怎么办？」 | 六 |
| 聊天 / 记忆 / 任务 | 流式渲染性能、长列表、本地存储与同步 | 「逐字输出怎么做到不掉帧？」 | 七 |
| 公共 App 能力 | 状态管理选型、组件库、主题、多形态布局 | 「你怎么组织一个 feature 目录？」 | 三 |
| 原生 SDK 集成 | Pigeon/Channel、线程模型、生命周期、错误传播 | 「Channel 调用把 UI 卡住了，怎么解？」 | 四 |
| 性能与质量 | 帧预算、DevTools 定位、测试、发布 | 「你怎么定位掉帧是 UI 还是 Raster？」 | 八 |
| 多端演进 | 桌面差异、平台能力抽象、依赖支持矩阵 | 「一套代码上 macOS 要改什么？」 | 九 |
| 智能穿戴（加分） | 耳机协议栈、手表配对与同步 | 「蓝牙耳机录音为什么音质差？」 | 五、六 |
| Kotlin/Swift/KMP（加分） | 自己写插件、KMP 桥接方案 | 「KMP 和 Flutter 怎么配合？」 | 九 |

### 0.3 复习优先级

| 优先级 | 模块 | 理由 |
| ------ | ---- | ---- |
| **P0** | Dart 异步与并发、状态管理与架构、平台通道/Pigeon、流式渲染 | 四条必备要求全部命中，且是「筛人题」 |
| **P1** | 音频链路、BLE/IoT、性能优化、双端发布 | 决定你是「及格」还是「对口」 |
| **P2** | 桌面端、KMP、无障碍、多语言、组件库 | 加分项，用一两个具体经验点破即可 |

### 0.4 你需要准备的三个故事

面试里技术题只占一半，另一半是「你做过什么」。这个 JD 的叙事主线应该是：
**「我做过一个把硬件数据变成实时界面的 App，并且我处理过它所有不舒服的边界。」**

准备三个 STAR 骨架（不要背稿，背关键数字）：

| 故事 | 要突出的能力 | 必备细节 |
| ---- | ------------ | -------- |
| ① 流式渲染改造 | 性能定位 + 架构取舍 | 改造前帧耗时/掉帧率 → 具体手段（节流、分段、局部刷新）→ 改造后数据 |
| ② 原生插件从 0 到 1 | 平台边界 + 线程 | 为什么必须原生做、Pigeon 契约怎么定、线程怎么切、错误怎么回传 |
| ③ 设备连接稳定性 | 状态机 + 异常兜底 | 断连率、重连策略、用户可感知的失败态设计 |

## 一、Flutter 基础速查（面试口径）

### 1.1 三棵树，一句话版本

| 树 | 一句话 | 面试时补的那半句 |
| -- | ----- | ---------------- |
| Widget | 不可变的配置描述 | 所以可以随便重建，diff 只需比 `runtimeType + key` |
| Element | 挂载实例与复用单位，也是 `BuildContext` | 所以 State 挂在它上面，Widget 重建 State 不丢 |
| RenderObject | 真正做布局与绘制 | 所以只有它才有 `size`、`constraints`、`paint()` |

> 常见误答：「每个 Widget 都会创建 RenderObject」。`Container`、`Padding`、`Theme` 这类组合/继承型 Widget 本身没有 RenderObject，最终由叶子节点（`RichText`、`ConstrainedBox`）承担。RenderObject 树比 Widget 树"瘦"。

### 1.2 一帧的四个阶段

```
Vsync → Build → Layout → Paint → Compositing  （UI 线程 / Dart isolate）
                                        │ Layer Tree
                                        ▼
                              Rasterize → GPU 合成  （Raster 线程）
```

- 布局是**单遍**的：约束向下、尺寸向上，O(n) 完成——这是 Flutter 比 Web 布局快的关键。
- 只有被 `markNeedsBuild` / `markNeedsPaint` 标记的子树参与重算。
- `RelayoutBoundary` / `RepaintBoundary` 是两级"止损线"：把变化圈在原地。

### 1.3 Flutter 的线程不止两个

很多人只记得 UI + Raster，但排查音频/设备问题必须知道这四个（外加平台线程）：

| 线程 | 跑什么 | 出问题时的典型现象 |
| ---- | ------ | ------------------ |
| **Platform（Android 主线程 / iOS main）** | 原生代码、Channel handler、平台 UI | 这里阻塞 = ANR / 卡死，且 Flutter 的卡顿面板看不出来 |
| **UI（Dart isolate）** | Dart 代码、Build/Layout/Paint | DevTools 里 `UI` 时间轴变长 |
| **Raster** | 光栅化、GPU 提交 | `Raster` 时间轴变长、GPU 相关 jank |
| **IO** | 图片解码、`dart:io` 阻塞操作 | 首次加载大图/读文件时抖动 |

> 常见误答：「Dart 单线程所以 Flutter 只有一个线程」。Dart 是**单 isolate 内单线程**，但引擎本身是多线程的；而且你可以开多个 isolate。更关键的是——**Channel 的原生 handler 跑在平台主线程**，这是 Flutter 卡顿分析里最容易被忽略的一类。

### 1.4 版本事实（2026-11，被追问时别说错）

| 项 | 现状 |
| -- | ---- |
| 稳定版 | Flutter 3.47.x（Dart 3.13.x） |
| 渲染引擎 | iOS 已是 Impeller 唯一（无法切回 Skia）；Android API 29+ 默认 Impeller，低版本回落 Skia；**3.47 起桌面端也默认 Impeller** |
| iOS 最低版本 | 3.47 起为 iOS 15 |
| Android 模板 | targetSdk 36 / minSdk 24 |
| iOS 依赖管理 | 3.44 起 Swift Package Manager 成为默认，CocoaPods 进入维护模式 |
| 代码生成 | Dart macros 已停止推进，`build_runner` 仍是官方路线 |

## 二、Dart 异步与并发（P0）

### 2.1 事件循环：两个队列，微任务优先

一个 isolate 一个事件循环，队列分两级：

```
当前同步代码执行完
   │
   ▼
① Microtask Queue（微任务）  ← 全部清空，期间新产生的继续插队
   │
   ▼
② Event Queue（事件：IO、Timer、用户输入、端口消息）
   │
   └──▶ 取一个事件 → 执行完 → 再清空微任务 → 循环
```

| 归属 | 内容 |
| ---- | ---- |
| 微任务 | `Future` 的 `then/catchError` 回调、`scheduleMicrotask`、`await` 之后的续体 |
| 事件 | `Timer`、`Future.delayed`、`Stream` 事件、`ReceivePort` 消息、平台消息 |

```dart
void main() {
  print('A');
  Future(() => print('B'));            // 事件队列
  Future.microtask(() => print('C'));  // 微任务队列
  scheduleMicrotask(() => print('D')); // 微任务队列
  print('E');
}
// 输出：A E C D B
```

> 常见误答：「`await` 会阻塞线程」。`await` 只是把后续代码注册成续体，线程立刻返回事件循环去干别的。真正阻塞线程的是 `sleep()`、密集 CPU 计算、同步 IO。

**实践含义**：微任务不能饿死事件队列。如果在一个微任务里无限 `scheduleMicrotask`，UI 会完全不响应——这比慢更可怕。

### 2.2 Future：一次性的异步结果

```dart
Future<T> 的三态：pending → completed(value) / completed(error)
```

几个必须说清的点：

| 现象 | 原因 |
| ---- | ---- |
| `then` 里抛异常，跳到 `catchError` | 异常沿 Future 链传播，不会变成未捕获异常 |
| `try/catch` 包不住没 `await` 的 Future | 没有 `await` 就没有把它接进当前同步控制流 |
| `Future.wait` 一个失败默认整体失败 | 需要 `eagerError: false` 拿全部结果，或逐个 `catchError` |
| `Future.delayed` 在页面销毁后仍会执行 | 定时器不认 Widget 生命周期，必须自己判断 `mounted` |

**取消问题**：Dart 的 `Future` **没有取消语义**。要取消只能：
- 用 `CancelableOperation`（`async` 包）做逻辑取消；
- 或让底层请求支持取消（`dio.CancelToken`、`HttpClientRequest.abort`、`StreamSubscription.cancel`）；
- 服务端侧同步中断（LLM 流式请求尤其重要，不然白烧 token）。

### 2.3 Stream：流式数据的主战场

| 维度 | 单订阅 Stream | 广播 Stream |
| ---- | ------------- | ----------- |
| 监听者 | 只能一个 | 可多个 |
| 缓存 | 监听前的数据会缓冲（取决于源） | 不缓冲，后来者错过就没了 |
| 典型来源 | 文件读、HTTP 响应、`StreamController()` 默认 | `onEvent` 类 UI 事件、`StreamController.broadcast()` |
| 二次监听 | 抛异常 | 允许 |

**背压（backpressure）**——这是 JD 里"流式事件"的核心：

```dart
// ✅ await for：天然背压，处理完一条才取下一件
await for (final chunk in response.stream) {
  await persist(chunk);
}

// ⚠️ listen：无背压，事件照发，消费慢了就在内存里堆
subscription = stream.listen(handle);

// ✅ 手动背压：处理不过来就暂停源
subscription.onData((data) async {
  subscription.pause();
  await handle(data);
  subscription.resume();
});
```

> 常见误答：「Stream 会自动帮你限流」。`listen` 不会。高频事件（麦克风 PCM、BLE notify）必须显式做**节流 / 合并 / 丢帧**策略，否则内存与 UI 双双爆炸。

**StreamController 的坑**：

| 坑 | 说明 |
| -- | ---- |
| `sync: true` | 事件同步派发，`add` 时立刻回调。若回调里又 `add`，重入导致栈溢出/顺序错乱。只在明确需要同步语义时用 |
| 不 `close()` | 监听者永远等不到 done，上游资源（原生监听、socket）不会释放 |
| 不 `cancel()` | 页面退出后回调仍在执行 → `setState after dispose`、原生侧监听泄漏 |
| `addError` 后 | `cancelOnError: true` 会取消订阅；默认流不自动结束，取决于源实现 |

### 2.4 Isolate：真并行，以及什么时候该用

| 维度 | 说明 |
| ---- | ---- |
| 内存 | 各自独立堆，**没有共享内存**，靠消息传递（拷贝）通信 |
| 通信 | `SendPort` / `ReceivePort`；Dart 2.15+ 支持 `Isolate.exit(port, obj)` 零拷贝转移所有权 |
| 可发送 | 基本类型、`List`/`Map`（元素也要可发送）、`SendPort`、`TransferableTypedData`；**闭包捕获的对象不行** |
| 一次性任务 | `Isolate.run(fn)`（Dart 2.19+）最轻量；`compute(fn, arg)` 是 Flutter 侧的等价封装 |

**这个岗位真正会用到的场景**：

| 场景 | 为什么必须丢到 isolate |
| ---- | ---------------------- |
| 音频编码（PCM → Opus/AAC） | 每 20ms 一帧的持续计算，放主 isolate 必然掉帧 |
| 大文件加解密 / 校验 | CRC、AES 都是纯 CPU 密集 |
| 长 Markdown/JSON 解析 | 首屏大文档解析会卡 200ms+ |
| 图片处理（缩略图、滤镜） | 解码本身已由引擎在 IO 线程做，但像素级处理仍需 isolate |

> 关键细节：**后台 isolate 默认不能直接用平台通道**。Dart 3.7+ 起可用 `BackgroundIsolateBinaryMessenger`（配合 `RootIsolateToken`）在后台 isolate 调 Channel；否则只能在主 isolate 收发，用端口转交。

```dart
// 后台 isolate 调平台通道的正确姿势
final token = RootIsolateToken.instance!;
Isolate.run(() async {
  BackgroundIsolateBinaryMessenger.ensureInitialized(token);
  await channel.invokeMethod('ping');
});
```

### 2.5 生命周期与"异步回调落地"的三条铁律

这三条是写业务代码的及格线，面试常以"你踩过什么坑"的形式问：

1. **看 `mounted`**：`await` 之后操作 `State` 前必须 `if (!mounted) return;`。
2. **`dispose` 里全部释放**：`AnimationController`、`TextEditingController`、`ScrollController`、`StreamSubscription`、`Timer`、`FocusNode`、原生监听回调。
3. **取消而非忽略**：页面退出时不只是"不处理结果"，而是**主动取消请求**（省流量、省 token、省服务端算力）。

```dart
@override
void dispose() {
  _sub?.cancel();
  _timer?.cancel();
  _controller.dispose();
  _cancelToken.cancel('page disposed'); // 请求侧也要断
  super.dispose();
}
```

## 三、状态管理与架构（P0）

### 3.1 先分清两种状态，再谈选型

| 类型 | 定义 | 存放位置 | 例子 |
| ---- | ---- | -------- | ---- |
| **Ephemeral（局部/短命）** | 只属于一个 Widget，重建即丢也无所谓 | `State` + `setState` | 输入框焦点、动画进度、展开/收起 |
| **App State（全局/长命）** | 跨页面、跨会话、需持久化或与原生共享 | 状态管理容器 | 登录态、设备连接状态、当前会话、录音状态 |

> 常见误答：「一切都上状态管理」。把动画进度、滚动位置塞进全局 store，是最典型的过度设计——每次滚动都触发全局监听，性能直接崩。

### 3.2 主流方案对比（要能说出取舍，不是背 API）

| 维度 | Provider | Riverpod | Bloc | GetX |
| ---- | -------- | -------- | ---- | ---- |
| 本质 | `InheritedWidget` 封装 + `ChangeNotifier` | 编译期安全的依赖注入 + 状态容器 | 事件驱动的单向数据流（Event → State） | 全家桶（状态/路由/依赖/弹窗） |
| 依赖 `BuildContext` | 是 | **否**（可在纯 Dart 层用） | 否（`BlocProvider` 需要） | 部分需要 |
| 类型安全 | 运行时才暴露问题 | 编译期（Provider 参数化） | 强（State 是密封类） | 弱 |
| 异步表达 | 手写 loading/error | `AsyncValue` 三态内建 | 状态机显式流转 | 手写 |
| 样板代码 | 低 | 低 | 高 | 最低 |
| 可测试性 | 中 | 高（可 override 任意依赖） | 最高（纯函数 reducer） | 低（全局单例） |
| 适合场景 | 中小项目、渐进迁移 | 中大型、异步重、需要细粒度重建 | 复杂业务流、多人协作、强审计 | 快速原型 |

**给这个岗位的选型口径**（面试可直接这么说）：

> 设备连接、录音、Agent 会话这三类状态都是**长生命周期 + 强异步 + 需要跨页面共享**，我会倾向 Riverpod 或 Bloc。理由不是"流行"，而是：
> ① 设备连接是个状态机（未绑定/连接中/已连接/同步中/失效），Bloc 的显式 Event→State 让状态迁移可穷举、可测试；
> ② 录音 + 流式对话有大量 loading/error/partial 状态，Riverpod 的 `AsyncValue` 直接把三态编码进类型，UI 侧只需 `switch` 三种分支，不会漏掉错误态；
> ③ 两者都能脱离 `BuildContext` 单测，这对插件层联调很重要。

### 3.3 Feature-first 的目录结构

JD 说"具备可维护的业务模块设计能力"，落地就是目录：

```
lib/
├── app/                    # 应用装配：路由、主题、启动流程
│   ├── router/
│   ├── theme/              # 主题与设计 token
│   └── bootstrap.dart
├── core/                   # 跨 feature 的基建
│   ├── network/            # dio 封装、SSE 客户端、重试与拦截器
│   ├── platform/           # 对 Pigeon 生成代码的封装（唯一出口）
│   ├── storage/            # 本地库、缓存策略
│   ├── error/              # 统一错误模型
│   └── utils/
├── features/
│   ├── device/             # 设备中心
│   │   ├── data/           # repository 实现、DTO、本地/远端数据源
│   │   ├── domain/         # 实体、用例、状态机定义
│   │   └── presentation/   # 页面、widget、状态容器
│   ├── recording/          # 录音/转写/播放
│   ├── conversation/       # 聊天/记忆/任务
│   └── account/            # 账号与设置
└── shared/                 # 跨 feature 复用的 UI 组件（设计系统）
```

**三条硬规则**：

1. `presentation` 不许直接 import Pigeon 生成的 `.g.dart`——只依赖 `core/platform` 暴露的接口。这样 mock 与桌面端（无原生实现）替换才可行。
2. `data` 层返回 `domain` 实体，不把 DTO（Pigeon 生成类 / JSON map）泄漏到 UI。
3. 跨 feature 复用一律下沉到 `shared/`，禁止 `features/a` 反向 import `features/b`。

### 3.4 组件库与主题：为"多设备形态 + 产品差异配置"设计

JD 第 2 条要求"支持多设备形态和产品差异配置"，这在工程上对应两件事：

**① 形态适配（Form Factor）**，不要靠 `if (Platform.isAndroid)` 散落各处：

```dart
// 断点驱动的布局，而不是设备类型驱动
enum WindowSize { compact, medium, expanded }

WindowSize sizeOf(BuildContext context) {
  final w = MediaQuery.sizeOf(context).width;
  if (w < 600) return WindowSize.compact;    // 手机
  if (w < 1024) return WindowSize.medium;    // 折叠屏展开 / 平板竖屏
  return WindowSize.expanded;                // 平板横屏 / 桌面窗口
}
```

- 手机：单栏 + 底部导航；平板/桌面：双栏（列表 + 详情）+ 侧边导航。
- 内容用 `ConstrainedBox(maxWidth: 720)` 限制可读宽度，桌面端不会被拉成一条横线。

**② 产品差异配置（Flavor / 白标）**：

```dart
// 编译期注入 + 运行期可读，避免 if 满天飞
abstract class AppConfig {
  String get appName;
  bool get enableRecording;
  ThemeData get theme;
  Set<DeviceType> get supportedDevices;
}

// main_dev.dart / main_pro.dart 各自 bootstrap(AppConfig...)
```

**设计 token 化**：颜色、圆角、间距、字号收敛到 `AppTokens` + `ThemeExtension`，不要散落魔法数字。这样"换一套品牌色"是改一个文件，不是全局搜索替换。

## 四、原生插件与平台通道（P0，JD 明确点名）

### 4.1 四种通道，先分清职责

| 通道 | 方向 | 编解码 | 典型用途 | 限制 |
| ---- | ---- | ------ | -------- | ---- |
| `MethodChannel` | Dart → 原生（请求/响应） | `StandardMethodCodec` | 单次命令：开始录音、读设备信息 | 弱类型，方法名拼错运行时才炸 |
| `EventChannel` | 原生 → Dart（单向流） | `StandardMethodCodec` | 持续事件：BLE 通知、录音电平、下载进度 | 只能一个监听者语义 |
| `BasicMessageChannel` | 双向任意消息 | 自定义（`StringCodec`/`JSONMessageCodec`/`BinaryCodec`） | 自定义协议、需要自有序列化 | 全手写，无类型 |
| **Pigeon** | 双向，代码生成 | `StandardMessageCodec` | **官方推荐**：所有自研插件 | 生成代码非稳定 API，需版本 lockstep |

> 常见误答：「Pigeon 比 MethodChannel 快」。**性能等价**——Pigeon 内部就是同一种 Channel + 同一种 Codec。它买到的是**类型安全与编译期校验**，不是更低延迟。真要说低延迟，那是 `dart:ffi`（同线程同步调用、零序列化）。

### 4.2 Pigeon：契约驱动，六个必须知道的注解

```dart
// pigeons/device_api.dart —— 只放契约，不放实现，不进 lib 的公开 API
import 'package:pigeon/pigeon.dart';

class DeviceInfo {
  DeviceInfo({required this.id, required this.name, required this.battery});
  final String id;
  final String name;
  final int battery;
}

@HostApi()                                  // 原生实现、Dart 调用
abstract class DeviceHostApi {
  DeviceInfo getDeviceInfo(String id);      // 同步：立即返回，别做重活
  @async Future<void> startRecording(RecordingConfig cfg);  // 异步：原生做完回调
}

@FlutterApi()                               // Dart 实现、原生调用
abstract class DeviceEvents {
  void onConnectionChanged(String id, int state);  // 必须回主线程调
}

@EventChannelApi()                          // 原生持续推事件（Pigeon 22.7+）
abstract class RecordingStreamApi {
  Stream<AudioFrame> onAudioFrame();
}
```

| 注解 | 谁实现 | 用途 |
| ---- | ------ | ---- |
| `@HostApi` | 原生 | Dart 调原生，最常用 |
| `@FlutterApi` | Dart | 原生回调 Dart（连接状态变化、播放完成） |
| `@async` | — | 标记"晚点完成"，生成 callback 版本；**I/O 方法一律加** |
| `@TaskQueue` | — | 让 handler 跑在后台队列（`TaskQueueType.serialBackgroundThread`），省去手动切换，但要保证实现线程安全 |
| `@EventChannelApi` | 原生 | 生成强类型 `Stream`，替代手写 EventChannel |
| `@ConfigurePigeon` | — | 指定各端输出路径与包名 |

**工程约束（被追问"Pigeon 有什么坑"时答这些）**：

- 各端生成代码必须来自**同一 Pigeon 版本**，否则"未定义行为，可能直接崩溃"。
- 生成代码**不是稳定 API**，禁止暴露在包的公开接口里，也不要跨包拆分。
- 契约文件只能声明、不能写逻辑（它是 schema，不是可执行代码）。
- 单测价值：`@HostApi` 生成的是普通 Dart 类，可用 `mocktail` 直接 mock，不需要起真机。

### 4.3 线程模型：这个岗位最容易出事故的地方

```
Dart 侧调用
   │  （在 UI isolate 里发起）
   ▼
跨到 Platform Thread（Android 主线程 / iOS main）执行 handler
   │
   ├─ 轻活：直接 return → 回到 Dart
   └─ 重活：切后台（协程/队列）→ 做完必须回主线程再回调
                                              │
                                              ▼
                                       回传 Dart（UI isolate）
```

**四条铁律**：

| 规则 | 违反的后果 |
| ---- | ---------- |
| 原生 handler 默认在**平台主线程**，不许做重活 | 界面卡死、ANR，且 Flutter DevTools 里看不到 |
| 切后台做完后，**必须回主线程**再调 result / callback | 偶发崩溃、数据竞争、难以复现 |
| 原生调 `@FlutterApi` 也必须在主线程 | 同上，且几乎无法稳定复现 |
| 原生异常必须转成错误码返回，不许逃逸 | Android 上未捕获异常会**直接崩整个 App** |

```kotlin
// Android：重活切走，回调回主线程
override fun startRecording(cfg: RecordingConfig, callback: (Result<Unit>) -> Unit) {
    lifecycleScope.launch {
        val ok = withContext(Dispatchers.IO) { recorder.start(cfg) }  // 重活在 IO
        callback(if (ok) Result.success(Unit)
                 else Result.failure(FlutterError("E_REC", "启动录音失败", null)))  // 回主线程
    }
}
```

```swift
// iOS：同样的纪律
func startRecording(cfg: RecordingConfig, completion: @escaping (Result<Void, Error>) -> Void) {
    DispatchQueue.global(qos: .userInitiated).async {
        do {
            try self.recorder.start(cfg)
            DispatchQueue.main.async { completion(.success(())) }   // 必须回主
        } catch {
            DispatchQueue.main.async { completion(.failure(error)) }
        }
    }
}
```

### 4.4 生命周期：三端各自的挂载点

| 平台 | 挂载点 | 注意 |
| ---- | ------ | ---- |
| Android | `FlutterActivity.configureFlutterEngine` / `FlutterFragment` | 用缓存的 engine 时，插件实例会被复用，**状态要能重置** |
| iOS | `AppDelegate.didFinishLaunching` → `GeneratedPluginRegistrant` | `FlutterViewController` 销毁 ≠ 引擎销毁（add-to-app） |
| 插件包 | `Plugin.registerWith` / `registerWithRegistrar` | 保存 `BinaryMessenger` 而不是 Activity，防泄漏 |

**必答的坑**：

- **Activity 重建**（旋转屏幕、深色模式切换）：原生侧持有 `Activity` 引用 → `IllegalStateException` 或内存泄漏。正确做法是持 `Context` / `BinaryMessenger`，需要 Activity 时用 `ActivityAware` 回调拿到最新实例。
- **EventSink 悬空**：Dart 取消订阅后原生仍在推事件 → 崩溃或泄漏。必须在 `onCancel` 里注销原生监听器。
- **引擎复用**：add-to-app 场景一个进程多引擎，每个引擎有独立 `BinaryMessenger`，**不要用静态单例存 channel**。

```kotlin
// EventChannel 的正确姿势：Dart 侧取消 → 原生必须停
private var sink: EventChannel.EventSink? = null
private val listener = object : BleListener {
    override fun onNotify(data: ByteArray) {
        mainHandler.post { sink?.success(data) }   // 回主线程再推
    }
}

override fun onListen(args: Any?, events: EventChannel.EventSink?) {
    sink = events
    ble.addListener(listener)      // 订阅开始
}

override fun onCancel(args: Any?) {
    ble.removeListener(listener)   // 必须真的停，否则泄漏 + 空推
    sink = null
}
```

### 4.5 错误如何跨边界

| 层 | 做法 |
| -- | ---- |
| 原生 | 用**稳定错误码 + 可读信息 + 可选 details**；不要抛裸异常 |
| 通道 | `FlutterError(code, message, details)` → Dart 侧收到 `PlatformException` |
| Dart 服务层 | 捕获 `PlatformException`，映射成**领域错误**（`DeviceUnavailable`、`PermissionDenied`） |
| 缺失实现 | `MissingPluginException` → 表示"该平台没实现"，桌面端/Web 尤其常见，要能优雅降级 |

> 记住一句：**把通道当网络边界对待**。它会展平成字节、会失败、会超时、会对端版本不一致。所以要在服务层做超时、重试、错误翻译，而不是在 Widget 里 `try/catch`。

### 4.6 什么时候必须写原生（而不是硬用 Flutter 绕）

| 场景 | 结论 |
| ---- | ---- |
| 设备/音频/传感器 SDK 只有原生实现 | 必须写插件（这个岗位的高频理由） |
| 低延迟音频（<20ms） | 原生 + Oboe/AAudio（Android）、AVAudioEngine（iOS） |
| 后台常驻（录音、BLE 长连接） | 必须原生前台服务 / 后台模式，Flutter 层管不了 |
| 系统级能力（通知、快捷指令、桌面小组件） | 原生 |
| 复杂动画/自定义绘制 | Flutter 强项，不必上原生 |
| 嵌入式地图/WebView/相机预览 | `PlatformView`，注意合成开销 |

> 关于 `PlatformView`：它把原生视图嵌进 Flutter 合成树，会产生额外的纹理拷贝与同步开销。**列表里批量嵌 PlatformView 是性能灾难**，改用纹理/截图方案或减少数量。

## 五、IoT 设备侧：BLE 与设备中心（P1，加分项主战场）

### 5.1 BLE 最小知识图

```
广播（Advertising）  ← 设备主动喊：我是谁、我支持什么
   │  扫描（Scan）
   ▼
连接（Connect） → 服务发现（discoverServices）
   │
   ├─ Service（服务，一个功能集合，16 位/128 位 UUID）
   │    └─ Characteristic（特征，读写的最小单位）
   │         ├─ read     读当前值
   │         ├─ write    带响应写（可靠、慢）
   │         ├─ writeNoResponse 无响应写（快、可能丢）
   │         └─ notify/indicate  订阅推送（需写 CCCD 描述符开启）
   └─ Descriptor（描述符，CCCD 用来开/关通知）
```

**必须能说清的数字**：

| 项 | 值 | 说明 |
| -- | -- | ---- |
| 默认 MTU | 23 | 有效载荷 = 23 − 3 = **20 字节**，所以默认要分包 |
| 协商 MTU | Android 可请求至 517；iOS 自动约 185 | 有效载荷 = MTU − 3 |
| 连接间隔 | 7.5ms ~ 4s（1.25ms 步进） | 越小越实时、越耗电 |
| 从机延迟（latency） | 0 ~ 499 | 允许从机跳过 N 个连接事件，省电 |
| 监督超时 | 100ms ~ 32s | 超时判定断连（10ms 步进） |

### 5.2 权限矩阵（写错就是"设备搜不到"）

| 平台 / 版本 | 必需权限 | 备注 |
| ----------- | -------- | ---- |
| Android 12+（API 31+） | `BLUETOOTH_SCAN`、`BLUETOOTH_CONNECT`（如需广播再加 `BLUETOOTH_ADVERTISE`） | 扫描加 `android:usesPermissionFlags="neverForLocation"` 可免定位权限 |
| Android ≤ 11 | `BLUETOOTH`、`BLUETOOTH_ADMIN`、`ACCESS_FINE_LOCATION` | 系统强制要求定位权限才能扫描 |
| Android 通用 | `usesFeature android.hardware.bluetooth_le`（`required=false`） | 否则无 BLE 设备装不上 |
| iOS | `NSBluetoothAlwaysUsageDescription` | 缺了直接崩，不是权限拒绝 |
| iOS 后台 | `UIBackgroundModes: bluetooth-central` | 后台扫描/保持连接必需 |
| iOS 本地网络（部分设备） | `NSLocalNetworkUsageDescription` + `NSBonjourServices` | Wi-Fi 配网类设备需要 |

> 常见误答：「Android 12 之后不用定位权限了，所以不用管定位」。准确说法是：**扫描本身不再需要定位权限**（声明 `neverForLocation` 时），但如果你要通过 BLE 推断地理位置，仍需定位权限与声明。

### 5.3 连接状态机（面试重点：不要说"连上/没连上"两态）

```
        ┌──────────┐  绑定      ┌──────────┐  连接成功   ┌──────────┐
        │ Unbound  │──────────▶│Connecting│───────────▶│ Connected│
        └──────────┘            └────┬─────┘            └────┬─────┘
                                     │ 超时/失败             │ 服务发现
                                     ▼                       ▼
                                ┌────────┐             ┌──────────┐
                                │ Failed │◀────────────│ Syncing  │
                                └────────┘   同步失败   └────┬─────┘
                                     ▲                       │ 同步完成
                                     │ 断连                  ▼
                                ┌────┴─────┐            ┌──────────┐
                                │Disconnected│◀─────────│  Ready   │
                                └──────────┘   链路断   └──────────┘
```

**每个状态必须有对应的 UI**——这正是 JD 里"完善权限、进度、离线及失败状态展示"的含义：

| 状态 | 用户看到什么 | 可执行操作 |
| ---- | ------------ | ---------- |
| Unbound | 空态 + 引导「添加设备」 | 扫码/搜索 |
| Connecting | 骨架 + 进度（已耗时） | 取消 |
| Failed | 具体原因（超时/配对失败/被占用）+ 排查建议 | 重试 |
| Syncing | 进度条（已同步 N/M 条录音） | 后台继续 |
| Ready | 电量、固件版本、最后同步时间 | 全部功能 |
| Disconnected | 顶部横幅「设备已断开，正在重连(2/5)」 | 手动重连 |

### 5.4 断连与重连策略

| 策略 | 说明 | 坑 |
| ---- | ---- | -- |
| 指数退避重连 | 1s → 2s → 4s → 8s，上限 30s，加 ±20% 抖动 | 无抖动会出现"惊群"，一批设备同时重连打爆服务端 |
| Android `autoConnect=true` | 系统级自动重连，设备回范围就恢复 | 首次连接慢（可能 10s+），不适合"点了就要连"的场景 |
| iOS 无 autoConnect | 必须自己 `connect` 重试 | 后台被挂起时重连会停，靠 `bluetooth-central` 后台模式 + 状态恢复 |
| 连接保活 | 定期读一个特征值 / 心跳包 | 太频繁耗电，太稀疏被系统判空闲断开 |
| 多设备并发 | 串行连接（Android 同时连接数有限） | 并行连接大量失败，需队列 |

**状态恢复（iOS 特有）**：`CBCentralManagerOptionRestoreIdentifierKey` + `willRestoreState`，让 App 被系统杀死后由系统在蓝牙事件时拉起。JD 里"设备中心"要做好，这是绕不开的点。

### 5.5 私有协议：分包、粘包、校验

设备通信不是 HTTP，没有"请求-响应"的天然边界。要给面试官讲清你的分帧方案：

```
┌──────┬────────┬──────┬─────────┬────────┬──────┐
│ MAGIC│ VERSION│ CMD  │  LEN    │ PAYLOAD│ CRC  │
│ 2B   │  1B    │ 1B   │  2B     │  N B   │ 2B   │
└──────┴────────┴──────┴─────────┴────────┴──────┘
```

| 问题 | 方案 |
| ---- | ---- |
| 单包超 MTU | 按 `MTU − 3` 切片，序号 + 总片数，接收端组包 |
| 粘包/半包 | 靠 `LEN` 字段定长切分，维护接收缓冲区，不足则等下一包 |
| 丢包 | CRC 校验失败 → 请求重传该序号；或整帧重发 |
| 版本兼容 | 帧头带 `VERSION`，新固件加字段时旧 App 能跳过未知 TLV |
| 命令超时 | 每条命令带 `seq` + 超时定时器，超时按策略重试（**幂等命令才可重试**） |
| 并发命令 | 单通道串行队列，避免设备端状态错乱 |

**幂等性设计**：`开始录音`、`删除文件` 这类命令要带唯一 `requestId`，设备侧去重。否则重试会导致重复动作——这是 IoT 端最典型的线上事故。

### 5.6 OTA 固件升级（被问到就是加分）

| 环节 | 要点 |
| ---- | ---- |
| 包传输 | 分片（MTU 对齐）、每片 CRC、断点续传（记录已确认偏移） |
| 校验 | 整包 SHA-256 + 版本号 + 机型匹配，防止刷错固件 |
| 写入 | 双分区 A/B：新固件写备用区，校验通过再切换 |
| 回滚 | 启动失败自动回滚到旧分区（设备侧 bootloader 实现） |
| 传输期保活 | 升级中禁止息屏/断连；App 侧保前台 + 心跳 |
| 用户体验 | 显示进度、剩余时间、失败原因；**升级中强提示"请勿关闭 App"** |
| 失败恢复 | 记录升级状态，下次连接时续传（用户不会重来一遍） |

### 5.7 智能穿戴加分项：耳机与手表

| 场景 | 关键技术点 |
| ---- | ---------- |
| 蓝牙耳机（经典蓝牙） | A2DP 只负责**播放**（高质量单向）；录音走 **HFP/HSP**（窄带 8k/16k，音质差）——这就是"戴耳机录音声音闷"的根因 |
| 蓝牙耳机（BLE / LE Audio） | LE Audio + LC3 编码，支持双向高质量；需要 Android 13+/iOS 16+ 与硬件支持 |
| 耳机状态读取 | Android：`BluetoothHeadset`、`AudioManager.getDevices()`；iOS：`AVAudioSession.currentRoute`、`AVAudioSession.availableInputs` |
| 手表配对与同步 | 手表一般走 BLE GATT 私有服务；同步大批量数据要限速 + 断点（手表的写入速率很低，狂写会触发流控） |
| 通知转发 | Android `NotificationListenerService`；iOS 无法读取其他 App 通知（系统限制）——面试时这是很好的"知道边界"的证明 |
| 电量与佩戴检测 | 通常靠 GATT 特征 notify 推送，需要处理"设备不在范围"导致的长时间无数据 |

## 六、音频链路（P1）

### 6.1 采集：参数决定后面一切

| 参数 | 语音场景常用值 | 说明 |
| ---- | -------------- | ---- |
| 采样率 | 16 kHz | ASR 标准；音乐用 44.1/48 kHz |
| 位深 | 16 bit | PCM 每样本 2 字节 |
| 声道 | 单声道 | 双声道对语音识别无益，翻倍流量 |
| 帧长 | 20 ms | 16kHz × 20ms = 320 样本 = 640 字节，编码器友好 |
| 码率（PCM） | 16000 × 16 × 1 = **256 kbps** | 所以必须编码后再传 |
| 编码 | Opus（语音首选）/ AAC / AMR-NB | Opus 在 16–32 kbps 即可保持可懂度 |

**平台 API 对照**：

| 能力 | Android | iOS |
| ---- | ------- | --- |
| 采集 PCM | `AudioRecord` | `AVAudioEngine` + `installTap` |
| 采集到文件 | `MediaRecorder` | `AVAudioRecorder` |
| 低延迟 | **Oboe**（AAudio / OpenSL ES） | `AVAudioEngine`（本身低延迟） |
| 播放 | `AudioTrack` / `MediaPlayer` / ExoPlayer | `AVAudioPlayer` / `AVAudioEngine` |
| 会话/路由 | `AudioManager`、`AudioFocus` | `AVAudioSession` |

### 6.2 iOS 音频会话：必背的 category/options

```swift
let session = AVAudioSession.sharedInstance()
try session.setCategory(
    .playAndRecord,                       // 同时录放（对讲/对话场景必须）
    mode: .voiceChat,                     // 语音优化：回声消除 + 降噪
    options: [.allowBluetoothHFP,         // 允许蓝牙耳机作为输入（关键！）
              .defaultToSpeaker,          // 默认外放，否则走听筒
              .allowBluetoothA2DP]        // 允许蓝牙高音质输出
)
try session.setActive(true, options: [])
```

| 想实现 | 配置 |
| ------ | ---- |
| 蓝牙耳机录音 | 必须 `.allowBluetoothHFP`（新版 SDK 也叫 `.allowBluetooth`），否则输入仍走手机麦克风 |
| 录音时外放 | `.defaultToSpeaker` |
| 与其他 App 混音 | `.mixWithOthers`（会失去部分独占优化） |
| 后台持续录音 | `UIBackgroundModes: audio` + session 保持 active |
| 处理来电打断 | 监听 `AVAudioSession.interruptionNotification`，`.began` 暂停、`.ended` 判断 `shouldResume` |

> 常见误答：「iOS 录音只要申请麦克风权限就行」。权限只决定"能不能录"，**路由与会话**决定"录到什么、会不会被静音、切后台还活不活"。这两件事常在同一次故障里同时出现。

### 6.3 Android 后台录音：Android 14 起的硬性要求

```xml
<uses-permission android:name="android.permission.RECORD_AUDIO"/>
<uses-permission android:name="android.permission.FOREGROUND_SERVICE"/>
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_MICROPHONE"/>

<service
    android:name=".RecordingService"
    android:foregroundServiceType="microphone"   <!-- 必写 -->
    android:exported="false"/>
```

| 版本 | 约束 |
| ---- | ---- |
| Android 9+ | 后台不能直接访问麦克风，必须前台服务 |
| Android 11+ | 前台服务启动时机受限（后台启动会抛异常） |
| Android 12+ | 禁止从后台启动前台服务（有豁免场景） |
| Android 13+ | `POST_NOTIFICATIONS` 运行时权限，否则前台服务通知不显示 |
| Android 14+ | **必须**声明 `foregroundServiceType="microphone"` 并申请对应权限，否则崩溃 |

**省电与厂商 ROM**：国内厂商（小米/华为/OPPO/vivo）会主动杀后台。工程手段：引导用户加白名单、用前台服务 + 常驻通知、避免长连接无心跳。这是"录音内容"类产品必须回答的现实问题。

### 6.4 流式上传：边录边传

```
麦克风 → PCM 帧(20ms) → 编码(Opus/AAC) → 分片缓冲(200~500ms)
   │                                          │
   │                                          ▼
   │                                   WebSocket 双工 / HTTP chunked
   │                                          │
   └──────────── 本地同时落盘（断网兜底）──────┘
```

| 设计点 | 做法 |
| ------ | ---- |
| 传输协议 | WebSocket 双向最灵活（可边发边收 ASR 结果）；HTTP chunked 只能单向上行 |
| 断网兜底 | **本地一定先落盘**，网络恢复后补传，带 `offset` 续传 |
| 重连 | 断线后从最后确认的字节偏移续传，服务端要支持幂等 |
| 背压 | 上行速度 > 网络带宽时，缓冲会无限增长 → 设上限，超限丢最旧或降码率 |
| 结束标记 | 显式 `end` 帧，否则服务端不知道你说完了（VAD 只做辅助） |
| 静音检测 | 本地 VAD 省流量；但不要用它替代 end 帧，误判会截断语音 |

### 6.5 播放：流式 TTS 边收边播

| 问题 | 方案 |
| ---- | ---- |
| TTS 分片到达不连续 | 维护 jitter buffer（预缓冲 200–500ms 再起播） |
| 播放卡顿 | 双缓冲队列 + 预解码下一片 |
| 打断（barge-in） | 立即 `stop()` + 清空缓冲队列 + 取消未到达的 TTS 请求 |
| 与录音共存 | iOS 用 `.playAndRecord`；Android 处理好 `AudioFocus`，否则 TTS 播报时录音被静音 |
| 播放进度/波形 | 用 `AnimationController` 或定时器驱动，**不要每帧 setState** |

### 6.6 波形与可视化性能

| 做法 | 评价 |
| ---- | ---- |
| 每帧重绘整条波形路径 | ❌ 点数上千时必卡 |
| 降采样到固定点数（如 200 点）+ `CustomPainter` | ✅ 推荐 |
| `RepaintBoundary` 包住波形区 | ✅ 避免波及整页 |
| 只画可见区（滚动场景） | ✅ 长录音必需 |
| 用 `ui.Picture` 缓存已绘制部分 | ✅ 极长录音可用 |
| 每帧 `setState` 传新数组 | ❌ 改为 `ValueNotifier<List<double>>` + `ValueListenableBuilder` |

## 七、AI 对话与流式渲染（P0，这个岗位的核心差异点）

> 这一章是"你和普通 Flutter 候选人拉开差距"的地方。JD 里"优化流式文本、长内容/列表"针对性极强。

### 7.1 流式协议：SSE 的解析没那么简单

```
HTTP/1.1 200 OK
Content-Type: text/event-stream

data: {"delta":"你"}

data: {"delta":"好"}

data: [DONE]

```

**三个必须处理的现实问题**：

| 问题 | 原因 | 处理 |
| ---- | ---- | ---- |
| 半包／粘包 | TCP 流没有消息边界，一次 `read` 可能拿到半个事件或多个事件 | 维护字符串缓冲，按 `\n\n` 切事件，残余留到下次 |
| UTF-8 多字节被切断 | 一个汉字 3 字节，可能被切在两个 chunk 之间 | 用 `utf8.decoder`（流式）而非 `utf8.decode(bytes)`；或用 `Utf8Decoder(allowMalformed: true)` 兜底 |
| 长连接中断 | 网络切换、服务端超时 | 心跳 + 断线重连 + 从最后 `messageId` 续接（服务端需支持） |

```dart
// 正确的 SSE 流式解析骨架
final request = http.Request('POST', uri)..body = jsonEncode(payload);
final response = await client.send(request);

final buffer = StringBuffer();
await for (final chunk in response.stream.transform(utf8.decoder)) {
  buffer.write(chunk);
  var text = buffer.toString();

  while (true) {
    final idx = text.indexOf('\n\n');       // 事件分隔符
    if (idx < 0) break;
    final rawEvent = text.substring(0, idx);
    text = text.substring(idx + 2);
    buffer
      ..clear()
      ..write(text);                        // 残余回填，等下一 chunk

    for (final line in rawEvent.split('\n')) {
      if (!line.startsWith('data:')) continue;
      final data = line.substring(5).trim();
      if (data == '[DONE]') return;
      onDelta(data);
    }
  }
}
```

> 常见误答：「用 `response.body` 就能拿到流式内容」。`response.body` 会**等整个响应结束**再返回，流式效果直接消失。必须走 `send()` + `response.stream`。

### 7.2 逐字渲染：为什么"每来一个 token 就 setState"会卡

一次回复 800 token = 800 次重建。如果整页重建，每次都要 build 整个消息列表 + Markdown 解析，16ms 预算瞬间击穿。

**四层优化，按顺序做**：

| 层级 | 手段 | 效果 |
| ---- | ---- | ---- |
| ① 缩小重建范围 | `ValueNotifier<String>` 存增量文本，`ValueListenableBuilder` 只包住当前这一条气泡 | 从重建整页 → 重建一个气泡 |
| ② 按帧合并 | token 到达只写入缓冲，用 `SchedulerBinding` 每帧刷一次 | 800 次重建 → 60 次/秒上限（通常更低） |
| ③ 分段冻结 | 已闭合的段落（遇到 `\n\n`）转成独立 Widget 并缓存，只有"最后一段"参与重建 | 长回复下重建成本恒定 |
| ④ 增量解析 Markdown | 只重新解析"未闭合"的尾部，前面的 AST 缓存复用 | 避免每次全量正则 |

```dart
// ② 按帧合并的典型写法
class StreamedText extends StatefulWidget { /* ... */ }

class _StreamedTextState extends State<StreamedText> {
  final _visible = ValueNotifier<String>('');
  String _pending = '';
  bool _scheduled = false;

  void onToken(String t) {
    _pending += t;
    if (_scheduled) return;
    _scheduled = true;
    SchedulerBinding.instance.scheduleFrameCallback((_) {
      _visible.value = _pending;   // 一帧只更新一次
      _scheduled = false;
    });
  }
}
```

**分段冻结的实现思路**：

```dart
// 已完成段落缓存成 Widget，只有尾段随流变化
final blocks = parseMarkdownIntoBlocks(fullText);
final stable = blocks.sublist(0, blocks.length - 1);
final tail = blocks.last;

Column(children: [
  ...stable.map((b) => _cachedBlock(b)),        // 命中缓存，直接复用
  RepaintBoundary(child: _RenderBlock(tail)),  // 只有这一段每帧变
]);
```

> 追问「为什么不直接用 `SelectableText`」：文本很长时，`SelectableText` 每次 setState 都要重新计算整段文本的选区与布局，开销随文本长度线性增长；而且它内部仍是整块重排。分段 + 常规 `Text`/`RichText` 组合才是可扩展方案（需要复制时按段提供选择，或整条消息转"复制全部"）。

### 7.3 Markdown 增量渲染

| 难点 | 说明 | 处理 |
| ---- | ---- | ---- |
| 未闭合语法 | `**粗体` 只来了前半，解析器行为不定 | 尾段用"宽松模式"，未闭合标记按纯文本显示；闭合后重解析 |
| 代码块高亮 | 逐行到达时代码块还在增长，语法高亮反复重算 | 代码块内部延迟高亮（如块闭合后再高亮），或只高亮最后 N 行 |
| 表格 | 表格需要整块结构才能渲染 | 表格期间显示原始文本，闭合后替换 |
| 数学公式/图片 | 依赖外部渲染，异步 | 占位 + 完成替换，避免布局跳动（预留高度） |
| 布局抖动 | 内容增长导致滚动位置跳 | 用 `reverse: true` 列表 + 自动吸底策略；用户手动上滑后停止吸底 |

**自动吸底规则**（聊天体验的关键细节）：

```
if (用户在底部附近) → 新内容到达时自动滚到底
if (用户手动上滑离开底部) → 停止吸底，显示"有新消息"按钮
if (用户回到底部) → 恢复吸底
```

判定阈值一般取 50–80px，且要用**滚动偏移与最大偏移的距离**判断，不能只看是否在滚动。

### 7.4 长列表：聊天列表的正确配置

```dart
ListView.builder(
  controller: _controller,
  reverse: true,                      // 聊天从底部开始长，天然贴底
  itemCount: messages.length,
  itemExtent: null,
  prototypeItem: MessageTile.prototype, // 高度固定时用，跳过逐项测量
  cacheExtent: 600,                    // 预构建区，减少快速滚动白屏
  addAutomaticKeepAlives: false,       // 消息不需要保活，省内存
  addRepaintBoundaries: true,          // 默认已开，保持
  itemBuilder: (ctx, i) => MessageTile(messages[i]),
)
```

| 优化点 | 说明 |
| ------ | ---- |
| `itemExtent` / `prototypeItem` | 高度可预测时给出，让列表跳过测量，滚动更稳 |
| `cacheExtent` | 太小 → 白屏；太大 → 内存与构建浪费 |
| `KeepAlive` 慎用 | 每个保活项都常驻内存，长会话必炸 |
| 图片固定尺寸 | 必须给 `width/height` 或 `AspectRatio`，否则加载完成时列表跳动 |
| `cacheWidth` | 大图按显示尺寸解码，别用原图 |
| 富内容分块 | 一个 item 里塞十几张图 → 拆成多个 item |
| 避免 `shrinkWrap: true` | 会强制测量全部子项，长列表禁用 |

### 7.5 记忆与任务：本地数据层

| 需求 | 方案 | 取舍 |
| ---- | ---- | ---- |
| 会话/消息持久化 | `drift`（SQLite，类型安全、支持复杂查询） | 首选，尤其要按时间/关键词检索 |
| 轻量 KV | `shared_preferences` | 只放设置项，不要放消息 |
| 文档型/对象缓存 | `isar` / `hive` | 读写快，复杂关联查询弱 |
| 大文件（录音） | 文件系统 + 元数据入库 | 别把音频塞数据库 |
| 全文检索 | SQLite FTS5 | 记忆检索的基础 |
| 向量检索 | 本地向量库或服务端 | 端侧成本高，通常服务端做 |

**同步冲突**（JD 提"记忆与任务"，一定会有多端同步问题）：

| 策略 | 适用 |
| ---- | ---- |
| LWW（最后写入胜出） | 简单、可接受丢少量更新（如设置项） |
| 版本号 + 服务端合并 | 消息类，服务端是权威 |
| 客户端操作日志（oplog）重放 | 任务状态变更，可审计 |
| CRDT | 离线协同编辑，成本最高 |

### 7.6 语音 Agent 的完整状态机（这个岗位的"主场景"）

```
        ┌──────┐  按下/唤醒    ┌─────────┐  检测到说话  ┌──────────┐
        │ Idle │──────────────▶│Listening│─────────────▶│Recognizing│
        └──────┘               └────┬────┘              └─────┬─────┘
             ▲                      │ 取消/静音超时           │ 文本到达
             │ 播放结束              ▼                        ▼
        ┌────┴─────┐           ┌────────┐              ┌──────────┐
        │ Speaking │◀──────────│Thinking│◀─────────────│  Text    │
        └────┬─────┘  首包TTS   └────────┘   请求发出    └──────────┘
             │                      ▲
             │ 用户打断(barge-in)    │ 网络失败 → 重试/降级为文字
             └──────────────────────┘
```

**每个状态的工程要点**：

| 状态 | 要点 |
| ---- | ---- |
| Listening | 麦克风占用、音量电平、VAD 检测、超时自动结束 |
| Recognizing | 部分结果实时显示（灰色/斜体），最终结果替换 |
| Thinking | 可取消；显示"思考中"但别超过 1.5s 无反馈（用户会以为卡死） |
| Speaking | 流式 TTS 首包延迟是体验关键（预缓冲）；打断要**同时**停播放 + 停止接收 + 通知服务端 |
| 全局 | 来电/其他 App 抢占音频 → 自动暂停并提示；切后台策略要明确（继续听 or 暂停） |

> 面试高光点：主动指出**打断（barge-in）是三方协同**——本地播放队列清空、Dart 层状态回滚到 Listening、服务端取消生成。只做本地停止会出现"TTS 还在后台生成、下次说话被前一段抢先播报"的诡异 bug。

## 八、性能与质量（P0）

### 8.1 帧预算：先算账再优化

| 刷新率 | 单帧预算 | 实际可用 |
| ------ | -------- | -------- |
| 60 Hz | 16.67 ms | UI ≈ 8ms + Raster ≈ 8ms 并行 |
| 90 Hz | 11.1 ms | 更紧张 |
| 120 Hz | 8.33 ms | 高端机才跑满 |

**两个线程是并行的**，所以"UI 用了 10ms"不一定掉帧（只要 Raster 也在 10ms 内）。判断标准是**总帧时间**与是否有**长任务阻塞**。

### 8.2 定位流程（面试要讲成方法论，不是罗列工具）

```
① 复现：--profile 模式（绝不能 release/profile 之外靠 debug 判断性能）
② 采集：DevTools Performance 录制 → 看 Frame Chart
③ 归因：
     UI 条长  → build / layout / paint 里谁重？
     Raster 条长 → 阴影、saveLayer、复杂裁剪、大图、着色器
     两者都短但卡 → 平台线程被阻塞（Channel handler / 原生 SDK）
④ 验证：改一处，只改一处，重新录制对比
⑤ 防回归：跑 integration test + 帧耗时基线
```

> 关键提醒：**debug 模式的性能数据一律不可信**。debug 下是 JIT、断言全开、没有 AOT 优化，慢 3–10 倍很正常。用 debug 下的表现下结论是最常见的减分项。

### 8.3 常见卡顿与对策（背这张表）

| 现象 | 根因 | 对策 |
| ---- | ---- | ---- |
| 首次进入某页卡一下 | 着色器编译 / 大图解码 | Impeller 已大幅缓解；图片 `cacheWidth`；骨架屏 |
| 列表快速滚动白屏 | `cacheExtent` 太小、item 太重 | 调 `cacheExtent`、拆 item、固定高度 |
| 动画掉帧 | 动画区与内容区同 Layer | `RepaintBoundary` 包动画区 |
| 阴影/圆角裁剪卡 | `saveLayer` 产生离屏渲染 | 减少 `ClipRRect` 嵌套、用 `BoxShadow` 替代 `ClipPath`、避免 `Opacity` 用 `AnimatedOpacity` 之外的场景 |
| 整页重建 | `setState` 作用域过大 | 拆 Widget、`ValueListenableBuilder`、`Selector` |
| 输入框打字卡 | 每次输入触发全页校验/搜索 | 防抖 + 局部刷新 |
| 平台线程卡 | Channel handler 做重活 | 切后台线程（见 4.3） |
| 内存抖动 | 频繁创建大对象、无 `const` | `const` 构造、对象复用、`dispose` 到位 |

### 8.4 内存

| 泄漏源 | 排查/解决 |
| ------ | --------- |
| 未 `dispose` 的 Controller | DevTools Memory → 快照对比 |
| 未 `cancel` 的 `StreamSubscription` | 同上，重点看原生监听是否被注销 |
| `GlobalKey` 持有大量子树 | 避免用 GlobalKey 做全局状态 |
| 图片缓存无限增长 | `PaintingBinding.instance.imageCache` 设上限 |
| 闭包捕获大对象 | 事件回调里别捕获整个页面 State |
| 原生侧持有 Activity/Context | Android LeakCanary 配合排查 |

### 8.5 启动优化

| 阶段 | 手段 |
| ---- | ---- |
| 引擎初始化 | 尽量少在 `main()` 里做同步初始化；耗时的挪到首帧后 |
| 首屏 | 骨架屏 + 本地缓存数据先渲染，再拉网络 |
| 延迟加载 | 非首屏 feature 用 `deferred as`（注意：Dart 延迟加载在 Flutter 上支持有限，主要靠代码拆分包） |
| 原生侧 | Android 冷启动看 `Displayed` 时间；iOS 看 `Xcode Organizer` 启动指标 |
| 体积 | `--split-debug-info`、`--obfuscate`、资源压缩、移除未用字体 |

### 8.6 测试：三层，别只写一层

| 层级 | 工具 | 测什么 | 这个岗位的重点 |
| ---- | ---- | ------ | -------------- |
| 单元测试 | `test` + `mocktail` | 状态机迁移、协议编解码、错误映射 | **协议分帧/CRC/状态机**必须单测 |
| Widget 测试 | `flutter_test` | 组件渲染、交互、状态切换 | 设备状态各分支 UI（六态都要有） |
| 集成测试 | `integration_test` | 真机端到端 | 录音→上传→转写→播放全链路 |
| 黄金测试 | `matchesGoldenFile` | 像素级回归 | 组件库改版防走形（注意字体差异，需固定字体） |

```dart
// 通道层的单测：不用真机，直接 mock 平台
setUp(() {
  TestDefaultBinaryMessengerBinding.instance.defaultBinaryMessenger
      .setMockMethodCallHandler(channel, (call) async {
    if (call.method == 'getDeviceInfo') return {'id': 'd1', 'battery': 80};
    return null;
  });
});
```

> 加分说法：「插件边界我会用接口隔离 + Pigeon 生成类 mock，让 90% 的业务逻辑能在纯 Dart 单测里跑，只有真机相关的部分进 integration test。」这句话直接命中 JD 的"自动化测试"。

### 8.7 稳定性与可观测

| 项 | 做法 |
| -- | ---- |
| Dart 异常 | `FlutterError.onError` + `PlatformDispatcher.instance.onError` + `runZonedGuarded` |
| 原生崩溃 | Crashlytics / Bugly，Dart 侧崩溃工具抓不到原生崩溃 |
| 卡顿监控 | 自建帧回调采样（`SchedulerBinding.addTimingsCallback`），上报超阈值的帧 |
| 日志 | 分级 + 设备日志随问题上报（注意隐私：录音内容绝不入日志） |
| 灰度 | 按设备型号/系统版本分桶，设备类问题强相关硬件 |
| 降级 | 新固件/新协议失败 → 回退旧路径；某平台能力缺失 → 隐藏入口而非崩溃 |

## 九、发布与多端（P1）

### 9.1 Android 发布

| 项 | 要点 |
| -- | ---- |
| 签名 | keystore + `key.properties`，**不要提交到仓库**；CI 用环境变量注入 |
| 产物 | Play 商店用 **AAB**；国内渠道/自建分发用 APK，可按 ABI `--split-per-abi` 减体积 |
| 混淆 | R8/ProGuard；注意保留 Pigeon 生成类、反射用到的类、序列化模型 |
| SDK 版本 | 3.47 模板 targetSdk 36 / minSdk 24；上架前对齐商店要求 |
| 权限 | 蓝牙/录音/通知都要运行时申请，且**说明用途**（审核会看） |
| 渠道 | 国内多渠道用 flavor + 渠道包注入 |
| 常见驳回 | 后台录音未说明、隐私政策缺失、权限申请时机不合理（一进 App 就弹） |

### 9.2 iOS 发布

| 项 | 要点 |
| -- | ---- |
| 证书 | 开发/分发证书 + Provisioning Profile；CI 用 `fastlane match` 统一管理 |
| 依赖 | 3.44 起默认 **Swift Package Manager**，CocoaPods 仅维护模式；老插件可能仍需 Pod |
| 测试分发 | TestFlight（内部 100 人 / 外部需审核） |
| 隐私 | `Info.plist` 用途文案必须具体（"用于录制语音并转写"而非"需要麦克风"）；`PrivacyInfo.xcprivacy` 声明所用 API |
| 后台音频 | `UIBackgroundModes: audio` 滥用会被拒，必须与功能强相关 |
| 审核高频坑 | 蓝牙配件需说明用途、需要硬件的功能要提供演示视频、账号删除入口必须存在 |
| 崩溃符号 | 上传 dSYM，否则崩溃栈不可读 |

### 9.3 CI/CD

```
push → 静态检查（flutter analyze / dart format --set-exit-if-changed）
     → 单测 + widget 测试（覆盖率门禁）
     → 构建（Android AAB / iOS IPA，并行）
     → 集成测试（真机云：Firebase Test Lab / 自建设备农场）
     → 分发（TestFlight / 蒲公英 / 内部渠道）
     → 符号上传 + 版本号自增
```

| 工具 | 适用 |
| ---- | ---- |
| GitHub Actions | 通用，macOS runner 可构建 iOS |
| Codemagic | Flutter 专用，配置最省心 |
| Fastlane | 与上面任一配合，管证书与商店上传 |
| 自建 | 有设备农场/内网需求时 |

### 9.4 桌面端（Windows / macOS）：JD 第 5 条的落地

**先明确"能跑"和"好用"的差距**：

| 维度 | 移动端习惯 | 桌面端必须改 |
| ---- | ---------- | ------------ |
| 窗口 | 全屏固定 | 可缩放、最小尺寸、位置记忆（`window_manager`） |
| 导航 | 底部 Tab / 页面栈 | 侧边栏 + 多栏布局；返回键不存在，要有面包屑 |
| 交互 | 触摸 | 鼠标 hover、右键菜单、滚轮、拖拽文件 |
| 快捷键 | 无 | `Shortcuts` + `Actions`，Cmd/Ctrl 组合键，Esc 关闭弹窗 |
| 菜单 | 无 | `PlatformMenuBar`（macOS 菜单栏）、Windows 窗口菜单 |
| 滚动 | 手指拖动 | 桌面默认鼠标滚轮，`ScrollBehavior` 需调整 `dragDevices` |
| 字体 | 系统默认即可 | macOS 用 SF、Windows 用 Segoe，行高/字重差异会导致布局溢出 |
| 文件 | `path_provider` 沙盒 | 需要文件选择器、拖放、最近文件 |
| 窗口尺寸变化 | 不适用 | 必须响应式，测试 320px ~ 4K 全区间 |
| 平台能力 | 传感器/蓝牙 | 桌面蓝牙能力弱，功能要能降级或隐藏 |

**代码层面的多端准备**（这才是 JD 说的"为后续复用做好组件和接口设计"）：

```dart
// ① 能力探测，而不是平台判断
abstract class PlatformCapabilities {
  bool get hasBluetooth;
  bool get hasBackgroundAudio;
  bool get supportsSystemShare;
}

// ② 宿主差异收敛到一处
PlatformCapabilities capabilitiesFor(TargetPlatform p) => switch (p) {
  TargetPlatform.android || TargetPlatform.iOS => MobileCapabilities(),
  TargetPlatform.macOS => MacCapabilities(),
  TargetPlatform.windows => WindowsCapabilities(),
  _ => const NullCapabilities(),
};
```

```yaml
# ③ 依赖先查支持矩阵，别写完才发现不支持
# pub.dev 页面顶部会标注 ANDROID / IOS / LINUX / MACOS / WEB / WINDOWS
```

**插件支持矩阵是这个岗位的真实风险点**：BLE、音频、后台服务类插件大多只支持移动端。做法是**在架构上就允许"某平台无实现"**——`NullCapabilities` 返回 false，UI 隐藏入口，而不是到处 `if (!kIsWeb && Platform.isAndroid)`。

### 9.5 无障碍（加分）

| 项 | 做法 |
| -- | ---- |
| 语义树 | `Semantics(label: ...)`、`MergeSemantics`（把一组合成一个可聚焦项）、`ExcludeSemantics`（装饰性图标） |
| 可点击区域 | 不小于 48×48 dp |
| 焦点顺序 | `FocusTraversalGroup`，弹窗打开时焦点要进入弹窗 |
| 对比度 | 正文 ≥ 4.5:1 |
| 朗读适配 | TalkBack / VoiceOver 实测；波形、进度等自绘内容要提供文字化描述 |
| 动效 | 尊重系统"减少动态效果"（`MediaQuery.disableAnimations`） |
| 测试 | `flutter test` 里用 `SemanticsTester` 断言语义节点 |

### 9.6 多语言（加分）

| 项 | 做法 |
| -- | ---- |
| 方案 | `flutter_localizations` + ARB 文件 + `gen-l10n`（官方） |
| 占位符 | 用 ARB 的 `placeholders`，不要字符串拼接 |
| 复数 | `Intl.plural`（英文 one/other，中文只有 other） |
| 布局 | 预留 30%+ 宽度余量；德语/俄语很长，中文很短 |
| RTL | `Directionality` 自动镜像；图标要区分"方向性图标"（箭头）与"无语义图标" |
| 日期数字 | `intl` 的 `DateFormat`/`NumberFormat`，不要手写格式 |
| 切换语言 | 需要重建整棵树（`WidgetsApp` 的 `locale` 变化会触发），注意保持页面状态 |

### 9.7 KMP 与 Flutter 的配合（加分项，问法通常是"你怎么看"）

| 方案 | 结构 | 优缺点 |
| ---- | ---- | ------ |
| KMP 只共享业务逻辑 | Kotlin 写 SDK → Android 打成 AAR、iOS 打成 Framework → Flutter 通过 Pigeon/FFI 调 | ✅ UI 仍用 Flutter 一套；❌ 多一层桥接，调试变复杂 |
| KMP 共享 UI（Compose Multiplatform） | 与 Flutter 二选一 | ❌ 和 Flutter 冲突，属于替代关系 |
| 纯 Flutter | 逻辑也用 Dart 写 | ✅ 最简单；❌ 复用不了已有 Kotlin 资产 |

**KMP SDK 接入的实际坑**：

- iOS Framework 需要正确设置 `xcodebuild` 产物路径与架构（arm64 真机 + 模拟器），Flutter 的 Podfile/SPM 里要挂上。
- 崩溃栈跨 Kotlin/Native + Dart，符号化要两头都做。
- Kotlin/Native 有内存模型与线程约束，跨线程回调要回到主线程。
- 包体积增加明显（Kotlin 运行时）。
- 版本升级牵动三端，需要 contract 测试兜底。

## 十、高频面试题（按 JD 命中率排序）

### 10.1 Flutter 与 Dart 基础

#### 10.1.1 Widget、Element、RenderObject 的关系？

**考察点**：是否理解声明式 UI 的复用机制，而不是背概念。

**要点**：

- Widget 是不可变配置；Element 是挂载实例与 diff 单位，也是 `BuildContext`；RenderObject 负责布局与绘制。
- `canUpdate` 判断 `runtimeType + key`，命中则复用 Element/RenderObject，只更新配置。
- State 挂在 StatefulElement 上，所以 Widget 每帧重建、State 不丢。
- 组合型 Widget（`Container`、`Padding`）没有自己的 RenderObject。

#### 10.1.2 `setState` 之后发生了什么？会重建整棵树吗？

**考察点**：dirty 机制与重建范围。

**要点**：

- `setState` → `markNeedsBuild()` → 该 Element 及其祖先标记 dirty → 下一帧只重建 dirty 子树。
- `const` 构造的子树 `identical` 命中，整棵跳过。
- 重建范围过大是常见性能问题，解法是拆小 StatefulWidget 或改用局部 notifier。

#### 10.1.3 `await` 会阻塞 UI 线程吗？

**考察点**：事件循环模型。

**要点**：

- 不会。`await` 把续体注册到微任务队列，线程立刻回到事件循环。
- 真正阻塞的是同步 CPU 计算、`sleep`、同步 IO。
- 微任务清空前不会处理事件，微任务里无限调度会饿死 UI。

#### 10.1.4 Future 能取消吗？流式请求怎么中断？

**考察点**：是否真的做过流式功能。

**要点**：

- `Future` 无取消语义；`StreamSubscription.cancel()` 可以中断流。
- 用 `dio.CancelToken` 或 `HttpClientRequest.abort()` 关闭底层连接。
- 取消要三层同时做：本地订阅、网络连接、服务端生成。

#### 10.1.5 Stream 的背压怎么处理？

**考察点**：流式数据量控制。

**要点**：

- `await for` 天然背压；`listen` 没有，需要 `pause/resume` 或主动丢帧。
- 高频源（PCM、BLE notify）必须做节流/合并，否则内存与 UI 双爆。

#### 10.1.6 什么时候必须用 Isolate？后台 isolate 能用平台通道吗？

**考察点**：并发边界认知。

**要点**：

- 持续 CPU 密集（音频编码、加解密、大文件解析）必须开 isolate；一次性任务用 `Isolate.run`。
- isolate 无共享内存，靠端口传消息，闭包捕获对象不可传。
- 后台 isolate 默认不能用通道，需 `BackgroundIsolateBinaryMessenger` + `RootIsolateToken`。

### 10.2 状态管理与架构

#### 10.2.1 你怎么选状态管理方案？

**考察点**：是否有取舍能力，而不是站队。

**要点**：

- 先分 ephemeral 与 app state；局部状态留在 `State`。
- 全局状态按"异步复杂度 + 团队规模 + 可测试性"选：Riverpod（`AsyncValue` 表达三态）、Bloc（状态机可穷举）。
- 反对 GetX 的理由是全局单例与隐式依赖，不利于测试与多人协作。

#### 10.2.2 一个 feature 的目录怎么组织？

**考察点**：可维护性。

**要点**：

- feature-first：`data / domain / presentation` 三层。
- `presentation` 不直接依赖 Pigeon 生成代码，只依赖 `core/platform` 的接口。
- 跨 feature 复用下沉 `shared/`，禁止横向依赖。

#### 10.2.3 多设备形态（手机/平板/桌面）怎么适配？

**考察点**：响应式设计而非"平台判断"。

**要点**：

- 断点驱动（compact/medium/expanded），不看 `Platform.isAndroid`。
- 内容宽度约束（`maxWidth`），避免桌面端一行拉太长。
- 能力探测（`hasBluetooth`）替代平台判断，缺能力就隐藏入口。

### 10.3 平台通道与插件

#### 10.3.1 MethodChannel、EventChannel、Pigeon 怎么选？

**考察点**：平台边界基本功。

**要点**：

- 单次命令 → MethodChannel；持续事件 → EventChannel（或 Pigeon 的 `@EventChannelApi`）；自研插件一律 Pigeon。
- Pigeon 是类型安全，不是更快；性能等价于手写通道。
- 需要极低延迟/高频才考虑 FFI。

#### 10.3.2 Channel 调用把界面卡住了，怎么排查？

**考察点**：线程模型，最高频的实战题。

**要点**：

- 原生 handler 默认在平台主线程执行，重活必须切后台，完成后回主线程回调。
- Flutter DevTools 看不到这类卡顿，要看 Android `StrictMode`/`BlockCanary`、iOS Instruments。
- 解法：`withContext(Dispatchers.IO)` / `DispatchQueue.global()`，或 Pigeon `@TaskQueue`。

#### 10.3.3 EventChannel 的泄漏怎么避免？

**考察点**：生命周期纪律。

**要点**：

- Dart 侧必须 `cancel()`；原生侧 `onCancel` 里注销监听、置空 sink。
- 原生侧持 `BinaryMessenger`/`Context`，不持 Activity。
- 推事件前判断 sink 是否为空，并回主线程。

#### 10.3.4 原生抛异常到 Dart 会发生什么？

**考察点**：错误传播。

**要点**：

- Android 上 handler 内未捕获异常会**崩溃整个 App**，不是只让这次调用失败。
- 规范做法：转 `FlutterError(code, message, details)` → Dart 收到 `PlatformException` → 服务层翻译成领域错误。
- 未实现的平台方法返回 `MissingPluginException`，要能降级。

#### 10.3.5 Pigeon 有哪些工程约束？

**考察点**：是否真在生产用过。

**要点**：

- 各端生成代码必须同版本，否则未定义行为。
- 生成代码不是稳定 API，不暴露到包公开接口、不跨包拆。
- 契约文件只能声明不能写逻辑。
- 版本升级要作为一次整体变更，配 contract 测试。

### 10.4 IoT / 设备

#### 10.4.1 设备连不上，你怎么逐层排查？

**考察点**：系统性排查能力（这题几乎必问）。

**要点**：

1. 权限层：定位/蓝牙权限是否授予，Android 版本对应的权限组合是否正确。
2. 扫描层：是否扫到广播（UUID、信号强度），扫描模式与过滤条件是否过严。
3. 连接层：连接超时、配对状态、是否被其他设备占用。
4. 服务发现层：`discoverServices` 是否成功，特征 UUID 是否与固件版本匹配。
5. 协议层：MTU、分包、CRC、命令 seq 是否对。
6. 系统层：厂商 ROM 后台限制、蓝牙栈缓存（需重启蓝牙/清缓存）。

#### 10.4.2 BLE 的分包与粘包怎么处理？

**考察点**：协议工程能力。

**要点**：

- 默认 MTU 23 → 有效 20 字节，必须按 `MTU-3` 切片。
- 帧头带长度字段，接收端维护缓冲，按长度切分，不足则等待。
- CRC 校验失败要重传；重传命令必须幂等（带 requestId）。

#### 10.4.3 断连重连策略怎么设计？

**考察点**：稳定性设计。

**要点**：

- 指数退避 + 抖动，避免惊群。
- Android 可用 `autoConnect=true`（省电但慢），iOS 需自己重连 + 后台模式。
- 重连期间 UI 要有明确状态与次数提示，不能静默失败。

#### 10.4.4 固件升级（OTA）要考虑什么？

**考察点**：是否做过真实硬件项目。

**要点**：

- 分片 + 逐片校验 + 断点续传；整包哈希 + 版本/机型校验。
- A/B 双分区与失败回滚；升级中保活、禁止息屏。
- 用户侧进度、剩余时间、失败原因、可续传。

### 10.5 音频

#### 10.5.1 为什么用蓝牙耳机录音音质很差？

**考察点**：协议栈理解（加分项经典题）。

**要点**：

- 经典蓝牙录音走 HFP/HSP，窄带 8k/16k 单声道；A2DP 只支持播放。
- iOS 必须 `.allowBluetoothHFP` 才切到耳机麦克风。
- 想高质量需 LE Audio + LC3（Android 13+/iOS 16+ 且硬件支持）。

#### 10.5.2 App 切后台录音被杀了怎么办？

**考察点**：平台后台机制。

**要点**：

- Android：前台服务 + `foregroundServiceType="microphone"`（14+ 强制）+ 通知权限；厂商 ROM 需引导加白。
- iOS：`UIBackgroundModes: audio` + 保持 `AVAudioSession` active。
- 兜底：本地先落盘，进程被杀后恢复时可续传。

#### 10.5.3 录音被打断（来电、其他 App 播放）怎么处理？

**考察点**：异常场景完整性。

**要点**：

- iOS 监听 `interruptionNotification`，`.began` 暂停，`.ended` 按 `shouldResume` 恢复。
- Android 处理 `AudioFocus` 丢失，暂停并提示。
- UI 上要有"录音已暂停/继续"的明确状态，不能静默。

#### 10.5.4 边录边传怎么设计？

**考察点**：流式链路。

**要点**：

- 20ms 帧 → 编码 → 200~500ms 分片 → WebSocket 上行。
- 同时本地落盘；断网后按 offset 续传；服务端需幂等。
- 背压：缓冲上限 + 降码率 + 丢帧策略。
- 结束要有显式 `end` 帧，不能只靠 VAD。

### 10.6 流式文本与列表

#### 10.6.1 逐字输出怎么做到不掉帧？

**考察点**：这个岗位的核心题。

**要点**：

- 四层：`ValueNotifier` 局部刷新 → 按帧合并 → 已闭合段落冻结缓存 → 增量 Markdown 解析。
- 不要每次 token 都 `setState` 整页。
- 长文本下避免 `SelectableText` 整段重排。

#### 10.6.2 聊天列表怎么优化？

**考察点**：长列表基本功。

**要点**：

- `reverse: true` + 自动吸底策略（用户上滑则停止吸底）。
- `itemExtent`/`prototypeItem`、`cacheExtent` 调优、`addAutomaticKeepAlives: false`。
- 图片固定尺寸 + `cacheWidth`；禁 `shrinkWrap`。

#### 10.6.3 流式返回的 JSON 怎么解析？

**考察点**：协议健壮性。

**要点**：

- 半包/粘包：按分隔符切分 + 残余缓冲。
- UTF-8 多字节截断：用流式 decoder。
- 增量字段（delta）累积；异常帧跳过而非整体失败。

### 10.7 性能与质量

#### 10.7.1 掉帧怎么定位？

**考察点**：方法论。

**要点**：

- `--profile` 录制 → 看 UI 还是 Raster 长 → 再细分 build/layout/paint。
- 两者都短但卡 → 平台线程被阻塞。
- debug 数据不可用；改一处验证一处。

#### 10.7.2 内存泄漏怎么查？

**考察点**：实战经验。

**要点**：

- DevTools Memory 快照对比、`dispose` 审查、订阅与定时器清单。
- 原生侧用 LeakCanary（Android）/ Instruments（iOS）。
- 常见源：Controller、GlobalKey、闭包捕获、图片缓存、原生持 Activity。

#### 10.7.3 你怎么做自动化测试？

**考察点**：JD 明确要求。

**要点**：

- 三层：单测（状态机/协议）、widget（各状态 UI）、integration（真机全链路）。
- 通道用 mock handler 或接口隔离，让业务逻辑可纯 Dart 测试。
- golden test 守住组件库视觉回归。

### 10.8 发布与多端

#### 10.8.1 Android 与 iOS 发布流程的关键差异？

**考察点**：是否真发过包。

**要点**：

- Android：keystore 管理、AAB、R8 保留规则、多渠道。
- iOS：证书/描述文件、SPM 与 Pod 混用、TestFlight、审核用途文案与隐私清单。
- 双端都要处理权限文案与后台能力声明，这是被拒的高频原因。

#### 10.8.2 同一套 Flutter 代码上 macOS，要改什么？

**考察点**：多端思维。

**要点**：

- 窗口管理、导航结构（侧边栏/多栏）、菜单与快捷键、鼠标 hover 与右键、滚动行为、字体差异。
- 插件支持矩阵：BLE/音频多半不支持桌面 → 能力探测 + 降级。
- 从第一天就把能力抽成接口，不要事后 `Platform.isXxx` 打补丁。

#### 10.8.3 KMP 和 Flutter 怎么配合？

**考察点**：技术视野（加分）。

**要点**：

- 共享逻辑（Kotlin → AAR/Framework），UI 仍用 Flutter，通过 Pigeon/FFI 桥接。
- 代价：桥接层、符号化、包体积、三端版本联动。
- Compose Multiplatform 与 Flutter 是替代关系，不是叠加。

## 十一、项目叙事与反问

### 11.1 三个故事的讲法模板

**故事①：流式渲染改造**

```
背景：Agent 回复逐字输出，长回复（>500 字）时明显卡顿，低端机掉帧率 30%
动作：① DevTools 定位到 UI 线程 build 阶段（Markdown 全量解析）
      ② 改 ValueNotifier 局部刷新 + 按帧合并
      ③ 已闭合段落缓存 Widget，只重建尾段
      ④ 增量解析，只重算未闭合块
结果：单帧 UI 耗时从 24ms → 6ms，掉帧率降到 3%
反思：早期就该把"流式内容"当成独立的渲染单元设计
```

**故事②：原生插件从 0 到 1**

```
背景：设备厂商只提供 Android/iOS SDK，无 Flutter 支持
动作：① 用 Pigeon 定契约（设备信息、命令、事件流）
      ② 重活切 IO 线程，回主线程回调
      ③ 错误码统一，Dart 侧翻译成领域错误
      ④ onCancel 注销原生监听，避免泄漏
结果：双端一套接口，业务层零平台判断
反思：契约先行让后续加设备型号几乎零成本
```

**故事③：设备连接稳定性**

```
背景：设备断连率高，用户投诉"设备离线但 App 显示已连接"
动作：① 引入显式状态机（六态），每态对应 UI
      ② 指数退避 + 抖动重连，心跳保活
      ③ 断连时立刻更新 UI 并给可操作建议
结果：误报"已连接"归零，重连成功率提升到 XX%
反思：状态机比布尔值可靠，UI 是协议的一部分
```

### 11.2 该问回去的问题（显示你真的懂这个岗位）

| 问题 | 你在探什么 |
| ---- | ---------- |
| 设备协议是我们自己定，还是跟硬件团队一起定？有版本兼容策略吗？ | 判断是否需要自己设计分帧/兼容层 |
| 录音是端侧 ASR 还是全量上传？上行带宽与合规怎么权衡？ | 判断流式链路复杂度 |
| Agent 的流式协议是 SSE 还是 WebSocket？有中断（barge-in）需求吗？ | 判断渲染与状态机难度 |
| 桌面端是"复用 UI"还是"复用能力层"？现有原生 SDK 有桌面实现吗？ | 判断多端工作的真实范围 |
| 团队现在 Flutter 与原生的人力配比？插件是 Flutter 同学写还是原生同学写？ | 判断你是否要兼写 Kotlin/Swift |

### 11.3 面试前一小时自查

| 检查项 | 标准 |
| ------ | ---- |
| 能 3 分钟讲清三棵树与一帧管线 | 不看稿，有图在脑子里 |
| 能说清 Channel 的线程模型与三类错误 | 能举自己踩过的坑 |
| 能画出设备连接状态机 | 六态 + 每态 UI |
| 能讲 SSE 解析的三个坑 | 半包、UTF-8 截断、断线续接 |
| 能说出流式渲染四层优化 | 顺序不能乱 |
| 能报出自己项目的 2 个数字 | 帧耗时、掉帧率、重连成功率任选 |
| 能说清 iOS/Android 后台录音的差异 | 具体到 API 与权限名 |
| 桌面端能说出 5 个差异点 | 窗口、导航、快捷键、字体、插件支持 |

## 小结

- 这个岗位的**筛人点**是四件事：Dart 异步、状态管理、**平台通道线程模型**、**流式渲染性能**。前两个决定你会不会写，后两个决定你配不配这个 JD。
- **差异化竞争力**在 IoT 与音频：能讲清 BLE 状态机、分包协议、静音/打断/后台这些"不舒服的边界"，比多背十个 Flutter API 有用。
- **多端不是最后再适配**，而是从第一天把能力抽成接口、把布局做成断点驱动、把依赖限定在支持矩阵内。
- 面试表达的原则：**先给结论，再给机制，最后给一个你亲手处理过的数字**。


# Android 基础

> Android 常用知识速览：系统架构 → 四大组件 → 消息机制与 Binder → View 体系 → 进程/线程与内存 → 存储网络 → Jetpack 与架构 → Compose → 性能优化。定位为复习速查，配合 [Android 面试题](Android面试题.md) 使用。

## 一、Android 系统架构

### 1.1 五层架构

| 层级 | 内容 | 说明 |
| --- | --- | --- |
| Application | 系统应用 + 三方应用 | Java/Kotlin 编写，运行在 ART |
| Framework（Java API） | AMS、WMS、PMS、View 体系、PackageManager | 通过 Binder 与系统服务通信 |
| Libraries + Runtime | ART 虚拟机、SQLite、OpenGL、WebKit、libc（Bionic） | Native 层（C/C++） |
| HAL | 硬件抽象层（相机、传感器、蓝牙） | 统一接口，厂商实现 |
| Linux Kernel | 进程/内存管理、Binder 驱动、电源、驱动 | Android 对内核做了 Binder、Ashmem、Low Memory Killer 等补丁 |

- Android 是**基于 Linux 内核**但并非标准 Linux 发行版：没有 glibc（用 Bionic）、没有原生 X Window、有独有的 Binder IPC 与 LMK。

### 1.2 ART 与 Dalvik

- **Dalvik**（Android 4.4 及以前）：JIT（运行时编译热点代码），dex 字节码，每次运行都要编译，安装快、启动慢、耗电。
- **ART**（5.0 起）：安装时 **AOT** 预编译为本地机器码（dex2oat），启动快、耗电低，代价是安装慢、占用更多存储空间。
- **7.0 起混合模式**：安装时不全量编译（首次启动快），运行时 JIT 记录热点方法生成 profile，**空闲+充电时**再按 profile 编译（profile-guided compilation）。
- `.dex`：Dalvik 可执行文件，一个 APK 可包含多个 dex（MultiDex 解决 65536 方法数限制）。

### 1.3 Zygote 与进程孵化

- `init` 进程 → 启动 **Zygote**（`app_process`），它预加载 Framework 类、系统资源、共享库。
- 启动应用时 Zygote **fork** 出子进程（Copy-On-Write 共享预加载内存），因此应用启动快、内存占用小。
- **SystemServer** 也是 Zygote fork 出来的，承载 AMS、WMS、PMS 等系统服务。
- 通信方式：Zygote 通过 **Socket**（而非 Binder）接收 fork 请求——fork 时多线程 Binder 状态不安全。

### 1.4 APK 结构与构建流程

```
app.apk
├── AndroidManifest.xml    # 组件声明、权限、包名（编译后为二进制 AXML）
├── classes.dex            # Java/Kotlin 字节码（可多个）
├── resources.arsc         # 资源索引表（id ↔ 资源映射）
├── res/                   # 编译后的资源（layout/drawable/values）
├── assets/                # 原始文件，按路径访问
├── lib/                   # 各 ABI 的 .so（armeabi-v7a/arm64-v8a/x86...）
└── META-INF/              # 签名信息
```

- 构建链路：`.kt/.java` → kotlinc/javac → `.class` → **D8/R8** → `classes.dex`（R8 同时做**代码压缩 + 混淆 + 优化**）；资源经 **AAPT2** 编译链接生成 `resources.arsc` 与 `R.java` → 打包 → `zipalign` 对齐 → `apksigner` 签名。
- 签名方案：**v1**（JAR 签名，校验每个文件，慢且可被改 zip 元数据）、**v2**（7.0+，校验整个 APK 的签名块，防篡改）、**v3**（9.0+，支持密钥轮转）、**v4**（11+，配合增量安装）。

## 二、四大组件

| 组件 | 作用 | 注册方式 | 所在线程 | 典型场景 |
| --- | --- | --- | --- | --- |
| Activity | 展示 UI、与用户交互 | Manifest | 主线程 | 页面 |
| Service | 后台长时间任务 / 跨进程调用 | Manifest | 主线程（默认） | 播放音乐、下载 |
| BroadcastReceiver | 接收系统/应用广播 | Manifest 或动态 | 主线程（onReceive） | 网络变化、开机 |
| ContentProvider | 跨应用数据共享 | Manifest | 主线程（onCreate/CRUD） | 通讯录、相册 |

- 四者都由 **AMS** 统一管理，都必须在 Manifest 声明（动态广播除外），都**默认运行在主线程**，耗时操作都要另起线程。
- Activity / Service / BroadcastReceiver 的启动都经过 **Binder → AMS**，由 `ActivityThread` 通过 `Handler` 回调到应用主线程。

## 三、Activity

### 3.1 生命周期

```
onCreate → onStart → onResume   （前台可见可交互，resumed）
onPause  → onStop               （被部分/完全遮挡）
onDestroy                       （销毁）
onRestart                       （stopped → started）
```

- `onCreate`：`setContentView`、初始化 ViewModel、恢复 `savedInstanceState`。
- `onStart`：可见但不可交互；`onResume`：可交互（获取焦点）。
- `onPause`：**必须轻量**，因为新 Activity 要等它执行完才启动；不能做耗时操作。
- `onStop`：完全不可见，释放/暂停动画、相机等重量级资源。

典型流程：

- A 启动：A.onCreate → A.onStart → A.onResume
- A 打开 B：A.onPause → B.onCreate → B.onStart → B.onResume → A.onStop（B 为透明/对话框主题时 A 只走到 onPause）
- B 返回 A：B.onPause → A.onRestart → A.onStart → A.onResume → B.onStop → B.onDestroy

### 3.2 异常销毁与状态保存

- `onSaveInstanceState(outState)`：在 `onStop` 前后（API 28+ 保证在 onStop 之后）保存临时 UI 状态（输入内容、列表位置）到 Bundle，重建时在 `onCreate` / `onRestoreInstanceState` 取回。
- 触发场景：旋转屏幕、语言切换、内存不足被回收、开发者选项"不保留活动"。
- 保存的是**轻量可序列化数据**（Bundle 有 1MB 左右的 Binder 限制），大数据用 `ViewModel` + 持久化。
- `ViewModel` 可跨配置变更存活，但**进程被杀仍丢失** → 配合 `SavedStateHandle`。

### 3.3 启动模式（launchMode）

| 模式 | 行为 | 场景 |
| --- | --- | --- |
| standard | 每次 new 一个实例入栈 | 默认 |
| singleTop | 栈顶已是该 Activity 则复用，回调 `onNewIntent` | 通知跳转、防重复点击 |
| singleTask | 栈内复用，并**清除其上的所有 Activity** | 主页、登录页 |
| singleInstance | 独占一个 Task，栈中只有它 | 来电界面、独立任务 |

- Intent Flag：`FLAG_ACTIVITY_NEW_TASK`、`FLAG_ACTIVITY_CLEAR_TOP`、`FLAG_ACTIVITY_SINGLE_TOP`、`FLAG_ACTIVITY_CLEAR_TASK`。
- `taskAffinity`：指定 Activity 归属的任务栈；配合 `allowTaskReparenting` 可让 Activity 回到自己的栈。
- 优先级：**Intent Flag > manifest launchMode**。

### 3.4 启动流程（简版）

```
Launcher.startActivity
  →（Binder）AMS.startActivity
  → 解析 Intent、权限校验、栈管理（ActivityStarter/ActivityStack）
  → 目标进程不存在：AMS →（Socket）Zygote → fork → ActivityThread.main()
  → ActivityThread.attach → 绑定 ApplicationThread（Binder 客户端）
  → AMS.scheduleLaunchActivity →（Binder）→ ApplicationThread → H(Handler) 切主线程
  → 反射创建 Activity → attach → onCreate → onStart → onResume
```

## 四、Fragment

- 生命周期：`onAttach` → `onCreate` → `onCreateView` → `onViewCreated` → `onStart` → `onResume` →（paused/stopped）`onDestroyView` → `onDestroy` → `onDetach`。
- 注意：**Fragment 的生命周期由宿主 Activity 驱动**，Activity 处于 resumed 时 Fragment 才能 resumed；`onDestroyView` 后 View 被销毁但 Fragment 实例仍在（ViewBinding 要在此置空，防泄漏）。
- 传参用 `arguments`（Bundle）+ `fragmentFactory`/`newInstance` 静态工厂，**不要用带参构造**（重建时会反射调用无参构造导致参数丢失）。
- 事务：`FragmentManager.beginTransaction()` → `add/replace/remove/hide/show` → `addToBackStack()` → `commit()`。
  - `commit`：异步（post 到主线程队列），在 `onSaveInstanceState` 之后调用会抛异常 → 用 `commitAllowingStateLoss`（有状态丢失风险）。
  - `commitNow`：同步执行，不能加入回退栈。
- `replace` 会销毁并重建 Fragment 视图；`add + hide/show` 保留实例，适合底部 Tab 切换。
- 通信方式：**共享 `ViewModel`（Activity 作用域，推荐）**、Fragment Result API（`setFragmentResultListener`）、Bundle、接口回调（旧）。

## 五、Service

- **启动态**（`startService`）：`onCreate` → `onStartCommand`（每次 start 都回调）→ `onDestroy`（`stopSelf`/`stopService` 触发）。与启动者**无关联**，独立存活。
- **绑定态**（`bindService`）：`onCreate` → `onBind` →（客户端拿到 `IBinder`）→ `onUnbind` → `onDestroy`。多个客户端可绑定同一个 Service，全部解绑后销毁。
- 混合使用：先 `startService` 再 `bindService`，销毁时需 `stopService` + `unbindService` 两者都做。
- `onStartCommand` 返回值决定被杀后行为：

| 返回值 | 行为 |
| --- | --- |
| `START_STICKY` | 重建 Service，但 Intent 为 null（适合音乐播放器） |
| `START_NOT_STICKY` | 不重建 |
| `START_REDELIVER_INTENT` | 重建并重传最后一个 Intent（适合下载任务） |

- Service **运行在主线程**，耗时任务必须开线程（或用 `IntentService`/协程）。它既不是新线程也不是新进程（除非显式指定 `android:process`）。
- 8.0 起**后台服务受限** → 用 `startForegroundService` + 5 秒内 `startForeground()` 显示通知，或用 `WorkManager`/`JobScheduler`。
- `IntentService` 已在 API 30 废弃（串行 HandlerThread，执行完自动停止）→ 推荐 `WorkManager`。

## 六、BroadcastReceiver

| 类型 | 特点 |
| --- | --- |
| 标准广播（Normal） | 完全异步，所有接收器几乎同时收到，无法拦截 |
| 有序广播（Ordered） | 按 `android:priority` 串行传递，可 `abortBroadcast()` 拦截、可 `setResultData()` 传递结果 |
| 本地广播 | `LocalBroadcastManager` 已废弃 → 改用 `LiveData`/`SharedFlow`/`ViewModel` |

- 注册方式：
  - **静态注册**（Manifest）：应用未启动也能接收（开机、网络变化）；8.0 起大部分**隐式广播被禁止静态注册**。
  - **动态注册**（`registerReceiver`）：随组件生命周期，必须在 `onDestroy`(或 `onStop`) 中 `unregister`，否则泄漏。
- `onReceive` 在主线程且**最长 10 秒**（超过 ANR），不能做耗时操作；需要异步用 `goAsync()`（配合 `PendingResult.finish()`）或交给 `WorkManager`。
- 安全：加自定义 `permission`、设置 `android:exported="false"` 防止被外部伪造广播。

## 七、ContentProvider

- 跨进程数据共享的**统一抽象层**，以 URI（`content://authority/path/id`）定位数据，通过 `ContentResolver` 调用 `query/insert/update/delete/getType`。
- 底层：**Binder 传递元数据 + 匿名共享内存（Ashmem）/共享内存传递 CursorWindow**（大数据避免拷贝）。
- `onCreate` 运行在**主线程**，且**早于 `Application.onCreate`**（Activity 启动前先 installProvider）——很多 SDK 利用这一点做免侵入初始化（现推荐 Jetpack **App Startup** 收敛）。
- 与 SQLite 的关系：SQLite 是存储引擎，ContentProvider 是**对外共享的封装层**，内部数据源可以是 SQLite、文件、网络。
- 权限：读写可分别声明 `readPermission` / `writePermission`；`grantUriPermission` 临时授权。

## 八、Handler 消息机制

### 8.1 四要素

| 角色 | 职责 |
| --- | --- |
| `Message` | 消息载体（`what`、`arg1/2`、`obj`、`when`、`target`），通过 `obtain()` 复用对象池 |
| `MessageQueue` | 单链表，按 `when` 时间排序；`next()` 无消息时 `epoll_wait` 阻塞 |
| `Looper` | 循环取消息 `loop()`，通过 `ThreadLocal` 保证每线程唯一 |
| `Handler` | 发送（`sendMessage`/`post`）与处理（`handleMessage`） |

```kotlin
Looper.prepare()                       // 创建 Looper + MessageQueue
val handler = object : Handler(Looper.myLooper()!!) { /* handleMessage */ }
Looper.loop()                          // 死循环取消息分发
```

- 主线程 Looper 在 `ActivityThread.main()` 中通过 `Looper.prepareMainLooper()` 创建，**永不退出**。
- `loop()` 是死循环但**不会 ANR**：没有消息时 `nativePollOnce` 使线程休眠释放 CPU，有消息时唤醒。主线程的所有工作（生命周期回调、UI 绘制）都是这个循环里的一条条消息。

### 8.2 进阶机制

- **同步屏障**（`postSyncBarrier`）：插入一条无 target 的消息，屏蔽后续同步消息，只让**异步消息**执行，保证 UI 绘制优先。Choreographer 接收 VSYNC 时就用它抢占主线程，绘制完成后 `removeSyncBarrier`。
- **异步消息**：`Message.setAsynchronous(true)` 或 `Handler(..., async = true)`。
- **`IdleHandler`**：消息队列空闲时执行，适合做延迟初始化。
- **`postDelayed` 不准时**：受前面消息耗时影响；它按 `when` 排队，不保证精确。
- **内存泄漏**：非静态内部类 Handler 隐式持有 Activity → 用静态内部类 + `WeakReference`，并在 `onDestroy` 中 `removeCallbacksAndMessages(null)`。

### 8.3 子线程更新 UI？

- `ViewRootImpl.checkThread()` 校验的是**创建该 ViewRoot 的线程**，绝大多数情况就是主线程。
- 在子线程创建 `Looper` 并添加 Window 理论上可行，但实践上禁止——UI 工具包非线程安全。
- 正确做法：`View.post()`、`Activity.runOnUiThread()`、`Handler(Looper.getMainLooper())`、协程 `Dispatchers.Main`。

## 九、Binder 与 IPC

### 9.1 为什么是 Binder

| 维度 | Binder | 管道/Socket | 共享内存 |
| --- | --- | --- | --- |
| 拷贝次数 | **1 次** | 2 次（用户↔内核↔用户） | 0 次 |
| 安全性 | 内核态校验 UID/PID，实名注册 | 依赖上层协议 | 无 |
| 使用方式 | 面向对象的方法调用 | 字节流 | 需自行处理同步 |

- 一次拷贝原理：Binder 驱动在内核开辟缓冲区，并把它**同时映射**到接收方用户空间（mmap）。发送方 `copy_from_user` 一次到内核缓冲区，接收方因已有映射**无需再拷贝**。

### 9.2 架构与调用

```
Client ──transact──> Binder 驱动 ──> Server(Binder 线程池)
   ↑                                        │
   └──────────── onTransact 返回 ───────────┘
ServiceManager（句柄固定为 0）：实名 Binder 的"DNS"，负责注册与查询
```

- **AIDL** 生成的 `Stub`（服务端，继承 Binder，实现 `onTransact`）与 `Proxy`（客户端，把参数写入 `Parcel` 调 `transact`）。
- 传输限制：**单次事务约 1MB**（Binder 事务缓冲区，异步为其一半），跨进程大数据改用 Ashmem、文件描述符传递或 `CursorWindow`。
- 死亡通知：`linkToDeath` / `DeathRecipient`，服务端进程死亡后客户端能感知并重连。
- 其他 IPC 方式：Bundle（Intent 传参）、文件共享、Messenger（基于 Binder 的串行消息）、AIDL、ContentProvider、Socket。

## 十、View 体系与绘制

### 10.1 三大流程

- `measure`：`onMeasure` → `setMeasuredDimension`。`MeasureSpec`（32 位 int，高 2 位 mode + 低 30 位 size）：
  - `EXACTLY`：精确值或 `match_parent`
  - `AT_MOST`：上限，对应 `wrap_content`（**自定义 View 不处理则 wrap_content 等同 match_parent**）
  - `UNSPECIFIED`：无限制（ScrollView measure 子 View、RecyclerView 测量）
- `layout`：`onLayout` 确定四个顶点位置（`left/top/right/bottom`）。
- `draw`：`drawBackground` → `onDraw`（自己）→ `dispatchDraw`（子 View）→ `onDrawForeground`（滚动条等）。
- 起点：从 `ViewRootImpl.performTraversals()` 开始，逐层向子 View 分发。

### 10.2 刷新机制

- **VSYNC**（60Hz 约 16.6ms 一帧）→ `Choreographer` 收到信号 → 执行 input/animation/traversals → 提交到 `SurfaceFlinger` 合成 → 屏幕显示。
- 一帧内没能完成绘制 → **掉帧/卡顿**（60ms 以上用户明显感知）。
- `invalidate()`：标记脏区，触发 **draw**（下一帧执行）；`postInvalidate()`：子线程版本；`requestLayout()`：触发 **measure + layout + draw**（会向上传递）。

### 10.3 事件分发

```
Activity.dispatchTouchEvent
  → PhoneWindow → DecorView.superDispatchTouchEvent
    → ViewGroup.dispatchTouchEvent
        ├─ onInterceptTouchEvent（ViewGroup 独有，询问是否拦截）
        ├─ 不拦截 → 逆序遍历子 View：child.dispatchTouchEvent
        │             ├─ onTouchListener.onTouch（可由外部消费）
        │             └─ 返回 false → onTouchEvent（消费点击/长按）
        └─ 子 View 都不消费 → 自己 onTouchEvent
              仍不消费 → 向上回传给父容器 → 最终 Activity.onTouchEvent
```

- 一个事件序列：`DOWN` 确定 target 后，后续 `MOVE/UP` 直接发给它（不会再走拦截询问，除非父容器拦截后发 `CANCEL`）。
- **滑动冲突**：
  - 外部拦截法：父容器在 `onInterceptTouchEvent` 中按方向判断（推荐，符合分发机制）。
  - 内部拦截法：子 View 调 `requestDisallowInterceptTouchEvent(true)` 请求父容器不拦截（需父容器 DOWN 时不拦截）。

### 10.4 自定义 View 与 Surface 系

- 要点：`onMeasure` 处理 `wrap_content` 与 `padding`、`onDraw` 中避免 new 对象、支持 `padding/margin`、提供自定义属性（`attrs.xml` + `obtainStyledAttributes`）、注意 `recycle()`。
- `SurfaceView`：拥有**独立 Surface**，可在子线程绘制，不阻塞主线程，适用于相机预览、视频、游戏（不支持平移缩放等普通动画）。
- `TextureView`：同样独立绘制但可作为普通 View 参与动画/变换，需要硬件加速，内存和功耗略高。

## 十一、进程与线程

### 11.1 进程优先级（LMK 回收顺序）

前台进程（正在交互） > 可见进程（可见不可交互） > 服务进程（Service） > 后台进程（onStop 的 Activity） > 空进程（缓存，用于加速启动）。

### 11.2 多进程

- 通过 `android:process=":remote"` 开启；`:xxx` 为私有进程（同名前缀），完整名为全局进程。
- 副作用：**Application 会被创建多次**、静态成员/单例失效、线程同步与 SP 失效（多进程 SP 不可靠）→ 跨进程通信必须用 Binder/AIDL/ContentProvider/MMKV。

### 11.3 线程形态

| 方式 | 说明 |
| --- | --- |
| 主线程（UI 线程） | 处理 UI 与生命周期，禁止耗时 |
| `HandlerThread` | 自带 Looper 的串行线程 |
| `ThreadPoolExecutor` | 复用线程、控制并发（IO 密集 vs CPU 密集分池） |
| `AsyncTask` | 已废弃（串行/并行混乱、内存泄漏、生命周期无感知） |
| 协程 | 结构化并发、可取消、天然切线程（现代首选） |

### 11.4 ANR

| 场景 | 超时 |
| --- | --- |
| 输入事件（触摸/按键） | 5s |
| 前台广播 | 10s（后台 60s） |
| 前台 Service 生命周期 | 20s（后台 200s） |
| ContentProvider 发布 | 10s |

- 本质：**主线程消息队列中被插入的任务不能在规定时间内执行完**。
- 常见原因：主线程做网络/IO/大量计算、主线程等待子线程的锁或 `Thread.join`、Binder 调用阻塞、CPU 被其他进程抢占、`SharedPreferences.apply` 在 `onPause` 等待落盘。
- 排查：`/data/anr/traces.txt`（或 `anr_*.txt`）、Android Studio App Inspection、BlockCanary / Matrix TraceCanary、Perfetto。

## 十二、内存

- **内存泄漏典型场景**：静态变量/单例持有 Activity、未注销的监听器与广播、`Handler` 延迟消息、匿名内部类（`Thread`/`Timer`/`Runnable`）隐式持有外部类、资源未关闭（`Cursor`、流、`Bitmap`）、`WebView` 未销毁、`RxJava` 未 dispose。
- 排查工具：LeakCanary、Android Studio Memory Profiler + Heap Dump、MAT（看 `hprof` 的 GC Root 引用链）。
- **OOM 常见原因**：大图加载（`Bitmap` 直接按像素占用内存，ARGB_8888 每像素 4 字节）、内存泄漏累积、内存抖动（短时间大量临时对象触发频繁 GC 导致卡顿）、线程数超限、FD 泄漏。
- 图片优化：`BitmapFactory.Options.inSampleSize` 采样、`inPreferredConfig = RGB_565`、`inBitmap` 复用内存、`RegionDecoder` 加载长图，配合三级缓存（内存 LruCache + 磁盘 DiskLruCache + 网络）。
- 进程内存上限：`dalvik.vm.heapsize`（常见 128–512MB，可用 `android:largeHeap="true"` 申请更大，但不应依赖）。
- 低内存回调：`onTrimMemory(level)`（配合 Glide 清理缓存）、`onLowMemory()`。

## 十三、存储与网络

### 13.1 存储

| 方式 | 适用 | 注意 |
| --- | --- | --- |
| `SharedPreferences` | 轻量键值（XML） | `apply` 异步、`commit` 同步返回；多进程不可靠；全量加载占内存 |
| `DataStore` | 替代 SP | Preferences/Proto 两种，基于协程 + Flow，解决 ANR 与类型安全 |
| `MMKV` | 高性能 KV | mmap + protobuf，支持多进程 |
| 文件 | 大块数据 | 内部存储（私有）/外部存储（需权限，10+ 分区存储） |
| `Room` | 结构化数据 | SQLite 之上，编译期校验 SQL，返回 Flow/PagingSource |

- 分区存储（Android 10+）：应用只能直接访问自己的目录与媒体集合，公共目录需通过 `MediaStore` 或 SAF。
- 7.0 起 `file://` URI 在跨应用传递会被拒绝 → 用 `FileProvider` 生成 `content://`。

### 13.2 网络

- **OkHttp**：责任链拦截器（应用拦截器 → `RetryAndFollowUp` → `Bridge` → `Cache` → `Connect` → 网络拦截器 → `CallServer`）；连接池复用 TCP/TLS、GZIP、缓存策略、Dispatcher 控制并发。
- **Retrofit**：接口 + 注解，运行时**动态代理**生成实现；`ServiceMethod` 解析注解，`CallAdapter`（适配 RxJava/协程）与 `Converter`（Gson/Moshi）解耦，底层委托 OkHttp。
- HTTPS：TLS 握手（证书链校验、SNI）、`CertificatePinner` 防中间人、自定义 `X509TrustManager` 需谨慎（禁止无条件信任）。
- 序列化：`Serializable`（Java 标准，反射 + 大量临时对象，慢，适持久化）；`Parcelable`（Android 专用，内存序列化，快，适 IPC/Bundle）。
- Bundle 传参有大小限制（约 1MB 的 Binder 事务上限），大对象走共享内存/文件/单例。

## 十四、Jetpack 与架构

- **MVVM**：View（Activity/Fragment/Compose）→ ViewModel（持有 UI 状态与业务调用）→ Model（Repository / 数据源：网络 + 本地）。通过 `StateFlow`/`LiveData` 单向数据流驱动 UI。
- **`Lifecycle`**：`LifecycleOwner`（Activity/Fragment）+ `LifecycleObserver`；用状态机（Event/State）让组件具备生命周期感知能力，自动解注册。
- **`ViewModel`**：配置变更（旋转）后存活，因为实例保存在 `ViewModelStore`，通过 `Activity` 的 `onRetainNonConfigurationInstance` 传递；作用域可为 Activity / Fragment / Navigation 图 / `hiltViewModel()`；配合 `SavedStateHandle` 应对进程重建。
- **`LiveData`**：生命周期感知、粘性分发（新观察者会收到最后一次值）、`setValue`（主线程）/`postValue`（切主线程，会丢中间值）。与 `StateFlow`（必需初始值、无粘性问题更可控）、`SharedFlow`（事件流、可重放配置）对比。
- 其他：
  - `Room`：SQLite 抽象层，编译期生成代码，支持 Flow/Paging3。
  - `WorkManager`：可靠后台任务（约束：网络、充电、延迟；兼容 JobScheduler/GCM/ForegroundService），替代轮询。
  - `Navigation`：单 Activity 多 Fragment 导航，支持 deeplink、Safe Args。
  - `Paging3`：分页加载（`PagingSource` + `RemoteMediator` + `PagingDataAdapter`）。
  - `Hilt`：基于 Dagger 的 Android 依赖注入；轻量替代为 Koin。
  - `ViewBinding`：替代 `findViewById` 与 `ButterKnife`（编译期生成，无反射）；`DataBinding` 支持布局表达式（MVVM 双向绑定）。

## 十五、Jetpack Compose

- 声明式：`@Composable` 函数描述 UI，**状态变化触发重组（Recomposition）**；`remember` 保存重组间的值，`mutableStateOf` 创建可观察状态。
- **状态提升**（State Hoisting）：把状态提到共同父节点，使子组件变成无状态（可复用、可预览）。
- 布局：`Column`/`Row`/`Box`、`Modifier`（尺寸、padding、点击、绘制链式调用）、`LazyColumn/LazyRow`（类 RecyclerView 复用）。
- 副作用 API：`LaunchedEffect`（协程，随 key 变化重启）、`DisposableEffect`（需要清理）、`SideEffect`（每次重组后执行）、`rememberCoroutineScope`、`produceState`。
- 性能：
  - 重组范围最小化：参数是**稳定类型**（`@Stable`/`@Immutable`）且未变化 → 跳过重组。
  - 用 `derivedStateOf` 减少无效重组、`key` 控制重组身份、避免在 Composable 中做重计算（放 `remember`）。
- 互操作：`ComposeView`（在 View 布局中嵌入 Compose）与 `AndroidView`（在 Compose 中嵌入传统 View/Map/广告）。

## 十六、性能优化

| 方向 | 手段 |
| --- | --- |
| 布局 | `ConstraintLayout` 减少层级、`merge`/`include`/`ViewStub`（懒加载）、避免嵌套 `weight`、检查过度绘制（开发者选项） |
| 启动 | 冷/温/热启动区分；`Application` 懒加载与异步初始化；App Startup 收敛 ContentProvider；闪屏主题（`windowBackground`）避免白屏；MultiDex 与 dex 重排 |
| 卡顿 | Systrace/Perfetto 定位主线程耗时；`Choreographer` 帧回调监控掉帧；避免主线程 IO/JSON 解析/数据库查询 |
| 包体积 | R8 混淆压缩、资源混淆（AndResGuard）、WebP 图片、`resConfigs` 裁剪语言、`abiFilters` 裁剪 so、插件化/动态下发、移除无用资源 |
| 内存 | 见上一节；避免内存抖动、及时释放大对象 |
| 功耗/流量 | WorkManager 批量任务、Doze/App Standby 适配、减少唤醒锁与轮询、数据压缩与增量更新 |
| 稳定性 | Crash 兜底（`Thread.setDefaultUncaughtExceptionHandler`）、ANR 监控上报、灰度与热修复（Tinker）、插件化（RePlugin/VirtualApk） |

## 十七、常用三方库

| 领域 | 库 |
| --- | --- |
| 网络 | OkHttp、Retrofit、Moshi/Gson |
| 图片 | Glide、Coil（Kotlin 优先）、Picasso |
| 异步 | Kotlin Coroutines + Flow、RxJava |
| 依赖注入 | Hilt、Koin |
| 数据库 | Room、SQLDelight、MMKV |
| 质量 | LeakCanary、Matrix（腾讯）、Chucker、BlockCanary |
| 事件通信 | `SharedFlow`、`EventBus`（历史项目常见） |

## 十八、版本演进要点

| 版本 | 关键变化 |
| --- | --- |
| 5.0 | ART 取代 Dalvik、Material Design |
| 6.0 | **运行时权限**（危险权限需动态申请） |
| 7.0 | 分屏、V2 签名、**禁止对外暴露 `file://`**（FileProvider） |
| 8.0 | 后台服务限制、**通知渠道**、未知来源安装权限 |
| 9.0 | 默认禁用明文 HTTP（`networkSecurityConfig`）、V3 签名 |
| 10 | **分区存储**、深色主题、手势导航 |
| 11 | 单次授权、包可见性（`<queries>`）、后台位置单独授权 |
| 12 | 组件必须显式声明 `android:exported`、SplashScreen API |
| 13 | 通知运行时权限 `POST_NOTIFICATIONS`、照片选择器、按语言设置 |
| 14+ | 更严格的后台启动限制、精确闹钟权限默认关闭、16KB Page Size 兼容要求 |

## 十九、推荐资料

- [Android Developers 官方文档](https://developer.android.com/)
- [Android 源码阅读（AOSP / cs.android.com）](https://cs.android.com/)
- 《第一行代码 Android（第 3 版）》
- 《Android 开发艺术探索》—— 组件、View、消息机制、IPC 原理
- [Now in Android（Google 官方架构示例）](https://github.com/android/nowinandroid)

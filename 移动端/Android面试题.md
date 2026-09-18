# Android 面试题

> Android 高频面试题整理：组件与生命周期 → 消息机制 → IPC/Binder → View 与绘制 → 线程/进程/ANR → 内存与性能 → 存储网络 → 架构/Jetpack/Compose → 开放题。每题为「考察点 + 回答要点」。

## 一、Activity 与 Fragment

### 1.1 Activity 生命周期？每个回调适合做什么？

- **考察点**：生命周期顺序、各回调职责边界。
- **要点**：
  - `onCreate`：初始化（setContentView、ViewModel、binding、恢复 savedInstanceState），只执行一次。
  - `onStart`：可见但不可交互；注册需要"可见才活跃"的监听。
  - `onResume`：可交互、获取焦点；开启动画/相机/定位。
  - `onPause`：**必须轻量**，因为下一个 Activity 要等它返回才启动；做暂停动画、停止传感器。
  - `onStop`：完全不可见；释放较重资源、取消网络回调。
  - `onDestroy`：最终清理；用 `isFinishing()` 区分"主动 finish"与配置变更。
  - 配对原则：在 `onStart/onStop`、`onResume/onPause` 中成对注册与注销，避免泄漏或浪费。

### 1.2 A 打开 B 再返回，完整的生命周期顺序？

- **考察点**：对 onPause/onStop 与前一个 Activity 启动时序的理解。
- **要点**：
  - 打开 B：`A.onPause` → `B.onCreate` → `B.onStart` → `B.onResume` → `A.onStop`。
  - 返回 A：`B.onPause` → `A.onRestart` → `A.onStart` → `A.onResume` → `B.onStop` → `B.onDestroy`。
  - 为什么先 `A.onPause`：保证旧页面先停止交互、保存状态，新页面才能启动。
  - B 若是**透明/对话框主题**，A 只到 `onPause`，不会 `onStop`。

### 1.3 `onSaveInstanceState` 什么时候调用？与 ViewModel 的区别？

- **考察点**：状态保存机制与数据恢复时机。
- **要点**：
  - 触发：非正常销毁（旋转、语言切换、内存回收、开发者选项"不保留活动"）；`onStop` 前后（API 28+ 保证在 `onStop` 之后）。**用户主动 finish/back 不触发**。
  - 存：临时 UI 状态（输入文本、滚动位置、Tab 索引），写入 Bundle；恢复在 `onCreate` 或 `onRestoreInstanceState`。
  - 限制：Bundle 走 Binder，**容量约 1MB**，只能放可序列化数据。
  - ViewModel：跨配置变更存活（存放在 `ViewModelStore`），但**进程被杀会丢失**；大数据/跨进程重建需 `SavedStateHandle` 或本地持久化。
  - 结论：轻量 UI 状态 → Bundle；业务数据 → ViewModel + Repository（本地缓存）。

### 1.4 Activity 的四种启动模式？

- **考察点**：任务栈与复用语义。
- **要点**：
  - `standard`：默认，每次新建实例压入当前栈。
  - `singleTop`：栈顶复用，回调 `onNewIntent`（通知跳转、防重复打开）。
  - `singleTask`：栈内复用，并把其上的 Activity **全部出栈**（主页、登录页）。
  - `singleInstance`：独占一个任务栈，全局唯一（来电界面）。
  - 优先级：**Intent Flag > manifest launchMode**；常用 Flag：`NEW_TASK`、`CLEAR_TOP`、`SINGLE_TOP`。
  - `taskAffinity` + `allowTaskReparenting` 决定归属栈。

### 1.5 Activity、Window、View 三者的关系？

- **考察点**：窗口体系层级。
- **要点**：
  - `Activity` 是生命周期与交互的控制者，不直接绘制 UI。
  - `PhoneWindow`（Window 的唯一实现）承载视图，持有 `DecorView`。
  - `DecorView` 是顶层 View（含状态栏区域 + `content` 的 `FrameLayout`），`setContentView` 的布局被添加到 `android.R.id.content`。
  - `ViewRootImpl` 不是 View，是连接 `WindowManager` 与 `View` 的桥梁，负责 `performTraversals`（measure/layout/draw）与事件分发入口。
  - 一个 Activity 对应一个 Window（`Dialog`、`Toast`、`PopupWindow` 会各自创建 Window）。

### 1.6 Fragment 生命周期与 Activity 的关系？如何通信？

- **考察点**：宿主驱动生命周期 + 现代通信方式。
- **要点**：
  - 顺序：`onAttach` → `onCreate` → `onCreateView` → `onViewCreated` → `onStart` → `onResume` → ... → `onDestroyView` → `onDestroy` → `onDetach`。
  - Fragment 的生命周期**由 Activity 驱动**；Activity resumed 后 Fragment 才能 resumed。
  - `onDestroyView` 之后 View 被销毁但 Fragment 实例仍在 → **ViewBinding 必须置空**，否则泄漏。
  - 传参：只能用 `arguments`（Bundle）+ `newInstance()` 工厂方法，重建时带参构造会丢失。
  - 通信：**共享 Activity 作用域的 ViewModel（首选）**、Fragment Result API（`setFragmentResultListener`）、接口回调（旧，容易泄漏）。

### 1.7 `commit()` 与 `commitAllowingStateLoss()` 的区别？

- **考察点**：状态保存后提交事务的异常问题。
- **要点**：
  - `commit()`：异步提交（post 到主线程队列），若在 `onSaveInstanceState` 之后调用会抛 `IllegalStateException: Can not perform this action after onSaveInstanceState`——因为该事务可能在重建时丢失。
  - `commitAllowingStateLoss()`：允许提交，但状态可能**不被保存**（恢复后 UI 状态不一致），需自行承担风险。
  - 规避：在 `onCreate`/`onResume` 等明确时机提交；或用 `commitNow()`（同步执行，不能入回退栈）。

## 二、Service / Broadcast / ContentProvider

### 2.1 Service 的两种启动方式？生命周期？

- **考察点**：启动态 vs 绑定态。
- **要点**：
  - `startService`：`onCreate` → `onStartCommand`（每次 start 都回调）→ `onDestroy`；与启动者解耦，需 `stopSelf`/`stopService` 停止。
  - `bindService`：`onCreate` → `onBind` → 客户端拿到 `IBinder` → `onUnbind` → `onDestroy`；所有客户端解绑后销毁。
  - 混合启动：先 start 再 bind，销毁需**同时** stop 与 unbind。
  - `onStartCommand` 返回 `START_STICKY`（重建但 intent 为 null）/ `START_NOT_STICKY`（不重建）/ `START_REDELIVER_INTENT`（重建并重传 intent）。

### 2.2 Service 是运行在子线程吗？和 Thread 的区别？

- **考察点**：最常见的误区。
- **要点**：
  - **Service 运行在主线程（UI 线程）**，它既不是新线程也不是新进程（除非显式声明 `android:process`）。
  - 在 Service 里直接做耗时操作同样会 ANR。
  - Thread 是执行单元，生命周期不随组件；Service 是**有生命周期的系统组件**，可被 AMS 管理、可被其他进程绑定。
  - 正确姿势：`IntentService`（已废弃）/ `HandlerThread` / 线程池 / 协程 `viewModelScope` / `WorkManager`。
  - 8.0 起后台服务受限：用 `startForegroundService` + 5 秒内 `startForeground()` 通知，或用 WorkManager。

### 2.3 广播的注册方式有什么区别？

- **考察点**：静态/动态注册与 8.0 限制。
- **要点**：
  - 静态（Manifest）：应用未启动也能接收，常驻；8.0 起**大部分隐式广播禁止静态注册**（保留开机、时区等白名单）。
  - 动态（`registerReceiver`）：随组件生命周期，必须在 `onStop/onDestroy` 中 `unregister`，否则泄漏与空指针。
  - `onReceive` 在主线程，**10 秒超时即 ANR** → 耗时用 `goAsync()` 或 WorkManager。
  - 本地广播 `LocalBroadcastManager` 已废弃 → 用 `SharedFlow`/`ViewModel`。
  - 安全：`android:exported="false"`、自定义权限，防止被伪造广播攻击。

### 2.4 ContentProvider 的作用与原理？

- **考察点**：跨进程数据共享。
- **要点**：
  - 统一数据访问接口（URI + CRUD），屏蔽底层存储细节，配合 `ContentResolver` 使用。
  - 底层是 Binder（控制与元数据）+ 匿名共享内存（`CursorWindow` 传大数据，避免拷贝）。
  - `onCreate` 在主线程，且**早于 `Application.onCreate`**——被很多 SDK 用作免侵入初始化入口（现推荐 Jetpack App Startup 统一收敛，避免启动变慢）。
  - 区别：SQLite 是存储引擎，ContentProvider 是共享抽象层。
  - 权限：`readPermission`/`writePermission`、`grantUriPermission` 临时授权。

### 2.5 `Context` 有哪几种？区别？

- **考察点**：Context 继承体系与误用。
- **要点**：
  - `ContextWrapper` → `ContextThemeWrapper` → `Activity`；`Application`、`Service` 直接继承 `ContextWrapper`。
  - `Activity` 的 Context 带主题、可弹 Dialog、可启动 Activity（需 `FLAG_ACTIVITY_NEW_TASK` 除外）；`Application` Context 的生命周期最长，**不能弹 Dialog**（无窗口 token，会 BadTokenException）。
  - 判断：`getApplicationContext()` 用于长生命周期对象（单例、数据库），避免持有 Activity 造成泄漏。
  - 数量：Context 数量 = Activity 数 + Service 数 + 1（Application）。

## 三、Handler / 消息机制

### 3.1 Handler 机制的原理？四个角色？

- **考察点**：消息循环模型。
- **要点**：
  - `Handler` 发送 → `MessageQueue.enqueueMessage`（按 `when` 插入单链表）→ `Looper.loop()` 循环 `next()` 取消息 → 回调 `msg.target.dispatchMessage` → `handleMessage`。
  - `Looper` 通过 `ThreadLocal` 保证**每线程唯一**；主线程在 `ActivityThread.main()` 中 `prepareMainLooper()`。
  - `Message` 用 `obtain()` 从对象池复用（`sPool` 链表），避免频繁分配。
  - `Looper.loop()` 死循环但**不卡死**：无消息时 `nativePollOnce` 让线程阻塞休眠并释放 CPU。
  - 主线程所有行为（生命周期、UI 绘制、点击事件）都是这个循环里的一条条消息。

### 3.2 为什么主线程不会因为 Looper 死循环卡死（ANR）？

- **考察点**：对 epoll 与事件驱动的理解。
- **要点**：
  - 没有消息时，`MessageQueue.next()` 调用 `nativePollOnce`，底层是 **epoll_wait**，主线程进入休眠状态并释放 CPU。
  - 有新消息时通过 `nativeWake` 写入 pipe 唤醒。
  - ANR 的真正原因是**某条消息处理超过规定时间**（输入 5s、广播 10s 等），而不是 loop 本身。
  - 主线程是"事件驱动"的，阻塞在休眠态是正常且必要的。

### 3.3 Handler 造成内存泄漏的原因与解决？

- **考察点**：内部类持有外部引用 + 消息队列生命周期。
- **要点**：
  - 非静态内部类/匿名 `Handler` 隐式持有外部 `Activity` 引用；延迟消息 `Message.target` 持有 Handler，而 MessageQueue 生命周期长于 Activity → Activity 无法被回收。
  - 解决：静态内部类 + `WeakReference<Activity>`；`onDestroy` 中 `removeCallbacksAndMessages(null)`；用 `Lifecycle` 或 `viewLifecycleOwner.lifecycleScope` 自动取消。
  - 同理：`Thread`、`Timer`、`RxJava` 订阅、匿名监听器都有同样问题。

### 3.4 同步屏障与异步消息是什么？

- **考察点**：UI 绘制优先级的保障机制。
- **要点**：
  - `postSyncBarrier()` 插入一条 `target == null` 的消息作为屏障，之后的**同步消息被暂时屏蔽**，只有异步消息能执行。
  - 应用：`Choreographer` 收到 VSYNC 后先投递异步消息，保证 UI 绘制优先于普通消息，绘制完成后 `removeSyncBarrier`。
  - 普通 Handler 通过 `Message.setAsynchronous(true)` 或构造时传 `async=true` 发送异步消息（一般应用开发不需要）。

### 3.5 子线程能不能更新 UI？`View.post` 的原理？

- **考察点**：线程检查与消息投递。
- **要点**：
  - `ViewRootImpl.checkThread()` 校验的是**创建 ViewRootImpl 的线程**（一般是主线程），违反会抛 `CalledFromWrongThreadException`。
  - 理论上在子线程创建 Looper 并添加 Window 也能更新 UI，但 UI 工具包非线程安全，**禁止这么做**。
  - 正确方式：`View.post()`、`runOnUiThread`、`Handler(Looper.getMainLooper())`、协程 `Dispatchers.Main`。
  - `View.post` 原理：已有 `AttachInfo` 时直接用主线程的 `mHandler.post`；未 attach 时先存入 `HandlerActionQueue`，等 `dispatchAttachedToWindow` 时统一执行——这也是它能"拿到 view 宽高"的原因（在 measure/layout 之后执行）。

## 四、Binder / IPC

### 4.1 Android 为什么用 Binder 而不是 Socket/共享内存？

- **考察点**：性能、安全、易用性权衡。
- **要点**：
  - **性能**：数据只拷贝一次（对比管道/消息队列/Socket 两次），仅次于共享内存。
  - **安全**：驱动层在内核态为每次通信加上 UID/PID，服务端可校验身份；Socket 只能靠上层协议。
  - **稳定/易用**：C/S 架构、面向对象的方法调用（像调用本地方法），支持死亡通知（linkToDeath）；共享内存无同步机制、易产生竞态。

### 4.2 Binder 一次拷贝是怎么做到的？

- **考察点**：mmap 映射原理。
- **要点**：
  - Binder 驱动在内核空间创建**数据接收缓冲区**，并通过 `mmap` 把它同时映射到**接收方用户空间**。
  - 发送方 `copy_from_user` 把数据从用户空间拷贝到内核缓冲区（**唯一一次拷贝**）。
  - 接收方因为已有地址映射，直接读取，**无需第二次拷贝**。

### 4.3 描述一次 Binder 通信的完整过程

- **要点**：
  1. Server 通过 `ServiceManager`（句柄固定为 0）**注册**实名 Binder。
  2. Client 向 ServiceManager **查询**拿到代理对象（handle）。
  3. Client 调用 Proxy 方法 → 数据写入 `Parcel` → `mRemote.transact()` → 陷入内核。
  4. Binder 驱动根据 handle 找到目标 Binder 实体，把数据拷贝到目标进程的缓冲区，唤醒 **Binder 线程池**中的线程。
  5. Server 端 `onTransact` 解包 → 执行真实方法 → 把结果写回 `reply` → 驱动返回给 Client（Client 线程阻塞等待）。
  6. Client 从 `reply` 中读取返回值。

### 4.4 AIDL 生成了什么？支持哪些数据类型？

- **考察点**：Stub/Proxy 结构与跨进程类型限制。
- **要点**：
  - 生成接口、`Stub`（`extends Binder implements 接口`，实现 `onTransact` 分发并调用真实方法）、`Proxy`（实现接口，`transact` 写入参数并读回结果）。
  - `asInterface`：同进程返回 Stub 本身，跨进程返回 Proxy。
  - 支持类型：基本类型、`String`、`CharSequence`、`List`/`Map`（元素也必须受支持）、`Parcelable`、AIDL 接口。
  - 定向 tag：`in`（客户端→服务端）、`out`、`inout`（`in` 最省，非必要不用 `inout`）。
  - **单次事务大小约 1MB**（异步为其一半），大数据走 Ashmem / 传文件描述符 / ContentProvider。

### 4.5 Android 有哪些 IPC 方式？怎么选？

- **要点**：
  - `Bundle`（Intent 传参，限大小）、文件共享（并发不可靠）、`Messenger`（串行，基于 Binder）、
  - **AIDL**（并行、复杂接口、跨进程方法调用）、`ContentProvider`（数据增删改查，天然支持大数据）、`Socket`（跨设备/长连接）。
  - 选型：同进程 → 事件总线/`SharedFlow`；跨进程小数据 → Bundle/Messenger；跨进程大数据 → ContentProvider/Ashmem；需要双向方法调用 → AIDL。

## 五、View 与绘制

### 5.1 View 的绘制流程？MeasureSpec 三种模式？

- **考察点**：三大流程与测量规格。
- **要点**：
  - `measure` → `layout` → `draw`，由 `ViewRootImpl.performTraversals()` 发起并自顶向下分发。
  - `MeasureSpec` = 32 位 int（高 2 位 mode + 低 30 位 size）：
    - `EXACTLY`：精确值或 `match_parent`
    - `AT_MOST`：上限，对应 `wrap_content`
    - `UNSPECIFIED`：无限制（如 RecyclerView/ScrollView 测量子 View）
  - 父 View 的 `MeasureSpec` + 子 View 的 `LayoutParams` 共同决定子 View 的 `MeasureSpec`（`getChildMeasureSpec`）。
  - **自定义 View 直接继承 View 时必须处理 `AT_MOST`**，否则 `wrap_content` 表现等同 `match_parent`。

### 5.2 事件分发机制？

- **考察点**：责任链传递顺序。
- **要点**：
  - `Activity.dispatchTouchEvent` → `PhoneWindow` → `DecorView` → `ViewGroup.dispatchTouchEvent`。
  - ViewGroup 先 `onInterceptTouchEvent`（仅 ViewGroup 有）；不拦截则**逆序**遍历子 View（后添加/上层优先），判断触点是否在子 View 区域内 → `child.dispatchTouchEvent`。
  - View 内部：`onTouchListener.onTouch` 优先于 `onTouchEvent`；`onTouchEvent` 中处理 click/longclick；`onTouch` 返回 true 则不再走 `onTouchEvent`（onClick 失效）。
  - 无人消费 → 回传给父容器的 `onTouchEvent` → 最终 `Activity.onTouchEvent`。
  - DOWN 确定 target 后，后续 MOVE/UP 直接发到该 View；父容器后来拦截会先发送 `ACTION_CANCEL`。
  - 拦截方法只在 ACTION_DOWN 或有 target 的 MOVE 中被询问；`requestDisallowInterceptTouchEvent` 可让子 View 禁止父容器拦截。

### 5.3 滑动冲突怎么解决？

- **考察点**：外部拦截法 / 内部拦截法。
- **要点**：
  - 场景：内外层滑动方向不一致（ViewPager + ListView）、方向一致（ScrollView + 嵌套列表）、多层嵌套。
  - **外部拦截法（推荐）**：父容器重写 `onInterceptTouchEvent`，DOWN 必须返回 false（否则后续事件全归父），MOVE 时按 dx/dy 与方向阈值决定是否拦截，UP 返回 false。
  - **内部拦截法**：子 View 在 `dispatchTouchEvent` 中根据条件调 `parent.requestDisallowInterceptTouchEvent(true/false)`；父容器需在 DOWN 时不拦截（否则子收不到事件）。
  - 现代方案：`NestedScrollingParent/Child`（嵌套滚动机制）、RecyclerView + CoordinatorLayout。

### 5.4 `invalidate` 与 `requestLayout` 的区别？

- **要点**：
  - `invalidate` / `postInvalidate`（子线程版）：标记脏区，下一帧只走 **draw**；适合纯重绘（颜色、进度）。
  - `requestLayout`：触发 **measure + layout + draw**，并沿 View 树向上传递（`mParent.requestLayout` 一直到 ViewRootImpl）；适合尺寸/位置变化。
  - 频繁调用 `requestLayout` 开销大；更改大小后通常只需 `requestLayout`（内部也会触发绘制）。
  - 两者都必须在主线程调用（`postInvalidate` 内部通过 Handler 切回主线程）。

### 5.5 `getWidth()` 在 `onCreate` 中为 0，怎么获取？

- **考察点**：View 绘制与 Activity 生命周期不同步。
- **要点**：
  - 原因：`onCreate` 时 View 尚未 measure/layout，且 Activity 生命周期与 View 绘制是两条不同的消息。
  - 方案：
    1. `View.post { ... }`（等 attach 后投递，执行时已完成布局）
    2. `ViewTreeObserver.OnPreDrawListener` / `OnGlobalLayoutListener`（记得移除）
    3. `doOnPreDraw`（Android KTX）
    4. `onWindowFocusChanged(hasFocus)`（窗口获得焦点时布局已完成）

### 5.6 SurfaceView 与 View 的区别？

- **要点**：
  - 普通 View 在**主线程**的共享 Surface（窗口）上绘制，受 VSYNC 驱动，适合常规 UI。
  - `SurfaceView` 拥有**独立的 Surface**，由 SurfaceFlinger 单独合成，可在子线程绘制、不阻塞主线程，适合相机预览、视频、游戏；缺点：不支持普通变换/动画（其位置变更需用 Window 层合成）。
  - `TextureView`：可作为普通 View 参与动画、变换，但需要硬件加速，内存与功耗略高。

## 六、线程 / 进程 / ANR

### 6.1 ANR 是什么？有哪些场景和超时？

- **考察点**：主线程阻塞的判定标准。
- **要点**：
  - ANR = Application Not Responding，即主线程消息队列中的任务**在规定时间内没被执行完**。
  - 超时：输入事件 **5s**；前台广播 **10s**（后台 60s）；前台 Service 生命周期 **20s**（后台 200s）；ContentProvider 发布 10s。
  - 触发路径：AMS 定时投递"ANR 消息"，任务按时完成则取消该消息，否则弹出 ANR 对话框并写 `traces.txt`。
  - 常见原因：主线程网络/IO/大计算、主线程等锁或 `join`、Binder 调用被阻塞、`SharedPreferences.apply` 在 `onPause` 同步等待落盘、CPU 饥饿。

### 6.2 如何排查 ANR？

- **要点**：
  - 取 `/data/anr/traces.txt`（或 `anr_*.txt`）看主线程堆栈卡在哪一行（注意"主线程空闲"可能是 Binder 阻塞或 CPU 被抢）。
  - 结合 `Logcat` 的 `ANR in` 行、CPU 使用率、iowait 判断是被阻塞还是资源不足。
  - 线上监控：`BlockCanary`、腾讯 `Matrix` TraceCanary（监听 Looper 的 `Printer` 与 Choreographer 掉帧）、`StrictMode`（磁盘/网络操作探测）。
  - 治理：异步化、降低锁粒度、避免主线程 Binder 调用、SP 改 DataStore/MMKV、启动任务分级延迟。

### 6.3 Android 有哪些线程/异步方式？

- **要点**：
  - `Thread` / `HandlerThread`（自带 Looper 的串行线程）/ `ThreadPoolExecutor`（分 IO 与 CPU 池）/ `AsyncTask`（已废弃）/ `IntentService`（已废弃）。
  - 现代首选：**协程**（`viewModelScope`/`lifecycleScope` + `Dispatchers`），结构化并发、可取消、无回调地狱；`Flow` 做数据流。
  - 需可靠执行（进程死亡也要跑）→ `WorkManager`；定时后台任务 → `AlarmManager`/`WorkManager`。

### 6.4 多进程会带来什么问题？

- **要点**：
  - `Application.onCreate` 会执行多次 → 初始化要按进程名判断，否则重复初始化（数据库、推送 SDK）。
  - 静态变量、单例、内存缓存**在每个进程各自一份**，不共享；`SharedPreferences` 多进程不可靠。
  - 线程同步锁、单例模式失效；调试断点需注意附加进程。
  - 跨进程通信必须用 Binder/AIDL/ContentProvider/MMKV/文件。

### 6.5 应用进程优先级与回收顺序？

- **要点**：前台进程 > 可见进程 > 服务进程 > 后台进程 > 空进程。
- 内存不足时 `Low Memory Killer` 按 `oom_adj` 从低优先级开始回收；提高优先级手段：前台服务通知、可见 Notification、`android:persistent`（系统应用）。

## 七、内存与性能

### 7.1 常见的内存泄漏场景与排查？

- **考察点**：引用链分析能力。
- **要点**：
  - 静态变量/单例持有 Activity 或 View；未反注册的监听器/广播；`Handler` 延迟消息；匿名内部类（`Thread`/`Timer`/`Runnable`）隐式持有外部类；资源未关闭（`Cursor`、流、`Bitmap`）；`WebView` 未销毁；属性动画未 `cancel`；RxJava 未 `dispose`。
  - 排查：**LeakCanary**（自动 dump + 引用链）、Android Studio Profiler 抓 Heap Dump、MAT 分析 GC Root（看 `hprof` 的 shortest path）。
  - 修复：弱引用、生命周期感知（在 `onDestroyView` 解绑）、及时 cancel/dispose/close。

### 7.2 OOM 的原因与 Bitmap 优化？

- **要点**：
  - 原因：大图加载、泄漏累积、内存抖动（频繁 GC 导致卡顿）、线程数超限、FD 泄漏、dex/so 过大。
  - Bitmap 内存 = 宽 × 高 × 每像素字节（ARGB_8888 = 4B，RGB_565 = 2B），与文件大小无关。
  - 优化：`inSampleSize` 按目标尺寸采样、`inPreferredConfig = RGB_565`（无透明度场景）、`inBitmap` 内存复用、`BitmapRegionDecoder` 加载长图、及时 `recycle`（API 33 后 `recycle` 已弃用，交给 GC）、使用 Glide/Coil（自动按控件尺寸采样 + 生命周期感知）。
  - 进程内存上限 `dalvik.vm.heapsize`（`android:largeHeap` 只是申请更大，不是解药）。

### 7.3 启动优化怎么做？

- **要点**：
  - 区分冷启动（进程不存在，需创建进程 + Application + Activity）、温启动、热启动。
  - 视觉优化：`windowBackground` 设置闪屏主题 → 消除白屏/黑屏。
  - 代码优化：Application 中任务**分级**（必须同步 / 可异步 / 可延迟 / 懒加载）；用 Jetpack **App Startup** 收敛多个 ContentProvider 初始化；`IdleHandler` 空闲执行；线程收敛与避免锁竞争。
  - 其他：MultiDex 优化、类预加载、抑制 GC、避免启动时大量 I/O 与 JSON 解析。
  - 度量：`adb shell am start -W`、`reportFullyDrawn()`、Perfetto/Systrace。

### 7.4 卡顿如何定位与优化？

- **要点**：
  - 指标：60Hz 下每帧 16.6ms；`Choreographer.FrameCallback` 统计掉帧（>16.6ms/掉帧数）。
  - 工具：Systrace / Perfetto（看主线程各段耗时、锁等待、CPU 频率）、`Looper` Printer 监控、`FrameMetricsAggregator`。
  - 常见原因：主线程 IO/DB/JSON、复杂布局（层级深、过度绘制）、频繁 `requestLayout`、GC、动画与列表滑动时大量对象创建。
  - 优化：布局扁平化（ConstraintLayout、`merge`/`ViewStub`）、列表复用与 `DiffUtil`、预加载与分页、耗时任务异步、关闭过度绘制。

### 7.5 包体积怎么瘦身？

- **要点**：
  - 代码：R8/ProGuard 压缩混淆、`minifyEnabled`、移除无用库与重复功能库、Kotlin 体积优化。
  - 资源：`shrinkResources`、WebP/矢量图替代 PNG、`resConfigs` 只保留必要语言、AndResGuard 资源混淆、`abiFilters` 裁剪 so（或 App Bundle 按 ABI 分发）。
  - 交付：**Android App Bundle（AAB）按需分发**、动态功能模块（Dynamic Feature）、插件化/远端下发。
  - 治理：资源重复检测、图片压缩流水线、依赖治理与大小监控告警。

## 八、存储与网络

### 8.1 SharedPreferences 的问题？与 DataStore 的区别？

- **考察点**：SP 的坑（面试常问）。
- **要点**：
  - **跨进程不可靠**（无跨进程同步保证）；**全量加载**到内存，大文件会占内存且读取慢。
  - `commit()` 同步写盘并返回结果（主线程调用可能卡顿）；`apply()` 异步但会把写入任务放进 `QueuedWork`，在 `onPause/onStop` 时调用 `QueuedWork.waitToFinish()` **等待其完成** → 可能引发 ANR。
  - 类型不安全、无异常提示、不支持事务/迁移。
  - `DataStore`：基于**协程 + Flow**，异步写入、无 ANR 风险、类型安全（Proto DataStore）、支持事务与错误处理，官方推荐替代 SP。
  - 高性能跨进程 KV：MMKV（mmap + protobuf）。

### 8.2 OkHttp 的拦截器链？

- **要点**：
  - `RetryAndFollowUp`（重试/重定向）→ `Bridge`（补 header、Cookie、GZIP）→ `Cache`（缓存命中则直接返回）→ `Connect`（获取连接/建连）→ 网络拦截器 → `CallServer`（真正写请求读响应）。
  - **应用拦截器**：只执行一次、可拿到最终请求（含重定向后）、不关心中间响应；**网络拦截器**：可看到每次重试、缓存未命中时才执行。
  - 连接池复用 TCP/TLS（`ConnectionPool`，默认 5 个空闲连接保活 5 分钟）、GZIP 透明解压、基于 HTTP 头（Cache-Control）的缓存策略。

### 8.3 Retrofit 的原理？

- **要点**：
  - 接口 + 注解声明 → 运行时通过 **`Proxy.newProxyInstance` 动态代理**生成实现类。
  - 调用方法时通过 `ServiceMethod` 解析注解（请求方式、路径、参数、header），构造 `Request`，交给 OkHttp 执行。
  - `CallAdapter` 决定返回类型（`Call`/`Observable`/`suspend` 函数）；`Converter` 负责序列化/反序列化（Gson/Moshi/Kotlinx）。
  - Kotlin 协程支持：给方法加 `suspend`，内部转为 `Call.enqueue` 的挂起实现。

### 8.4 Serializable 与 Parcelable 的区别？

- **要点**：
  - `Serializable`：Java 标准，使用反射、产生大量临时对象，性能低，适合**持久化/网络传输**。
  - `Parcelable`：Android 专用，内存序列化（写入 Parcel），**性能高数倍**，适 IPC/Bundle 传参；实现稍繁琐（Kotlin 用 `@Parcelize` 一行搞定）。
  - Bundle 传参受 Binder 事务大小限制，避免传大对象（用 ID + 本地缓存、或共享内存）。

### 8.5 HTTPS 握手过程与客户端需要注意什么？

- **要点**：
  - 握手：ClientHello（支持的 TLS 版本、加密套件、随机数）→ ServerHello（选定套件、随机数）+ 证书链 → 客户端校验证书（有效期、CA 链、域名 SNI）→ 密钥交换（ECDHE）→ 生成会话密钥 → 切换加密通信。
  - 注意：**禁止重写 TrustManager 无条件信任所有证书**（中间人攻击）；如需自签证书用 `networkSecurityConfig` 限定 debug 或指定 CA。
  - 可选证书锁定 `CertificatePinner`；关注 TLS 1.2/1.3 与套件兼容。

## 九、Jetpack / 架构 / Compose

### 9.1 ViewModel 为什么能在旋转屏幕后存活？

- **考察点**：跨配置变更保存机制。
- **要点**：
  - Activity 因配置变更销毁重建时，`ActivityClientRecord` 中的 **`NonConfigurationInstances`**（含 `ViewModelStore`）由 AMS 侧保留并传给新 Activity。
  - 因此 `ViewModelStore` 里的 ViewModel 实例不重建，只重建 Activity 与 View；当 Activity **真正 finish（非配置变更）**时 ViewModel 被 `clear()`，回调 `onCleared`。
  - 进程被系统杀死时 ViewModel 也不存在 → 用 `SavedStateHandle` 保存关键状态（写入 Bundle）。
  - 作用域：Activity、Fragment、Navigation 图（`hiltNavGraphViewModel`），按需选择以避免过度共享。

### 9.2 LiveData 的"粘性"是什么？与 StateFlow/SharedFlow 怎么选？

- **要点**：
  - LiveData 内部有 `mVersion`，观察者注册时会把**最新值**立即分发一次 → 新观察者收到历史数据（这就是粘性），适合状态但会导致"事件被重复消费"（Toast/导航弹两次）。
  - `setValue` 主线程、`postValue` 切主线程（连续 post 只保留最后一个值）。
  - `StateFlow`：必须有初始值、值相等（`==`）不分发、天然防重复；配合 `repeatOnLifecycle` 收集，适合 UI 状态。
  - `SharedFlow`：可配置 `replay`、`extraBufferCapacity`，适合一次性事件（导航、Toast）；或用 `Channel`。
  - 事件去重方案：`SingleLiveEvent`（内部 Boolean flag）、`Event wrapper`（标记 consumed）、`MutableSharedFlow(replay = 0)`。

### 9.3 Lifecycle 是怎么实现生命周期感知的？

- **要点**：
  - `LifecycleOwner`（Activity/Fragment 实现）+ `LifecycleRegistry`；`ComponentActivity` 通过 `ReportFragment`（或 `ActivityLifecycleCallbacks`）把生命周期事件分发给 `LifecycleRegistry`。
  - 内部用状态机（Event ↔ State：`INITIALIZED`/`CREATED`/`STARTED`/`RESUMED`/`DESTROYED`），并把观察者按状态对齐（`addObserver` 时会补齐事件）。
  - 应用：`LiveData` 自动解注册、`lifecycleScope` 自动取消协程、`repeatOnLifecycle` 控制收集时机。

### 9.4 MVVM 与 MVP/MVC 的区别？

- **要点**：
  - MVC：Activity 同时承担 View 和 Controller，职责重、易臃肿。
  - MVP：Presenter 持有 View 接口，解耦但需手动处理生命周期与内存泄漏，接口膨胀。
  - **MVVM**：ViewModel + 数据驱动（LiveData/StateFlow）观察更新，**不持有 View 引用**，配合生命周期感知自动解绑，是 Google 官方推荐架构。
  - 现代分层：UI（Activity/Fragment/Compose）→ ViewModel → UseCase（可选）→ Repository → 数据源（Remote/Local）。单向数据流（UDF）：事件向上、状态向下。

### 9.5 Compose 的重组是什么？如何优化？

- **考察点**：声明式 UI 性能。
- **要点**：
  - 重组：状态变化时，只重新执行**读取了该状态**的 Composable 函数并更新对应的布局节点（跳过未受影响的部分）。
  - 跳过条件：参数类型是**稳定类型**（`@Stable`/`@Immutable`，或全部由稳定类型组成的 data class）且**与上次相等** → 跳过重组。
  - 优化手段：
    - 用 `remember` 缓存计算结果，`derivedStateOf` 合并高频状态变化减少重组。
    - 列表用 `LazyColumn` 并设置稳定 `key`。
    - 把 lambda 提到 Composable 外或用 `remember` 包裹，避免每次重建实例。
    - 状态提升 + 无状态组件，缩小重组范围；避免在 Composable 中做重计算或 IO。
  - 副作用：`LaunchedEffect`（协程，key 变化重启）、`DisposableEffect`（清理）、`SideEffect`、`rememberCoroutineScope`。

### 9.6 Compose 与 View 体系如何互操作？

- **要点**：
  - 在 View 布局中嵌入 Compose：`ComposeView`（XML 或代码添加），调用 `setContent { }`。
  - 在 Compose 中嵌入传统 View：`AndroidView`（工厂创建 + `update` 更新），适合 MapView、WebView、广告 SDK。
  - 注意：主题桥接（`AndroidViewBinding`）、`CompositionLocal` 传值、滚动与焦点/IME 的兼容。

## 十、开放与场景题

### 10.1 设计一个图片加载框架要考虑什么？

- **要点**：
  - 三级缓存：**内存 LruCache（活跃）→ 磁盘 DiskLruCache → 网络**；内存缓存按 `maxSize = 进程可用内存 / 8` 估算。
  - 采样与压缩：按控件尺寸 `inSampleSize`、`RGB_565`、WebP；长图区域解码（如 SubsamplingScaleImageView）。
  - 生命周期感知：随 Fragment/Activity 销毁取消请求与释放资源（Glide/Coil 已内建）。
  - 并发：请求去重（相同 URL + 尺寸只加载一次）、线程池（网络池 + 解码池分离）、队列优先级（可见优先）。
  - 其他：占位图/错误图/淡入动画、磁盘缓存 key（URL + 变换参数）、监控（加载耗时、命中率、OOM 率）。

### 10.2 如何实现一个线程安全的单例？

- **要点**：
  - 饿汉（类加载即创建，无锁但可能浪费）、懒汉（synchronized，性能差）、**双重检查锁定 DCL**（`volatile` 防指令重排）、静态内部类（推荐，类加载机制保证线程安全且懒加载）、**枚举单例**（防反射与序列化破坏）。
  - Kotlin：**`object` 声明即为线程安全的懒加载单例**；带参单例用 `companion object` + `@Volatile` + 双重检查。
  - 反问点：Android 上单例若持有 Context，必须传 `applicationContext` 防泄漏；多进程下单例不共享。

### 10.3 冷启动白屏/黑屏怎么解决？

- **要点**：
  - 原因：进程创建 + Application 初始化期间，系统按主题先画一个背景窗口（StartingWindow）。
  - 方案：给启动 Activity 设置带 `windowBackground`（logo/品牌图）的主题，或 `android:windowDisablePreview=true`（不推荐，会变成"点击无响应感"）；Android 12+ 用官方 `SplashScreen` API。
  - 同时做真正的启动加速（见 7.3），否则只是遮掩。

### 10.4 如何保证后台任务在进程被杀后仍能执行？

- **要点**：
  - `WorkManager`：带约束（网络/充电/延迟）的**可靠**异步任务，信息持久化到数据库，进程被杀或重启后仍会调度；底层按版本选择 JobScheduler / Firebase JobDispatcher / AlarmManager + Broadcast。
  - 需要精确时间点：`AlarmManager`（`setExactAndAllowWhileIdle`，12+ 需 `SCHEDULE_EXACT_ALARM` 权限）。
  - 需要立即前台执行：前台服务 + 通知（8.0+）。
  - 不要依赖 `Service` 常驻（会被杀）；也不要用空进程保活（违反规范，且现代系统无效）。

### 10.5 说一个你做过的性能优化案例（答题框架）

- **要点**：用"**指标 → 定位 → 方案 → 结果 → 防劣化**"五段式回答：
  1. 指标：明确量化目标（启动耗时 P95 从 1800ms 降到 900ms；帧率从 48 提升到 58）。
  2. 定位：工具（Perfetto/Systrace/Profiler/Matrix）与证据（火焰图、traces、掉帧分布）。
  3. 方案：具体改动（任务分级、懒加载、布局扁平化、缓存复用、异步化）。
  4. 结果：前后对比数据 + 稳定性/崩溃率无劣化 + 灰度验证。
  5. 防劣化：CI 卡口、监控告警、代码规范。

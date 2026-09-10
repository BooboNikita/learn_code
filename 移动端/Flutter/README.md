# Flutter 渲染树与渲染流程

> Flutter 是 Google 推出的跨端 UI 框架，自绘引擎（Skia/Impeller）直接接管渲染，不依赖平台原生控件。理解"三棵树"与一帧的渲染管线，是从会用 Flutter 到做好性能优化的分水岭。

## 一、渲染体系全景

### 1.1 与原生渲染的本质区别

- **原生（Android/iOS）**：控件 → 平台原生 View（系统帮你画）。
- **Flutter**：控件 → 自建渲染管线（自己画），每个像素都由 Flutter 引擎输出。
- 因此 Flutter 界面在 Android 和 iOS 上高度一致，代价是要自己管理好渲染性能。

### 1.2 一帧的完整链路

```
Vsync 信号
   │
   ▼
┌─────────────── UI 线程（Dart）───────────────┐
│  Build（重建 Widget） → Layout（布局）         │
│  → Paint（绘制） → Compositing（合成 Layer 树）│
└──────────────────┬───────────────────────────┘
                   │ Layer Tree（提交给引擎）
                   ▼
┌────────── Raster 线程（Skia / Impeller）──────┐
│  Rasterize 光栅化 → GPU 合成 → 上屏            │
└──────────────────────────────────────────────┘
```

- **UI 线程**：跑 Dart 代码，负责 Widget/Element/RenderObject 三棵树。
- **Raster 线程**：执行光栅化与 GPU 合成，两线程**并行流水线**。
- DevTools 中 `UI` / `Raster` 两条时间轴卡顿，对应的就是这两个线程。

## 二、三棵树：Widget / Element / RenderObject

### 2.1 各自的职责

| 维度 | Widget 树 | Element 树 | RenderObject 树 |
| ---- | --------- | ---------- | --------------- |
| 定位 | **配置描述**（长什么样） | **桥梁/上下文**（挂载实例） | **真正干活**（布局+绘制） |
| 可变性 | **不可变**（immutable） | 可变（生命周期内复用） | 可变（内部持有状态） |
| 数量 | 最多（可随意重建） | 中等 | 最少（非叶子节点才合并） |
| 典型类 | `Container`、`Text` | `StatefulElement`、`RenderObjectElement` | `RenderBox`、`RenderParagraph` |
| 生命周期 | 一帧一换 | 跨帧长期存在 | 跨帧长期存在 |

```
Widget 树（配置）          Element 树（实例/桥）        RenderObject 树（渲染）
    Container        ──▶      ContainerElement      ──▶      ┌─────────────┐
      ├─ Row         ──▶        ├─ RowElement     ──▶   RenderFlex(Row)   │
      │   ├─ Text    ──▶        │   ├─ TextElement──▶  RenderParagraph    │
      │   └─ Icon    ──▶        │   └─ IconElement──▶   RenderBox(icon)   │
      └─ Padding     ──▶        └─ PaddingElement ──▶  RenderPadding      │
                                                                         │
                                              Container 无 RenderObject ──┘
                                              （组合类 Widget，只是配置壳）
```

> 要点：**不是每个 Widget 都有 RenderObject**。`Container`、`Padding` 这类"组合 Widget"只是把配置层层传递，最终叶子上的 `RichText`/`ConstrainedBox` 才创建 RenderObject。Element 树的深度 = Widget 树的深度，RenderObject 树更"瘦"。

### 2.2 三者关系细节

- **Widget → Element**：`Widget.createElement()` 挂载时创建，Element 持有对 Widget 的引用。
- **Element → RenderObject**：`RenderObjectWidget.createRenderObject()` 创建，Element 负责更新（`updateRenderObject`）与卸载。
- **一个 Widget 可对应多个 Element**：同一个 Widget 实例被挂到树上多个位置（虽然不推荐），会产生多个 Element。
- **BuildContext 就是 Element**：`context` 实际是 `Element` 的接口，所以能用它 `findRenderObject()`、向上查祖先。

### 2.3 为什么 Widget 设计成不可变？

- UI 是**声明式**的：`UI = f(state)`，状态变化时不用手动改界面，直接用新配置**重新描述**即可。
- 不可变对象轻量、线程安全、可自由复用与重建，让 diff 简单：**比较引用即可**（`identical`）。
- 真正昂贵的东西（Element、RenderObject、State）被 Element 树保护起来，不随 Widget 重建而丢失。

### 2.4 setState 之后发生了什么（diff 核心）

```dart
// Element 更新子节点时的复用判断（简化）
static bool canUpdate(Widget old, Widget new) {
  return old.runtimeType == new.runtimeType
      && old.key == new.key;
}
```

`setState()` → `markNeedsBuild()`（标记该 Element 为 dirty）→ 下一帧 `build()` 返回**新 Widget** → 父 Element 逐个 diff 新旧子 Widget：

| 情况 | 结果 |
| ---- | ---- |
| `canUpdate` 为 true（类型+key 相同） | **复用** Element 与 RenderObject，只更新引用/属性 |
| 类型或 key 变了 | 废弃旧 Element，创建新 Element（子树重建） |
| 新 Widget 为 null | 卸载对应 Element 与 RenderObject |

- **StatefulWidget 的 State 挂在 Element 上**，所以 Widget 每帧重建，`State`（含控制器、动画）不会丢——这就是"Widget 频繁重建但状态不丢"的原理。
- **Key 的作用**：列表中间插入/删除、交换顺序时，让 diff 按 key 而非位置匹配，避免状态错位。
  - `LocalKey`（`ValueKey`/`ObjectKey`）：同级比较。
  - `GlobalKey`：跨树复用 Element 与 State（也可拿 context/state），代价大，慎用。

## 三、渲染流程四阶段

> UI 线程每帧依次执行：**Build → Layout → Paint → Compositing**，任一阶段超时（>16.6ms）就掉帧。

### 3.1 Build 阶段

- 从根 `RenderObjectToWidgetElement` 开始，把所有 **dirty 的 Element** 重新执行 `build()`，生成新 Widget 子树并 diff（见 2.4）。
- 只有被 `markNeedsBuild` 标记的子树会重建，未变化的子树整棵跳过。
- 耗时大头：`build()` 里构造了大量临时 Widget、调用了昂贵函数（如排序、正则）。

### 3.2 Layout 阶段（约束向下，尺寸向上）

Flutter 布局是**单遍递归**（contrast 传统 CSS 的多次回流）：

```
父节点：传约束 Constraints（BoxConstraints：min/max 宽高）
   │
   ▼
子节点 performLayout()：在约束内计算自身 size
   │
   ▼
父节点：拿子 size + 自身逻辑决定位置（parentData.offset）
```

- **约束由上往下，尺寸由下往上**，O(n) 一次完成，这是 Flutter 布局快的原因。
- 常见约束：`BoxConstraints`（盒模型）、`SliverConstraints`（列表懒加载）。
- 典型现象：`Container` 宽高不生效、`Expanded` 必须在 `Flex` 里、`UnconstrainedBox` 报错"size overflow"——都是约束传递规则在起作用。

### 3.3 Paint 阶段（生成 Layer 树）

- `paint()` 把绘制指令写入 **Layer**，形成 Layer 树（与 RenderObject 树解耦的"绘制结果"结构）。
- 只有被 `markNeedsPaint` 标记的子树才重绘，其祖先只要在同一个 Layer 内就**不受影响**。
- 常见 Layer：`PictureLayer`（普通绘制指令）、`TextureLayer`（视频/相机纹理）、`PlatformViewLayer`（原生视图嵌入）。

### 3.4 Compositing & Rasterize 阶段

- Layer 树被合成为最终场景（`Scene`），提交给引擎。
- Raster 线程用 Skia/Impeller 把绘制指令**光栅化**成 GPU 操作，合成上屏。
- Impeller 的出现：Skia 的着色器 JIT 编译可能导致首次卡顿，Impeller **预编译着色器**，从根本上解决 jank。

## 四、两大性能边界：RelayoutBoundary 与 RepaintBoundary

### 4.1 RelayoutBoundary（布局边界）

当某个 RenderObject 满足以下**任一**条件时，自身成为布局边界：

- 约束是 **tight**（父约束唯一确定尺寸，如固定宽高、`SizedBox` 包裹）。
- `parentUsesSize == false`（父不关心它的尺寸）。
- 自身 `sizedByParent`（尺寸与子树无关）。

作用：子树内部 `markNeedsLayout` 时，**只从边界处重排**，不传导到根，避免整棵树重新布局。

### 4.2 RepaintBoundary（重绘边界）

- 强制该 RenderObject **独占一个 Layer**，其子树重绘被隔离在该 Layer 内。
- 典型应用：
  - 列表滚动容器（`ListView` 等自动给 item 加边界）。
  - 频繁局部刷新区域（时钟、动画、图表）外包 `RepaintBoundary`。
- `Container` 会自动在子树外层加 `RepaintBoundary`（有 child 时）。

### 4.3 优化要点速查

| 优化手段 | 作用 |
| -------- | ---- |
| `const` 构造 Widget | 复用同一实例，diff 时 `identical` 直接跳过整棵子树 build |
| 拆小 `StatefulWidget` | 缩小 setState 的 dirty 范围 |
| `RepaintBoundary` 包裹 | 隔离重绘，避免大范围 paint |
| 缓存/拆分 build 中的重计算 | 减少单帧 build 耗时 |
| `ListView.builder` 懒加载 | 只 build 可见 + 缓存区 |
| DevTools Timeline / Performance | 定位 UI 线程卡（build/layout/paint）还是 Raster 卡（shader、复杂裁剪） |

## 五、高频面试问答

**Q1：Widget、Element、RenderObject 分别是什么关系？**
Widget 是不可变配置；Element 是 Widget 挂载后的实例、diff 与复用的桥梁，也是 BuildContext；RenderObject 负责真正的布局与绘制。Widget 每 frame 可重建，Element/RenderObject 尽量复用。

**Q2：setState 后一定会重新 build 整棵树吗？**
不会。只有该 Element 到根之间标记 dirty 的路径会重建，`canUpdate` 匹配（类型+key 相同）则复用底层 Element/RenderObject，`const` 构造的子树直接整棵跳过。

**Q3：为什么 StatefulWidget 重建 State 不丢？**
State 保存在 StatefulElement 上，Element 在 diff 中被复用时，State 一并保留，只在 Element 真正被卸载或 key 变化时销毁。

**Q4：Flutter 为什么流畅（对比 WebView/H5）？**
自绘引擎直出像素，无 JS Bridge、无 DOM 多层抽象；单遍布局；dirty 机制局部刷新；UI 与 Raster 双线程流水线 + 独占 GPU 合成。

**Q5：界面卡顿怎么排查？**
DevTools Performance 看帧耗时：**UI 线程**长 → build/layout/paint 过重（用 `const`、拆组件、RepaintBoundary）；**Raster 线程**长 → 着色器、saveLayer、复杂裁剪/阴影（换 Impeller、简化视觉效果）。

## 小结

- 一帧 = **Vsync 驱动**：Build → Layout → Paint → Compositing（UI 线程）+ Rasterize（Raster 线程）双线并行。
- 三棵树 = **配置（Widget）/ 桥梁（Element）/ 渲染（RenderObject）**，不可变 Widget + 可复用 Element 是声明式 UI 与高性能兼得的关键。
- diff 规则 = **runtimeType + key** 决定 Element 复用与否。
- 布局 = **约束向下、尺寸向上**，单遍完成。
- 性能 = 缩小 dirty 范围 + 利用 Relayout/Repaint 边界 + `const` 构造，用 DevTools 定位是 UI 还是 Raster 线程的锅。

---
name: "web-article-to-docsify"
description: "抓取任意技术文章 URL，归纳整理为体系化的 docsify 复习笔记，自动落库到仓库对应分类下，支持下载原文图片。Invoke when user provides a web article URL (xiaolincoding.com, juejin.cn, segmentfault.com, cnblogs.com, zhihu.com 等) and asks to summarize / 归纳 / 总结 / 精简 / 复习."
---

# Web 技术文章 → Docsify 复习笔记

把任意技术长文（小林 coding、掘金、博客园、知乎等）**精简成可背诵的体系化复习笔记**，自动落到本仓库的 docsify 文档站里，支持下载原文关键图片，更新侧边栏与分类索引。

---

## 1. 适用场景

用户消息里出现以下任一信号即可触发：

- 出现技术文章 URL（`xiaolincoding.com`、`juejin.cn`、`segmentfault.com`、`cnblogs.com`、`zhihu.com`、`csdn.net`、`infoq.cn`、`medium.com` 等）
- 关键词：`归纳`、`总结`、`精简`、`简化`、`复习`、`笔记`、`保存图片` 等
- 用户说"把当前链接内容归纳到对应文件夹"

**不适用**：

- 微信公众号文章 → 用 `wechat-to-docsify` skill
- 视频/课程链接

---

## 2. 仓库现状（执行前必读）

工作目录 `/Users/huyabaodi/Documents/codenew/learn_code` 已搭建 docsify 文档站，分三大类：

- **前端**：`前端/<子分类>/README.md`
- **后端**：`后端/<子分类>/README.md`
- **人工智能**：`人工智能/<子分类>/README.md`

关键入口文件：

- [\_sidebar.md](file:///Users/huyabaodi/Documents/codenew/learn_code/_sidebar.md) — 侧边栏导航
- [README.md](file:///Users/huyabaodi/Documents/codenew/learn_code/README.md) — 站点首页
- 各分类下 `<分类>/<子分类>/README.md` — 该分类索引页

---

## 3. 执行流程（5 步必须走完）

### Step 1：抓取原文

```bash
WebFetch(url=<文章 URL>)
```

- 输出可能超过 20KB，**会落到 `/tmp/trae/toolcall-output/xxx.txt`**。
- 用 `Read` 完整读完，不要只读 preview。
- 抓取时同时记录原文中所有 `<img src=...>` 的 URL（特别是图床如 `aka.doubaocdn.com`），后续 Step 4 会用到。

### Step 2：确定落点

根据文章主题映射到具体路径：

| 文章主题                         | 落点路径                      | 命名建议            |
| -------------------------------- | ----------------------------- | ------------------- |
| 分库分表 / Sharding / 数据迁移   | `后端/分布式/`                | `<主题>.md`         |
| 分布式事务 / TCC / SAGA / 2PC    | `后端/分布式/`                | `分布式事务.md`     |
| 高可用 / 集群 / 主从 / 选举      | `后端/分布式/`                | `系统高可用设计.md` |
| 消息队列 / Kafka / RocketMQ      | `后端/分布式/`                | `消息队列<主题>.md` |
| 缓存 / Redis / 多级缓存          | `后端/Redis/`                 | `<主题>.md`         |
| MySQL 索引 / 锁 / 性能           | `后端/MySQL/`                 | `<主题>.md`         |
| Java 基础 / JVM / 并发           | `后端/Java/`                  | `<主题>.md`         |
| 操作系统 / 计算机网络 / 数据结构 | `后端/计算机基础/`            | `<主题>.md`         |
| 微服务 / Spring Cloud / Dubbo    | `后端/微服务/`                | `<主题>.md`         |
| 高并发 / 性能优化 / 限流 / 熔断  | `后端/高并发/`                | `<主题>.md`         |
| 系统设计 / 秒杀 / 短链 / 排行榜  | `后端/系统设计/`              | `<主题>.md`         |
| 设计模式                         | `后端/设计模式/`              | `<主题>.md`         |
| Vue / React / 前端工程化         | `前端/<子分类>/`              | `<主题>.md`         |
| AI / LLM / Agent / ML            | `人工智能/<子分类>/`          | `<主题>.md`         |
| 其他                             | **用 AskUserQuestion 问用户** | —                   |

### Step 3：精简归纳（体系化优先）

**目标**：原文通常 20~30KB，**压到 5~10KB**，**按知识体系分块**而非按原文顺序。

**归纳原则（区别于纯简化）**：

- **按知识体系重组**：原文按问答或章节组织，笔记应按"概念 → 机制 → 对比 → 应用"的体系重组
- **合并同类项**：如多个小标题都在讲"堆分代"，合并为一张表
- **抽取对比维度**：凡涉及"X vs Y"，必须出对比表
- **保留关键数字与阈值**：如 `8:1:1`、`-Xss`、`15 次晋升` 等

**保留**：

- 核心概念定义（一句话说清）
- 关键机制 / 协议 / 算法的对比
- 必要的小段代码示例（≤ 20 行）
- 关键数字、参数、阈值
- 关键图片（结构图、流程图、对比图）

**丢弃**：

- 客套话、自我宣传（"今天我们来聊聊"）
- 重复论述
- 文末推荐阅读链接、广告

**结构模板**：

```markdown
# <主题>

> 来源：<原文链接>
> 一句话总结：<核心结论>

（可选：封面图）

## 一、<知识模块 1>（如"内存模型"）

### 1.1 <子主题>

> 关键概念一句话定义

| 维度 | A   | B   |
| ---- | --- | --- |
| ...  | ... | ... |

### 1.2 <子主题>

...

## 二、<知识模块 2>（如"垃圾回收"）

| 对比项 | 方式 1 | 方式 2 | 方式 3 |
| ------ | ------ | ------ | ------ |
| ...    | ...    | ...    | ...    |

## N. 复习清单（必带）

1. **问题 1？** 简短答案。
2. **问题 2？** 简短答案。
   ...（8~12 个）
```

**风格要求**：

- 全文中文
- 关键对比必须用 Markdown 表格（**至少 3 个表格**）
- 关键代码用代码块 + 语言标签
- 每个一级模块下要有 ≥ 2 个子主题
- 复习清单放在文末（**8~12 题**）

### Step 4：下载图片 + 写入笔记

#### 4.1 下载关键图片

原文中的关键图片（JVM 结构图、堆分代图、流程图等）应下载到本地，避免外链失效：

```bash
python3 -c "
import urllib.request, os
base = '<落点路径>/images'
os.makedirs(base, exist_ok=True)
imgs = {
    '<语义化文件名>.png': '<原图 URL>',
    ...
}
for name, url in imgs.items():
    req = urllib.request.Request(url, headers={'User-Agent': 'Mozilla/5.0'})
    with urllib.request.urlopen(req, timeout=30) as r, open(os.path.join(base, name), 'wb') as f:
        f.write(r.read())
"
```

**命名规范**：`<主题英文>-<说明>.png`，如：

- `jvm-memory-structure.png`
- `heap-generation.png`
- `gc-algorithm-compare.png`

**选用原则**（避免全下，只下关键的）：

- 体系结构图 / 内存布局图
- 流程图 / 时序图
- 多对象对比图
- **不下**：表情包、广告图、封面装饰图

#### 4.2 笔记中用相对路径引用

```markdown
![JVM 内存结构](images/jvm-memory-structure.png)
```

#### 4.3 写入主文章

`<落点路径>/<文件名>.md`

#### 4.4 更新侧边栏

在 [\_sidebar.md](file:///Users/huyabaodi/Documents/codenew/learn_code/_sidebar.md) 对应分类下加 2 空格缩进的子项：

```markdown
- [Java 基础](后端/Java/README.md)
  - [JVM 面试题](后端/Java/JVM面试题.md) # ← 新增
```

**注意**：每次 Edit 前必须 `Read` 对应文件（工具限制）。

### Step 5：Git 提交

所有文件写入完成后，**必须自动执行 git commit**，将本次新增/修改的文件提交到版本库：

```bash
# 1. 先查看状态，确认变更文件
git status
git diff

# 2. 添加本次涉及的文件（按文件名逐个添加，不要 git add .）
git add <落点路径>/<文件名>.md <落点路径>/images/ _sidebar.md

# 3. 提交，commit message 格式：docs: 归纳<文章主题>
git commit -m "docs: 归纳<文章主题>"
```

**commit message 规范**：

- 格式：`docs: 归纳<主题关键词>`
- 示例：`docs: 归纳Java并发编程`、`docs: 归纳JVM面试题`、`docs: 归纳Redis数据类型`

**注意**：

- 只 add 本次新增/修改的文件，**不要** `git add .` 或 `git add -A`
- 不需要 `git push`，除非用户明确要求
- 提交后用 `git status` 确认工作区干净

### Step 6：回报

完成后**给用户清晰回报**：

- 列出本次新增/修改的文件
- 给出 docsify 站点可访问路径
- 提示硬刷新（Ctrl+Shift+R）
- 确认 git commit 已完成

---

## 4. 验证清单（提交前自检）

- [ ] 文章已用 `Read` 完整读完（不是只看 preview）
- [ ] 精简后体积 5~10KB
- [ ] **按知识体系分块**（不是按原文章节）
- [ ] 至少有 **3 个对比表**
- [ ] 关键图片已下载到 `images/` 并用相对路径引用
- [ ] 文末有 **8~12 题复习清单**
- [ ] 侧边栏已更新（\_sidebar.md）
- [ ] 文件名用中文且简洁
- [ ] 已执行 git commit（只 add 本次变更文件）

---

## 5. 异常处理

| 异常                           | 处理                                                      |
| ------------------------------ | --------------------------------------------------------- |
| `WebFetch` 输出被截断          | 用 `Read` 读 `/tmp/trae/toolcall-output/xxx.txt` 完整文件 |
| 文章主题无法归类               | **必须**用 `AskUserQuestion` 问用户放哪                   |
| `_sidebar.md` 已被其他进程修改 | `Read` 后再 `Edit`，不要盲改                              |
| 图片下载失败（防盗链）         | 跳过该图，保留外链或删掉引用                              |
| 用户没给 URL                   | **必须**用 `AskUserQuestion` 确认文章来源                 |
| 用户明确说"不要图片"           | 跳过 Step 4.1，仅写笔记                                   |

---

## 6. 核心原则

- **少做事**：用户没要求的功能（封面、动画、SEO）一律不主动加
- **可复习**：每篇结尾必须有复习清单，对比表、代码块缺一不可
- **体系化**：按知识体系重组，而非按原文顺序搬运
- **零构建**：docsify 是纯静态站，所有 .md 文件直接 commit 即生效
- **中文优先**：所有内容用中文（除非用户明确要求英文）
- **图片可控**：默认下载关键图，但用户可主动要求"只下文字"或"全下"

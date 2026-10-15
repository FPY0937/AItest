# 格式示例：一份 README 怎么改造成 glue 目录

**这不是已登记资产。** 下面的 `id`、正文、标签只用来说明「现在只有一份 README」时，提供方要落哪些文件。不得抄进真实目录，不得当成可调用地址。

假设提供方现在是这样发组件的：

```text
某组件工程/
  README.md          ← 文档全在这里
  examples/
    App.java         ← 可运行样例（可选）
  （程序包按现有方式发布，本示例不写坐标）
```

`glue` 不托管程序包本身。要改的是：**把 README 变成可按章拉取的记录，把一句话能力写进身份，发布时生成检索索引。**

---

## 改造前：README.md

```markdown
# 写操作幂等

为写操作提供按业务键去重，避免同一请求被处理两次。

## 引入

在本服务依赖中声明该组件后，即可使用幂等过滤器。

## 使用

调用方为每次写操作提供业务键。相同键在窗口内只处理一次。

## 错误

键为空时拒绝。窗口外的重复请求按新请求处理。
```

今天的问题：代理若要复用，只能整份翻 README。glue 规定一次只拉一章、检索不扫正文，所以必须拆。

---

## 改造分两步，选一个就够开工

### 最小改法：整份 README 当作一章

README 小于 256 KiB 时，可以先不按标题拆。

```text
catalog/source/write-idempotent/
  identity.yaml
  versions/
    1.0.0.yaml
    1.0.0.chapters.yaml
    1.0.0.samples.yaml          ← 没有通过测试的样例就不要这个文件
objects/
  write-idempotent/1.0.0/chapters/readme
```

`1.0.0.chapters.yaml` 只有一行，`chapter` 为 `readme`，正文就是原来那份 README。

详细设计拉证据时：`get_doc_section(id=write-idempotent, version=1.0.0, chapter=readme)`。一次拿到全文。以后再按标题拆也不改 `id`。

### 推荐改法：按二级标题拆章（下面整篇按这个写）

每个 `##` 一章。详细设计只拉「使用」或「错误」那一章当完成方式证据。

| README 里的标题 | 章键 `chapter` | 对象路径（相对 `{root}/objects/`） |
|---|---|---|
| （文首一段话） | `overview` | `write-idempotent/1.0.0/chapters/overview` |
| 引入 | `import` | `write-idempotent/1.0.0/chapters/import` |
| 使用 | `usage` | `write-idempotent/1.0.0/chapters/usage` |
| 错误 | `errors` | `write-idempotent/1.0.0/chapters/errors` |

`id` 用路径安全的稳定名，**不要**用中文当目录名。中文放在 `summary`、`title`、`tags` 里。

---

## 改造后：人维护的源文件

### `catalog/source/write-idempotent/identity.yaml`

`summary` 从 README 第一段抽出，**检索只切这一句**，不是整份 README。

```yaml
schemaVersion: "1"
id: write-idempotent
summary: 为写操作提供按业务键去重
tags:
  - 幂等
  - 写操作
maintainer: ""
status: active
```

### `catalog/source/write-idempotent/versions/1.0.0.yaml`

`gav` 填该版本真实发布标识。没有事实就留空，不要编。

```yaml
schemaVersion: "1"
id: write-idempotent
version: "1.0.0"
gav: ""
sampleTag: "1.0.0"
sampleTestsPassed: true
docKeyTemplate: "{root}/{id}/{version}/chapters/{chapter}"
```

本示例假定 `examples/App.java` 自身测试已通过，所以 `sampleTestsPassed` 为真。若样例没测过：`sampleTestsPassed: false`，`sampleTag` 为空，编码阶段不能 `get_sample`。

### `catalog/source/write-idempotent/versions/1.0.0.chapters.yaml`

```yaml
schemaVersion: "1"
id: write-idempotent
version: "1.0.0"
chapters:
  - chapter: overview
    title: 概述
    ordinal: 0
    byteSize: 63
    contentHash: ""
    objectKey: write-idempotent/1.0.0/chapters/overview
  - chapter: import
    title: 引入
    ordinal: 1
    byteSize: 57
    contentHash: ""
    objectKey: write-idempotent/1.0.0/chapters/import
  - chapter: usage
    title: 使用
    ordinal: 2
    byteSize: 60
    contentHash: ""
    objectKey: write-idempotent/1.0.0/chapters/usage
  - chapter: errors
    title: 错误
    ordinal: 3
    byteSize: 57
    contentHash: ""
    objectKey: write-idempotent/1.0.0/chapters/errors
```

`byteSize` / `contentHash` 以实际文件为准。这里的数字只配合下面几段短文，发布时按真实字节重算。`contentHash` 有文件后再填 SHA-256。

### 四章正文（原来 README 拆开）

`objects/write-idempotent/1.0.0/chapters/overview`

```text
为写操作提供按业务键去重，避免同一请求被处理两次。
```

`objects/write-idempotent/1.0.0/chapters/import`

```text
在本服务依赖中声明该组件后，即可使用幂等过滤器。
```

`objects/write-idempotent/1.0.0/chapters/usage`

```text
调用方为每次写操作提供业务键。相同键在窗口内只处理一次。
```

`objects/write-idempotent/1.0.0/chapters/errors`

```text
键为空时拒绝。窗口外的重复请求按新请求处理。
```

### `catalog/source/write-idempotent/versions/1.0.0.samples.yaml`

没有样例就删掉这个文件，并把版本记录里的 `sampleTestsPassed` 设为假。

```yaml
schemaVersion: "1"
id: write-idempotent
version: "1.0.0"
sampleTag: "1.0.0"
files:
  - path: App.java
    byteSize: 0
    contentHash: ""
    objectKey: references/write-idempotent-ref/1.0.0/App.java
```

`objects/references/write-idempotent-ref/1.0.0/App.java` 的内容就是原来的 `examples/App.java`。`byteSize` 按实际文件填。

---

## 发布时生成的索引（人不用手写倒排）

发布脚本读全部 `identity.yaml` 和版本记录，写出一份 `catalog/built/search-index.yaml`。针对这一条，里面会有：

```yaml
schemaVersion: "1"
builtFrom: ""
components:
  - id: write-idempotent
    summary: 为写操作提供按业务键去重
    tags:
      - 幂等
      - 写操作
    maintainer: ""
    status: active
    versions:
      - version: "1.0.0"
        gav: ""
        sampleTag: "1.0.0"
        sampleTestsPassed: true
postings:
  tag:
    - tag: 幂等
      ids: [write-idempotent]
    - tag: 写操作
      ids: [write-idempotent]
  token:
    - token: 为写
      ids: [write-idempotent]
    - token: 写操
      ids: [write-idempotent]
    - token: 操作
      ids: [write-idempotent]
    - token: 作提
      ids: [write-idempotent]
    - token: 提供
      ids: [write-idempotent]
    - token: 供按
      ids: [write-idempotent]
    - token: 按业
      ids: [write-idempotent]
    - token: 业务
      ids: [write-idempotent]
    - token: 务键
      ids: [write-idempotent]
    - token: 键去
      ids: [write-idempotent]
    - token: 去重
      ids: [write-idempotent]
  deprecatedTag: []
```

注意：**「引入」「使用」「错误」这些章节正文没有进倒排。** 检索命中靠 `summary` 和 `tags`。

服务启动后内存是：

| 表 | 这一例里的内容 |
|---|---|
| `byId["write-idempotent"]` | 上面那条 component |
| `byVersion["write-idempotent"+"1.0.0"]` | gav / sampleTag / sampleTestsPassed |
| `byTag["幂等"]` | `[write-idempotent]` |
| `byToken["去重"]` 等 | `[write-idempotent]` |

---

## 开发组实际会调到什么

某需求要「写操作按业务键去重」。概要设计只调检索：

**入参**

```yaml
callerStage: 概要设计
summary: 写操作按业务键去重
tags:
  - 幂等
id: ""
includeDeprecated: false
```

查询句切出「写操」「操作」等，和 `byToken` 相交；标签「幂等」再过滤。本例恰好一条。

**出参（只有四个字段，没有 README）**

```yaml
ok: true
matchCount: 1
matches:
  - id: write-idempotent
    summary: 为写操作提供按业务键去重
    tags:
      - 幂等
      - 写操作
    status: active
```

概要只记下 `write-idempotent`，标记「待详细设计绑定」。**此时还没有读任何一章。**

详细设计点查并看目录：

**入参** `get_component`

```yaml
callerStage: 详细设计
id: write-idempotent
version: "1.0.0"
```

**出参（有标题，无正文）**

```yaml
ok: true
id: write-idempotent
summary: 为写操作提供按业务键去重
tags:
  - 幂等
  - 写操作
maintainer: ""
status: active
version: "1.0.0"
gav: ""
docKeyTemplate: "{root}/{id}/{version}/chapters/{chapter}"
sampleTag: "1.0.0"
sampleTestsPassed: true
chapters:
  - chapter: overview
    title: 概述
    ordinal: 0
  - chapter: import
    title: 引入
    ordinal: 1
  - chapter: usage
    title: 使用
    ordinal: 2
  - chapter: errors
    title: 错误
    ordinal: 3
```

人指定用「使用」这一章做「既有资产」证据：

**入参** `get_doc_section`

```yaml
callerStage: 详细设计
id: write-idempotent
version: "1.0.0"
chapter: usage
```

**出参（只有这一章）**

```yaml
ok: true
id: write-idempotent
version: "1.0.0"
chapter: usage
title: 使用
byteSize: 60
contentHash: ""
content: 调用方为每次写操作提供业务键。相同键在窗口内只处理一次。
```

完成方式写成：既有资产，证据为文档名「使用」、章节 `usage`。不能一次把四章都拉下来挑选。

编码阶段若要样例，先 `get_sample` 不带 `path` 拿到 `App.java` 清单，再带 `path: App.java` 拉文件。程序包仍按该版本的 `gav` 在现有仓库取，不经过 glue。

---

## 对照：README 里的东西分别去哪了

| 原来 | 改造后 | 谁在什么时候看见 |
|---|---|---|
| 标题下的一句话能力 | `identity.yaml` 的 `summary` | 概要设计检索结果 |
| 人工分类 | `identity.yaml` 的 `tags` | 同上 |
| 整份 README | 拆成 `objects/.../chapters/*` | 详细设计 / 编码，一次一章 |
| `##` 标题 | `chapters.yaml` 的 `title` + `chapter` | `get_component` 的目录 |
| `examples/` 且测试通过 | `samples.yaml` + `objects/references/...` | 仅编码 `get_sample` |
| 程序包坐标 | `{version}.yaml` 的 `gav` | 详细设计指定版本之后 |
| README 全文检索 | **不做** | 正文不进倒排 |

提供方每发一个新版本：新增 `versions/1.1.0.yaml` 和对应 chapters/samples，把该版 README（或该版文档）再拆一遍；`identity.yaml` 不动。旧版本文件留着，服务不会自动改成新版本。

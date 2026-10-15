# glue 查阅服务实现

MCP 只是调用协议。对提供方来说，`glue` 就是一个长期运行的进程，对外四个方法。换成 HTTP 时，四个方法变成四个 POST，**body 字段完全一样**。IDE 用 MCP 是为了让代理按工具名调用，不是另做一套业务。

服务地址、对象存储根路径、真实组件标识本仓库保持空白。下面的空字符串只表示字段形状。

一份现有 README 怎么拆成这些文件，见 [glue-example-readme.md](./glue-example-readme.md)。那篇是格式演示，不是已登记资产。进程里 HashMap 与文件读取的顺序见 [glue-runtime.md](./glue-runtime.md)。

## 1. 一次调用怎么走

```text
IDE 代理
  → MCP 会话（或 HTTP POST）带 GLUE_TOKEN
  → 门禁：令牌、callerStage
  → 对应方法
  → 查内存表或按 objectKey 读一个对象
  → 写一条审计
  → 返回 JSON
```

- 令牌从环境变量 `GLUE_TOKEN` 来，不进请求 body，不进仓库。
- `callerStage` 是这次调用发生的阶段名，必须由调用方传入。服务按第 4 节对照表拒绝不允许的阶段。
- 业务代码不得调用。阶段包里没有这个进程。

四个方法对应 HTTP 时的路径（路径本身不是已发布地址）：

| 方法 | 若用 HTTP |
|---|---|
| `search_components` | `POST /search_components` |
| `get_component` | `POST /get_component` |
| `get_doc_section` | `POST /get_doc_section` |
| `get_sample` | `POST /get_sample` |

公共入参（每个方法都有）：

| 字段 | 位置 | 类型 | 说明 |
|---|---|---|---|
| 令牌 | MCP 会话 / HTTP `Authorization` | 字符串 | 与 `GLUE_TOKEN` 比对 |
| `callerStage` | body | 字符串 | `概要设计` / `详细设计` / `程序设计` / `编码` 四值之一。其它阶段一律拒绝 |

公共拒绝（所有方法）：

```yaml
ok: false
code: ""
message: ""
```

| `code` | 何时 |
|---|---|
| `missing_token` | 没有令牌或令牌不对 |
| `forbidden_stage` | `callerStage` 不允许调用该方法 |
| `not_found` | 指定的 `id` / `version` / `chapter` / `path` 不存在 |
| `empty_query` | `search_components` 的 `id`、`summary`、`tags` 都空 |
| `missing_version` | 拉章或取样例时没有 `version` |
| `missing_chapter` | 拉章时没有 `chapter`，或要求整份文档 |
| `too_large` | 单章大于 262144 字节，或一次样例正文合计大于 1048576 字节 |
| `sample_not_passed` | `sampleTestsPassed` 为假仍要读样例正文 |
| `hash_mismatch` | 读到的对象与记录的 `contentHash` 对不上 |

成功时 `ok: true`，业务字段见各方法。下面各节的 YAML 都是成功时的 body（MCP 工具返回值、HTTP 响应体同一套）。

## 2. 数据存在哪里

全部在提供方。使用方工程和阶段包里都没有这些文件。

```text
{root}/                          # 服务配置。本仓库空白
  catalog/
    source/{id}/
      identity.yaml              # 身份，一个组件一条
      versions/
        {version}.yaml           # 该版本的发布信息
        {version}.chapters.yaml  # 该版本文档目录（无正文）
        {version}.samples.yaml   # 该版本样例目录（无文件内容）
    built/
      search-index.yaml          # 发布时由 source 生成，检索只读这一份
  objects/
    {id}/{version}/chapters/{chapter}              # 一章正文
    references/{id}-ref/{version}/{path}           # 一个样例文件
  audit/
    {yyyy}/{mm}/{dd}.jsonl       # 每次调用追加一行。不进 catalog，不进阶段包
```

`{root}`、对象存储是否另机、审计目录的真实路径，都由提供方服务配置决定，这里不填。

三类数据不要混：

| 放哪 | 放什么 | 谁读 |
|---|---|---|
| `catalog/source/` | 人维护的源记录 | 发布时生成索引；`get_component` 指定版本后读章节标题；拉章/取样例时用 `objectKey` |
| `catalog/built/search-index.yaml` | 生成物 | 进程启动载入内存。检索和点查身份只打内存，不扫 `source` |
| `objects/` | 正文和样例文件 | 只有 `get_doc_section`、`get_sample` 按键读一个对象 |
| `audit/` | 调用日志 | 只写，不参与检索 |

`search-index.yaml` 生成完成前继续用上一份，生成完一次换上。

## 3. 磁盘上的记录（数据结构）

### 3.1 身份 `catalog/source/{id}/identity.yaml`

主键：`id`（与目录名相同，发布后不改）。

```yaml
schemaVersion: "1"
id: ""
summary: ""
tags: []
maintainer: ""
status: active
```

| 字段 | 类型 | 约束 |
|---|---|---|
| `id` | 字符串 | 路径名：字母、数字、点、下划线、连字符 |
| `summary` | 字符串 | 一句话能力，1–80 字符。倒排只切这一句 |
| `tags` | 字符串列表 | 人填。每项 1–32 字符，不重复 |
| `maintainer` | 字符串 | 可空 |
| `status` | 枚举 | `active` / `experimental` / `deprecated` |

### 3.2 版本 `catalog/source/{id}/versions/{version}.yaml`

主键：`id` + `version`。同一 `id` 下多版本并存，互不覆盖。服务不按版本号大小选“最新”。

```yaml
schemaVersion: "1"
id: ""
version: ""
gav: ""
sampleTag: ""
sampleTestsPassed: false
docKeyTemplate: "{root}/{id}/{version}/chapters/{chapter}"
```

| 字段 | 类型 | 约束 |
|---|---|---|
| `gav` | 字符串 | 该版本发布标识。无事实则空 |
| `sampleTag` | 字符串 | `sampleTestsPassed` 为真时必须等于 `version`，否则空 |
| `sampleTestsPassed` | 布尔 | 为真才允许读样例正文 |
| `docKeyTemplate` | 字符串 | 固定上式。`{root}` 来自配置 |

### 3.3 章节目录 `.../{version}.chapters.yaml`

一行一章，**没有正文**。

```yaml
schemaVersion: "1"
id: ""
version: ""
chapters:
  - chapter: ""
    title: ""
    ordinal: 0
    byteSize: 0
    contentHash: ""
    objectKey: ""
```

| 字段 | 含义 |
|---|---|
| `chapter` | 章主键，本文件内不重复 |
| `title` | 给人看的标题，1–80 字符 |
| `ordinal` | 从 0 递增，决定标题顺序 |
| `byteSize` | 正文字节数。大于 262144 不得发布 |
| `contentHash` | 正文 SHA-256，十六进制小写。正文未提供则空 |
| `objectKey` | `{id}/{version}/chapters/{chapter}`，不含主机名 |

正文实际文件：`{root}/objects/{objectKey}`。

### 3.4 样例目录 `.../{version}.samples.yaml`

一行一个文件，**没有文件内容**。

```yaml
schemaVersion: "1"
id: ""
version: ""
sampleTag: ""
files:
  - path: ""
    byteSize: 0
    contentHash: ""
    objectKey: ""
```

| 字段 | 含义 |
|---|---|
| `path` | 样例内相对路径，不含 `..`，本文件内不重复 |
| `objectKey` | `references/{id}-ref/{version}/{path}` |

样例实际文件：`{root}/objects/{objectKey}`。

### 3.5 发布索引 `catalog/built/search-index.yaml`

由全部 `source` 生成。章节目录不复制进来。

```yaml
schemaVersion: "1"
builtFrom: ""
components:
  - id: ""
    summary: ""
    tags: []
    maintainer: ""
    status: active
    versions:
      - version: ""
        gav: ""
        sampleTag: ""
        sampleTestsPassed: false
postings:
  tag:
    - tag: ""
      ids: []
  token:
    - token: ""
      ids: []
  deprecatedTag:
    - tag: ""
      ids: []
```

| 倒排 | 收录谁 |
|---|---|
| `postings.tag` | `status` 为 `active` 或 `experimental` 的标签 → `id` 列表 |
| `postings.token` | 对 `summary` 切词后的词 → `id` 列表 |
| `postings.deprecatedTag` | `status` 为 `deprecated` 的标签 → `id` 列表 |

`ids` 已按字典序排序、不重复。

切词（建索引和查询同一套，不切正文）：

1. 去首尾空白。
2. 连续英文字母和数字成一个词，字母转小写；长度小于 2 丢掉。
3. 连续汉字每相邻两字一个词。
4. 标点、空白只用来断开。

### 3.6 审计一行

```yaml
time: ""
identity: ""
action: ""
id: ""
version: ""
allowed: false
reason: ""
```

| 字段 | 含义 |
|---|---|
| `time` | 调用时间 |
| `identity` | 谁发起（令牌对应的调用方身份） |
| `action` | 四个方法名之一 |
| `id` | 涉及的资产；分类检索未命中可空 |
| `version` | 涉及的版本；分类检索为空 |
| `allowed` | 这次是否接受 |
| `reason` | 拒绝时填 `code` |

## 4. 内存里的表（进程启动时装）

只从 `search-index.yaml` 建成。之后检索不再读磁盘。

| 表 | 键 | 值 |
|---|---|---|
| `byId` | `id` | `components` 里那一条（含 `versions` 摘要） |
| `byVersion` | `id` + `version` | 该版本的 `gav`、`sampleTag`、`sampleTestsPassed` |
| `byTag` | `tag` | `postings.tag` 的 `ids` |
| `byToken` | 词 | `postings.token` 的 `ids` |
| `byDeprecatedTag` | `tag` | `postings.deprecatedTag` 的 `ids` |

章节标题、正文、样例文件 **不进这些表**。指定版本后才读对应 yaml；拉章/取样例才读 `objects/`。

方法与存储对照：

| 方法 | 读内存 | 读磁盘 |
|---|---|---|
| `search_components` | `byId` / `byTag` / `byToken` / `byDeprecatedTag` | 不读 |
| `get_component`（无版本） | `byId` | 不读 |
| `get_component`（有版本） | `byId`、`byVersion` | `{version}.chapters.yaml` 的标题列表 |
| `get_doc_section` | 不检索 | `{version}.chapters.yaml` 取 `objectKey`，再读一个对象 |
| `get_sample`（无 path） | `byVersion` 看 `sampleTestsPassed` | `{version}.samples.yaml` 的 `path`/`byteSize` |
| `get_sample`（有 path） | 同上 | 再读那一个对象 |

## 5. 四个方法的入参和出参

每节的入参都还要带第 1 节的 `callerStage`。下面只列业务字段。

### 5.1 `search_components`

允许阶段：`概要设计`、`详细设计`。

入参：

```yaml
callerStage: ""
id: ""
summary: ""
tags: []
includeDeprecated: false
```

| 字段 | 类型 | 规则 |
|---|---|---|
| `id` | 字符串 | 可空。详细设计复核时必须带概要留下的标识。有值则只做主键点查，不再用 `summary`/`tags` |
| `summary` | 字符串 | 可空。概要设计放入这次新能力的一句话 |
| `tags` | 字符串列表 | 可空。有值时每个标签都要命中 |
| `includeDeprecated` | 布尔 | 默认 `false`。为真才查停用资产 |

处理：

1. `id` 非空：只查 `byId`。没有记录，或未允许停用且 `status=deprecated` → 命中 0。结束。
2. `summary` 和 `tags` 都空 → `empty_query`。
3. `tags` 非空：每个标签取 `byTag`（或 `byDeprecatedTag`）的 `ids`，做交集。任一标签没有倒排 → 空。
4. `summary` 切词。词空则跳过。否则每个词取 `byToken` 的 `ids`，做并集。
5. 标签交集和词并集都有 → 再相交。只有标签 → 用标签交集。只有查询词 → 保留至少命中一半词的 `id`（一半向上取整）。
6. 再经 `byId` 去掉未允许的 `deprecated`。
7. 按命中词数降序，相同则按 `id` 字典序。

出参只含四个字段，不含 `gav`、正文、章节：

```yaml
ok: true
matchCount: 0
matches:
  - id: ""
    summary: ""
    tags: []
    status: active
tagFacets:
  - tag: ""
    count: 0
```

| 命中数 | `matches` | `tagFacets` |
|---|---|---|
| 0 | `[]` | 不出现 |
| 1 | 一条 | 不出现 |
| 2–20 | 全部 | 不出现 |
| 大于 20 | `[]`，`matchCount` 为总数 | 这些命中里每个标签的 `id` 个数 |

调用方：0 条沿原变更继续；1 条记下 `id` 待详细设计绑定；2–20 条停住一次一问；大于 20 条先收窄标签再查。

### 5.2 `get_component`

允许阶段：`详细设计`、`程序设计`。

入参：

```yaml
callerStage: ""
id: ""
version: ""
```

| 字段 | 规则 |
|---|---|
| `id` | 必填。必须是概要已经记下的标识 |
| `version` | 可空。空则只返回身份和版本号列表，不选“最新” |

无 `version` 时出参（只读 `byId`）：

```yaml
ok: true
id: ""
summary: ""
tags: []
maintainer: ""
status: active
versions:
  - version: ""
    sampleTestsPassed: false
```

有 `version` 时再读 `byVersion` 和 `{version}.chapters.yaml`。出参：

```yaml
ok: true
id: ""
summary: ""
tags: []
maintainer: ""
status: active
version: ""
gav: ""
docKeyTemplate: "{root}/{id}/{version}/chapters/{chapter}"
sampleTag: ""
sampleTestsPassed: false
chapters:
  - chapter: ""
    title: ""
    ordinal: 0
```

不返回 `objectKey`、`contentHash`、正文。`id` 或 `version` 没有记录 → `not_found`。

### 5.3 `get_doc_section`

允许阶段：`详细设计`、`编码`。

入参：

```yaml
callerStage: ""
id: ""
version: ""
chapter: ""
```

三个字段都必填。缺 `version` → `missing_version`。缺 `chapter` 或表示要整份文档 → `missing_chapter`。

处理：在 `{version}.chapters.yaml` 找到唯一行 → 读 `{root}/objects/{objectKey}` → 字节数必须等于 `byteSize`；`contentHash` 非空时必须对上。一次只读这一章。

出参：

```yaml
ok: true
id: ""
version: ""
chapter: ""
title: ""
byteSize: 0
contentHash: ""
content: ""
```

`content` 是该章正文。没有该章 → `not_found`，不返回目录。对象大于 262144 → `too_large`。

### 5.4 `get_sample`

允许阶段：`编码`。且程序设计已锁定版本、人已确认。

入参：

```yaml
callerStage: ""
id: ""
version: ""
path: ""
```

| 字段 | 规则 |
|---|---|
| `id`、`version` | 必填 |
| `path` | 可空。空则只返回文件列表；有值则读这一个文件 |

`sampleTestsPassed` 为假 → `sample_not_passed`。`sampleTag` 必须等于 `version`。

`path` 为空时出参：

```yaml
ok: true
id: ""
version: ""
sampleTag: ""
files:
  - path: ""
    byteSize: 0
```

`path` 有值时出参：

```yaml
ok: true
path: ""
byteSize: 0
contentHash: ""
content: ""
```

一次返回的正文合计超过 1048576 字节则 `too_large`，未读的文件留给下一次带 `path` 的调用。没有该 `path` → `not_found`。

## 6. 实现时不要做的事

- 不扫 `objects/` 做全文或向量检索。
- 不把章节正文写入 `search-index.yaml` 或内存倒排。
- 调用方没传 `version` 时，不按版本号挑选一条返回。
- 不在一次 `get_doc_section` 里返回多章或整本。
- 不把 `objectKey` 或 `{root}` 提前暴露给概要设计。
- 不在本仓库填查阅地址、对象存储地址或真实组件 `id`。

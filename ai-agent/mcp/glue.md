# glue 查阅

MCP 是调用协议。四个方法换成 HTTP 时，body 字段相同。IDE 按工具名调用，不另做一套业务。服务地址保持空白，直到人提供。本文件不写实地址、真实组件标识或令牌值。

业务代码不调用。需求、测试设计、测试执行、分提不调用。

## 1. 一次调用

```text
IDE 代理
  → MCP（或 HTTP POST），令牌来自 GLUE_TOKEN
  → 校验令牌、callerStage、glueProtocolVersion
  → 查内存表，或按路径读一个文件
  → 返回 JSON
```

| 字段 | 位置 | 说明 |
|---|---|---|
| 令牌 | `Authorization: Bearer` | 与环境变量 `GLUE_TOKEN` 比对。不进 body，不进仓库 |
| `callerStage` | body | `概要设计`、`详细设计`、`程序设计`、`编码` 之一 |
| `glueProtocolVersion` | body，成功响应也带 | 本期为 `1`。对不上则停止查阅并询问 |

取程序包用现有仓库凭证，不用 `GLUE_TOKEN`。

| 方法 | 若用 HTTP | 允许阶段 |
|---|---|---|
| `search_components` | `POST /search_components` | 概要设计、详细设计 |
| `get_component` | `POST /get_component` | 详细设计、程序设计 |
| `get_doc_section` | `POST /get_doc_section` | 详细设计、编码。程序设计仅能读该版本标明的迁移章 |
| `get_sample` | `POST /get_sample` | 编码，且程序设计已锁定版本、人已确认 |

路径不是已发布地址。

失败时：

```yaml
ok: false
code: ""
message: ""
glueProtocolVersion: 1
```

`code` 为 `missing_token`、`forbidden_stage`、`not_found`、`empty_query`、`missing_version`、`missing_chapter`、`too_large`、`sample_not_passed`、`hash_mismatch`、`yanked`、`protocol_mismatch`、`index_not_ready` 之一。

## 2. search_components

概要设计按可单独验收的一行检索。详细设计带上概要留下的标识时，只做主键点查，不再用一句话和标签重搜。

```yaml
glueProtocolVersion: 1
callerStage: ""
id: ""
summary: ""
tags: []
includeDeprecated: false
```

| 字段 | 规则 |
|---|---|
| `id` | 可空。有值则只做主键点查 |
| `summary` | 可空。概要设计放入这一行的一句话 |
| `tags` | 可空。有值时每个标签都要命中。只取这一行正文里已经写明的词 |
| `includeDeprecated` | 默认 `false`。为真，或本次需求写明保留旧资产时，才包含 `deprecated` |

`id`、`summary`、`tags` 都空 → `empty_query`。

出参只有标识、一句话、标签、状态，另有命中数。不含发布标识，不含正文。

```yaml
ok: true
glueProtocolVersion: 1
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

| 命中数 | 出参 | 调用方 |
|---|---|---|
| 0 | `matches` 为空 | 这一行沿原变更继续，不编造资产 |
| 1 | 一条 | 记下 `id`，标成待详细设计绑定 |
| 2–20 | 全部列出 | 停住，一次只问一个问题。代理不得自行挑选 |
| 大于 20 | `matches` 为空，`matchCount` 为总数，另给 `tagFacets` | 先收窄标签再查 |

## 3. get_component

`id` 必填，必须是概要已经记下或代码库里已有的标识。`version` 可空。服务不挑选较新版本。`yanked` 拒绝。

```yaml
glueProtocolVersion: 1
callerStage: ""
id: ""
version: ""
```

无版本时返回身份和版本号列表。每项带 `sampleTestsPassed`，以及该版本是否标明不兼容。

```yaml
ok: true
glueProtocolVersion: 1
id: ""
summary: ""
tags: []
maintainer: ""
status: active
versions:
  - version: ""
    sampleTestsPassed: false
    apiCompatibility: compatible
```

有版本时再返回该版本的发布标识、样例标记、是否通过、不兼容标记、迁移章，以及按顺序排列的章节标题。不返回正文。

```yaml
ok: true
glueProtocolVersion: 1
id: ""
summary: ""
tags: []
maintainer: ""
status: active
version: ""
gav: ""
docKeyTemplate: ""
sampleTag: ""
sampleTestsPassed: false
apiCompatibility: compatible
migrationDocChapter: ""
chapters:
  - chapter: ""
    title: ""
    ordinal: 0
```

`gav` 为空的版本可以出现在列表里。程序设计不得把它锁成编码依赖。

## 4. get_doc_section

`id`、`version`、`chapter` 都必填。缺版本、缺章节，或要求整份文档，都拒绝。单章大于 262144 字节拒绝。程序设计只能读 `migrationDocChapter`。

```yaml
glueProtocolVersion: 1
callerStage: ""
id: ""
version: ""
chapter: ""
```

```yaml
ok: true
glueProtocolVersion: 1
id: ""
version: ""
chapter: ""
title: ""
byteSize: 0
contentHash: ""
content: ""
```

`content` 是这一章的纯文本。编码阶段只作对照，不把整章当成已经契合本次需求的实现。

## 5. get_sample

`id`、`version` 必填。`path` 空则只返回文件清单；有 `path` 再读那一个文件。

`sampleTestsPassed` 为假，或 `sampleTag` 不等于 `version`，则拒绝。一次返回的正文合计超过 1048576 字节则停止，其余文件留下次按 `path` 再取。

```yaml
glueProtocolVersion: 1
callerStage: ""
id: ""
version: ""
path: ""
```

`path` 为空：

```yaml
ok: true
glueProtocolVersion: 1
id: ""
version: ""
sampleTag: ""
files:
  - path: ""
    byteSize: 0
```

`path` 有值：

```yaml
ok: true
glueProtocolVersion: 1
path: ""
byteSize: 0
contentHash: ""
content: ""
```

只复制样例测试已经通过、且样例标记等于锁定版本的文件。只编写相对样例还要写的那部分。样例自身的测试通过，不算本服务通过。

## 6. 不要做的事

- 不在概要设计调用 `get_component`、`get_doc_section`、`get_sample`。
- 不把多个候选的正文一起拉下来再挑。
- 不按版本号大小让服务挑选一条。
- 确认锁定之后，不自动换成更新的版本。
- 不把查阅结果回写成应用规约的「必须复用」。
- 地址空白、令牌缺失或调用失败时，不猜接口，不改用仓库里的过期文档。该能力点标成未验证并停住。

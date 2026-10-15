# Contract: 组件索引与拉取

提供方保存两层数据。第一层是可提交的源记录，一个组件一个目录。第二层是发布时生成的检索索引，服务只把这一层载入内存。文档正文和样例文件在对象存储里，不写入检索索引。

根地址、真实组件标识、真实坐标保持空白。下面的空字符串和空列表表示字段形状，不是已登记的资产。

## 磁盘布局

```text
catalog/
  source/
    {id}/
      identity.yaml
      versions/
        {version}.yaml
        {version}.chapters.yaml
        {version}.samples.yaml
  built/
    search-index.yaml
```

`{id}` 和 `{version}` 各是一段路径名，只允许字母、数字、点、下划线和连字符。`{id}` 发布后不改。

`search-index.yaml` 由全部源记录生成。生成完成前，服务继续使用上一份。生成完成后一次换上。

## identity.yaml

一条组件身份。

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
| `schemaVersion` | 字符串 | 固定 `"1"` |
| `id` | 字符串 | 与目录名相同，主键 |
| `summary` | 字符串 | 一句话能力，1 至 80 个字符。检索只对这一句切词 |
| `tags` | 字符串列表 | 发布时由人填写。每项 1 至 32 个字符，列表内不重复 |
| `maintainer` | 字符串 | 可为空 |
| `status` | 枚举 | `active`、`experimental`、`deprecated` 三值之一 |

## {version}.yaml

该组件的一个版本。同一 `id` 下版本互不覆盖。

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
| `schemaVersion` | 字符串 | 固定 `"1"` |
| `id` | 字符串 | 必须等于身份的 `id` |
| `version` | 字符串 | 与文件名 `{version}.yaml` 相同。和 `id` 组成主键 |
| `gav` | 字符串 | 该版本的发布标识。没有事实时为空 |
| `sampleTag` | 字符串 | `sampleTestsPassed` 为真时必须等于 `version`，否则为空 |
| `sampleTestsPassed` | 布尔 | 为真才允许读取样例正文 |
| `docKeyTemplate` | 字符串 | 固定为上面的模板。`{root}` 来自服务配置，在本仓库保持空白 |

服务不根据版本号大小选择记录。调用方没有传入 `version` 时，不读取任何一份 `{version}.yaml` 的 `gav`。

## {version}.chapters.yaml

这一版组件文档的索引。一行是一章，没有正文。

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

| 字段 | 类型 | 约束 |
|---|---|---|
| `schemaVersion` | 字符串 | 固定 `"1"` |
| `id` | 字符串 | 等于身份 `id` |
| `version` | 字符串 | 等于版本记录的 `version` |
| `chapters` | 列表 | 可为空。`chapter` 在本文件内不重复 |
| `chapter` | 字符串 | 章节主键，路径名规则与 `{id}` 相同 |
| `title` | 字符串 | 给人看的章节标题，1 至 80 个字符 |
| `ordinal` | 整数 | 从 0 递增，决定标题列表的顺序 |
| `byteSize` | 整数 | 正文字节数。大于 262144 的章节不得发布 |
| `contentHash` | 字符串 | 正文的 SHA-256，十六进制小写。正文未提供时为空 |
| `objectKey` | 字符串 | `{id}/{version}/chapters/{chapter}`。不写主机名 |

正文存放在 `{root}/{objectKey}`。`get_doc_section` 用 `id`、`version`、`chapter` 在这张表里找到 `objectKey`，再读那一个对象。表中没有该 `chapter` 时拒绝，不列目录。

## {version}.samples.yaml

这一版样例的索引。一行是一个文件，没有文件内容。

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

| 字段 | 类型 | 约束 |
|---|---|---|
| `schemaVersion` | 字符串 | 固定 `"1"` |
| `id` | 字符串 | 等于身份 `id` |
| `version` | 字符串 | 等于版本记录的 `version` |
| `sampleTag` | 字符串 | 与 `{version}.yaml` 的 `sampleTag` 相同 |
| `files` | 列表 | 可为空。`path` 在本文件内不重复 |
| `path` | 字符串 | 样例内的相对路径，不含 `..` |
| `byteSize` | 整数 | 文件字节数 |
| `contentHash` | 字符串 | 文件的 SHA-256，十六进制小写。可为空 |
| `objectKey` | 字符串 | `references/{id}-ref/{version}/{path}` |

`get_sample` 先返回 `files` 里的 `path` 和 `byteSize`。调用方再点名 `path`，服务读 `{root}/{objectKey}`。`sampleTestsPassed` 为假时不返回内容。一次返回的正文合计超过 1048576 字节时停止，其余文件留给下一次点名。

## search-index.yaml

发布时由源记录生成。服务启动后只读这一份做检索。

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

| 字段 | 类型 | 约束 |
|---|---|---|
| `schemaVersion` | 字符串 | 固定 `"1"` |
| `builtFrom` | 字符串 | 这次生成所依据的发布记号。没有事实时为空 |
| `components` | 列表 | 每个源目录的身份一条。`id` 不重复 |
| `components[].versions` | 列表 | 只含版本号、发布标识、样例标记和测试是否通过。不含章节正文 |
| `postings.tag` | 列表 | `status` 为 `active` 或 `experimental` 的标签倒排 |
| `postings.token` | 列表 | 由 `summary` 切出的词倒排，规则见下一节 |
| `postings.deprecatedTag` | 列表 | 只收 `status` 为 `deprecated` 的标签倒排 |
| `tag` / `token` | 字符串 | 倒排的键 |
| `ids` | 字符串列表 | 命中的 `id`，已按字典序排序，列表内不重复 |

章节索引不复制进 `search-index.yaml`。详细设计拿到 `id` 和 `version` 之后，再读 `catalog/source/{id}/versions/{version}.chapters.yaml`。

内存中的查找表由这一份文件建成：

| 表 | 键 | 值 |
|---|---|---|
| `byId` | `id` | `components` 中的那一条 |
| `byVersion` | `id` + `version` | `versions` 中的那一条 |
| `byTag` | `tag` | `postings.tag` 的 `ids` |
| `byToken` | `token` | `postings.token` 的 `ids` |
| `byDeprecatedTag` | `tag` | `postings.deprecatedTag` 的 `ids` |

## 切词

索引和查询使用同一套规则。只切 `summary`、查询句和章节 `title`。不切文档正文。

1. 去掉首尾空白。
2. 连续的英文字母和数字成一个词，英文字母改为小写。长度小于 2 的词丢掉。
3. 连续的汉字每相邻两个字成一个词。例如“订单幂等”得到“订单”“单幂”“幂等”。这句只说明切词形状，不是资产登记。
4. 标点、空白和其它符号只用来断开，本身不成为词。

`summary` 的词写入 `postings.token`。章节 `title` 的词不写入这份倒排，只在读取某一版 `chapters.yaml` 时现切。

## search_components 的检索

输入字段：

| 字段 | 类型 | 约束 |
|---|---|---|
| `summary` | 字符串 | 可为空。概要设计放入这次新能力的一句话 |
| `tags` | 字符串列表 | 可为空。有值时每个标签都要命中 |
| `id` | 字符串 | 可为空。详细设计复核时必须放入概要留下的标识 |
| `includeDeprecated` | 布尔 | 默认 `false` |

步骤：

1. `id` 非空时只查 `byId`。记录不存在则命中数为 0。存在则检查 `status`：`includeDeprecated` 为假且状态是 `deprecated` 时命中数为 0。此分支到此结束，不再使用 `summary` 和 `tags`。
2. `summary` 和 `tags` 都为空时拒绝这次检索，不扫描 `components`。
3. `tags` 非空时，对每个标签取 `byTag` 的 `ids`。`includeDeprecated` 为真时改取 `byDeprecatedTag`。这些 `ids` 做交集。任一标签没有倒排，交集为空。
4. `summary` 切词后得到查询词。查询词为空则跳过这一步。否则每个词取 `byToken` 的 `ids`，做并集。
5. 标签交集和词并集都存在时，两者再做交集。只有标签时用标签交集。只有查询词时，保留至少命中一半查询词的 `id`，一半向上取整。查询词有 4 个时，一个 `id` 至少出现在其中 2 个词的倒排里。
6. 命中的 `id` 再经 `byId` 取出 `status`。`includeDeprecated` 为假时去掉 `deprecated`。
7. 按命中的查询词个数从多到少排，个数相同则按 `id` 字典序排。

返回只带这些字段：`id`、`summary`、`tags`、`status`。

| 命中数 | 返回 |
|---|---|
| 0 | `matches: []`，`matchCount: 0` |
| 1 | `matches` 一条 |
| 2 至 20 | `matches` 全部 |
| 大于 20 | `matches` 为空，`matchCount` 为总数，`tagFacets` 为这些命中里每个标签的 `id` 数量 |

`tagFacets` 的形状：

```yaml
matchCount: 0
tagFacets:
  - tag: ""
    count: 0
```

调用方看到 0 条就沿原变更类型继续。看到 1 条就记下 `id`，标记待详细设计绑定。看到 2 至 20 条就停住，一次只问一个。看到 `tagFacets` 就先收窄标签再检索。

## get_component 读哪些字段

`get_component(id)` 只读 `byId` 和其中的 `versions`。返回：

```yaml
id: ""
summary: ""
tags: []
maintainer: ""
status: active
versions:
  - version: ""
    sampleTestsPassed: false
```

`get_component(id, version)` 再读 `byVersion` 和 `catalog/source/{id}/versions/{version}.chapters.yaml`。返回身份字段、该版本的 `gav`、`docKeyTemplate`、`sampleTag`、`status`，以及按 `ordinal` 排列的 `chapter` 和 `title`。不返回 `objectKey` 和正文。

## get_doc_section 读哪些字段

用 `id`、`version`、`chapter` 在 `{version}.chapters.yaml` 中取唯一行。读取 `{root}/{objectKey}`。返回的正文字节数必须等于该行的 `byteSize`。`contentHash` 非空时，读到的正文必须对上这个校验和，否则拒绝。

一次调用只允许一个 `chapter`。

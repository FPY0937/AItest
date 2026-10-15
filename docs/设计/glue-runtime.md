# glue 运行时：内存索引与文件读取

进程用 Java 实现。检索用的五张表在内存里，是 `HashMap`。`chapters.yaml` 和章正文不进这五张表。命中标识并指定版本之后，按路径做文件读取。

入参出参和磁盘字段见 [glue-implementation.md](./glue-implementation.md)。本文只写进程内部怎么读。

## 1. 进程里有什么

```text
volatile IndexSnapshot current     ← 检索只读这一份

IndexSnapshot
  byId              HashMap<id, 身份记录>
  byVersion         HashMap<id+version, 版本摘要>
  byTag             HashMap<tag, id 列表>
  byToken           HashMap<词, id 列表>
  byDeprecatedTag   HashMap<tag, id 列表>

ChapterCache                     ← 可选，只缓存解析后的章节目录，不缓存正文
  key = id + version
  value = 已解析的 chapters 列表
  上限 256 条，索引换上时整表丢掉
```

`IndexSnapshot` 里的 `Map` 和 `List` 建成后不可修改。一次请求开始时先把 `current` 读到局部变量，这次请求全程用这一份，避免换索引时两张表拼在一起。

身份记录来自 `search-index.yaml` 的 `components` 一条：`id`、`summary`、`tags`、`maintainer`、`status`、`versions`。版本摘要只有 `version`、`gav`、`sampleTag`、`sampleTestsPassed`。没有章节标题，没有正文。

## 2. 索引怎么装进 HashMap

只在启动和发布换索引时做，不在查阅请求里做。

1. 用 `Files.newInputStream` 读 `{root}/catalog/built/search-index.yaml`。
2. 用 SnakeYAML 解析成对象。
3. 新建一份 `IndexSnapshot`：
   - 每条 `components` 放进 `byId`。
   - 每条 `versions` 以 `id` 和 `version` 拼成键放进 `byVersion`。
   - `postings.tag` / `token` / `deprecatedTag` 各自放进对应的 `HashMap`，`ids` 保持文件里已排好的字典序。
4. 把五个 `Map` 包成不可修改。
5. 写 `current = 新快照`。写之前，查阅仍用旧快照。
6. 丢掉 `ChapterCache`。

生成新的 `search-index.yaml` 完成前不执行第 5 步。

## 3. 四个方法各自读哪里

| 方法 | 内存 | 文件 |
|---|---|---|
| `search_components` | 只查 `current` 里的 HashMap | 不读 |
| `get_component` 且没有 `version` | 只查 `byId` | 不读 |
| `get_component` 且有 `version` | `byId`、`byVersion` | 读 `{version}.chapters.yaml` |
| `get_doc_section` | 不查倒排 | 先读 `{version}.chapters.yaml`，再读章正文文件 |
| `get_sample` 不要文件内容 | `byVersion` 看 `sampleTestsPassed` | 读 `{version}.samples.yaml` |
| `get_sample` 要某个 `path` | 同上 | 再读那一个样例文件 |

`search_components` 的交集、并集、截断见 [catalog-index.md](./contracts/catalog-index.md)。那些步骤只在 `HashMap` 的 `id` 列表上做。

## 4. chapters.yaml 就是文件读取

指定了 `version` 之后，章节目录不在 HashMap 里。服务按约定路径打开那一个 YAML 文件，解析，用完即结束。下一次同一版本可以命中 `ChapterCache`，缓存里仍是目录，不是正文。

路径：

```text
{root}/catalog/source/{id}/versions/{version}.chapters.yaml
```

读取步骤：

1. `id`、`version` 必须匹配 `^[A-Za-z0-9._-]+$`。否则拒绝，不拼路径。
2. `base = root.resolve("catalog/source").normalize()`。
3. `file = base.resolve(id).resolve("versions").resolve(version + ".chapters.yaml").normalize()`。
4. `file` 必须以 `base` 开头。防止 `..` 读到目录外。
5. `Files.isRegularFile(file)` 为假则 `not_found`。
6. `Files.size(file)` 大于 1 MiB 则拒绝。章节目录只应是标题列表，不应是正文。
7. 缓存有 `id+version` 则用缓存。否则 `Files.newInputStream(file)`，SnakeYAML 解析成章节列表，放入缓存。
8. 调用方给了 `chapter` 时，在这个列表里找唯一一行。没有则 `not_found`，不列目录给调用方。

这一步用的是 `java.nio.file.Files` 读本地文件，再解析 YAML。不查数据库，不扫 `catalog/source` 目录，不打开 Docsify。

`samples.yaml` 同样：路径换成 `{version}.samples.yaml`，解析文件列表。样例目录可以沿用同一个缓存思路，键加上文件种类。上限仍按条目计，正文不进缓存。

## 5. 章正文也是文件读取

`get_doc_section` 在第 4 节拿到那一行之后，读 `objectKey` 指向的一个文件。

```text
{root}/objects/{objectKey}
```

`objectKey` 的形状是 `{id}/{version}/chapters/{chapter}`，已写在该行里。

1. `chapter` 同样只允许路径名字符。
2. 用该行的 `objectKey` 拼路径，`normalize` 后必须落在 `{root}/objects` 下面。
3. `Files.size` 大于 262144，或大于该行 `byteSize`，返回 `too_large`，不把文件读进内存。
4. `Files.readAllBytes` 读这一个文件。
5. 字节数必须等于该行 `byteSize`。`contentHash` 非空时，对读到的字节算 SHA-256，必须相同，否则 `hash_mismatch`，不把内容返回。
6. 把字节按 UTF-8 放进响应的 `content`。不缓存这份正文。

一次调用只读这一章。图片不在这个文件里。

样例正文同样：`objectKey` 为 `references/{id}-ref/{version}/{path}`，`Files.readAllBytes` 读一个文件。一次返回的正文合计超过 1048576 字节则停止。

## 6. 一次点查在进程里的顺序

详细设计先点查再拉「使用」这一章时，进程内部是：

```text
get_component(id, version)
  读局部变量 snapshot = current
  snapshot.byId.get(id)              HashMap
  snapshot.byVersion.get(id, version) HashMap
  读 chapters.yaml                    文件 → 解析 → 只把 chapter/title/ordinal 放进响应

get_doc_section(id, version, chapter)
  再读同一份 chapters.yaml            有缓存则不再读盘
  在列表里找到 chapter
  读 objects/.../chapters/usage       文件，一次
```

概要设计的 `search_components` 停在 `HashMap`。它拿不到 `chapters.yaml` 的路径，也打不开章正文。

## 7. 不采用的读法

- 不把全部 `chapters.yaml` 和章正文装进 `HashMap`。身份索引很小；正文装进内存后，手册厚度会重新决定进程大小。
- 查阅请求里不遍历 `catalog/source`。
- 不用数据库存倒排，不用搜索引擎扫正文。
- 不请求 Docsify 的 HTTP 页面来充当 `get_doc_section`。

## 8. 和 Context7 的取舍

[upstash/context7](https://github.com/upstash/context7) 的公开仓库是 MCP、SDK 和命令行。库索引和文档检索在它的服务端：库元数据用 Redis Search，文档切成片段后做向量检索。代理先 `resolve-library-id`，再按库标识和一句自然语言 `query-docs`，由模型在多条结果里挑选。

glue 沿用它的两段调用：先得到稳定标识，再取该版本的一小段文档，搜索结果里不带正文。

不沿用它的后半段实现：

- `query-docs` 按问句返回相似片段，并由模型选定库。glue 在 2 至 20 条时停住问人；拉正文时由人指定 `chapter`，一次一章。
- 不引入 Redis、向量库或嵌入模型。内部组件用标签和一句话能力，`HashMap` 加文件读取足够。
- 不增加会在任意提问时自动查阅的技能。查阅仍只发生在既有阶段允许的时机。

# Contract: 阶段包

开发组取得的阶段包是一组文件。公司现有安装入口只负责把文件放到使用方工程的 `ai-agent/` 下。它不另写一套规则正文。

## 首次放入时写入

- `ai-agent/rules/工程规约.md`
- `ai-agent/mcp/glue.md`
- `ai-agent/agents/README.md`
- `ai-agent/agents/**/rules/*公共规约.md`
- `ai-agent/agents/02_设计智能体/01_概要设计/rules/架构师确认规约.md`
- `ai-agent/agents/**/template/agent.md`
- `ai-agent/agents/**/template/eval.md`
- `ai-agent/skills/**/skill.md`，但不包含下面的排除项
- 六份应用规约，路径见 `data-model.md`

## 再次放入时

- 六份应用规约若已存在，必须跳过。
- 上表中的其余文件写回发布版本。使用方改过的公共规则副本因此不再保留；该副本本来就不是发布版本。
- 不得整目录删除 `ai-agent/` 后再拷贝。

## 不得放入阶段包

- 组件程序包、组件文档正文、样例
- 查阅服务进程、目录索引、令牌
- `ai-agent/skills/coding/code-quality-skill/archive/`
- `specs/`、`.specify/`、`.archify/`、`agent-skills/`
- `docs/交接-SDD与胶水编程.md`
- `docs/胶水编程-最终版设计-for-Cursor.md`
- `docs/规格驱动与资产复用.md`
- `docs/工程规范_A.md`
- `docs/工程规范_B.md`
- 使用方 IDE 的整份配置文件

## 使用方本地配置

使用方在自己的 IDE 配置中增加服务名 `glue`。地址字段保持空白，直到人提供。`GLUE_TOKEN` 只存在于环境变量中。阶段包不提供这份配置文件，避免覆盖使用方已有的其他服务。

地址仍空白时，`add_capability`、`new_service_or_call`、`replace_capability` 和需要读登记的 `upgrade_existing` 停住该能力点并标成未验证。真正无关的范围内改动不因地址空白而停止。

## 规则正文要落到的现有文件

实现时修改这些已有文件，不新增技能文件。

| 文件 | 写入的规则 |
|---|---|
| `ai-agent/rules/工程规约.md` | 意图、四种查阅结果、变更类型顺序、漂移分类、查阅不可用时停该能力点 |
| `ai-agent/skills/requirements/requirement-spec-skill/skill.md` | 需求不选定资产 |
| `ai-agent/skills/design/outline-design-skill/skill.md` | 先写意图；该检索的意图必须先查阅 |
| `ai-agent/skills/design/detail-design-skill/skill.md` | 完成方式、证据、与概要矛盾时返回 |
| `ai-agent/skills/design/program-design-skill/skill.md` | 版本锁定；升级点查；一次命中不回写必须复用 |
| `ai-agent/skills/coding/test-driven-skill/skill.md` | 只复制已锁定且样例测试通过的文件；验证资产成功和失败 |
| `ai-agent/skills/coding/code-review-skill/skill.md` | 重写已点名资产为阻断级；交付前再做漂移分类 |
| `ai-agent/skills/pipeline/sdd-flow-skill/skill.md` | 一次会话一个阶段；人确认通过后询问再进入下一阶段；继续编码前先分类漂移 |
| `ai-agent/mcp/glue.md` | 四个方法；MCP 与 HTTP 的 body 相同 |
| 六份应用规约 | 每个待填字段的四项说明；事实栏保持空白 |

测试技能和分提技能不增加资产查阅，只写明本阶段不调用 glue。测试执行仍不编写单元测试。

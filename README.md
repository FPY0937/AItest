# 规格驱动与资产复用

这个仓库分两块。阶段包给使用方的智能体用。设计文档给人读方案用。

```text
.
├── ai-agent/          阶段包。需求到分提的规约、智能体、技能、查阅说明
└── docs/
    ├── 设计/          本方案的设计正文，从这里读
    ├── 应用架构.md    使用方知识模板，当前没有真实系统事实
    └── …              更早的说明，以及功能、数据、案例的目录占位
```

glue 查阅进程、检索索引、组件程序包、样例正文和令牌不在本仓库。地址和确认人没有事实的，保持空白。

## 怎么读

1. 读 `docs/设计/详细设计.md`。先看第 1 节架构，再看第 4 节什么时候检索，第 5 节四个方法。
2. 读 `docs/设计/流程设计.md`。按八个阶段看每一步做什么、资产查阅发生在哪。
3. 读 `docs/设计/阅读说明.md`。里面有其余设计文件各自看什么，以及哪些旧文档不能覆盖详细设计。
4. 要看使用方实际执行的句子，再打开 `ai-agent/`。入口是 `ai-agent/rules/工程规约.md` 和 `ai-agent/mcp/glue.md`。

`docs/设计/` 里的文件若写着 `specs/001-asset-reuse-harness/`，改为看 `docs/设计/` 下的同名文件。若写着 `.specify/memory/constitution.md`，改为看 `docs/设计/constitution.md`。

## 阶段包里有什么

`ai-agent/` 是交给开发组的文件。公司安装时放到使用方工程的 `ai-agent/` 下。它不另带一套规则目录。

| 路径 | 作用 |
|---|---|
| `ai-agent/rules/工程规约.md` | 阶段门禁、意图、四种查阅结果、漂移 |
| `ai-agent/mcp/glue.md` | 四个查阅方法的入参、出参和允许阶段 |
| `ai-agent/agents/` | 八个阶段里各智能体的模板、评估和规约 |
| `ai-agent/skills/` | 各阶段怎么做。不另增技能文件 |
| `ai-agent/agents/**/rules/*应用规约.md` | 待填事实。每项写了填什么、留空问谁。事实栏是空的 |

代码质量技能下的 `archive/` 不进入阶段包。

## 八个阶段从哪份文件进入

一次会话只做一个阶段。人确认通过之后，才询问是否进入下一阶段。

| 阶段 | 打开 |
|---|---|
| 判断下一阶段 | `ai-agent/skills/pipeline/sdd-flow-skill/skill.md` |
| 需求 | `ai-agent/agents/01_需求智能体/template/agent.md` |
| 概要设计 | `ai-agent/agents/02_设计智能体/01_概要设计/template/agent.md` |
| 详细设计 | `ai-agent/agents/02_设计智能体/02_详细设计/template/agent.md` |
| 程序设计 | `ai-agent/agents/02_设计智能体/03_程序设计/template/agent.md` |
| 编码 | `ai-agent/agents/03_编码智能体/template/agent.md` |
| 测试 | `ai-agent/agents/04_测试智能体/template/agent.md` |
| 分提 | `ai-agent/skills/delivery/split-submit-skill/skill.md` |

测试设计可以在需求确认通过后，与概要、详细、程序、编码并行。测试执行和分提在编码评估通过之后。

## 使用方怎么接查阅

在自己的 IDE 配置里增加服务名 `glue`。地址保持空白，直到人提供。令牌只放环境变量 `GLUE_TOKEN`，不写入仓库。取程序包用现有仓库凭证，不用这个令牌。

业务代码不调用查阅。需求、测试设计、测试执行、分提不调用。

## 代码写在哪里

本仓库不放业务代码。一个需求跨多个代码库时，实现写在使用方工作区的 `git-repo-*` 检出里。目标库还没检出时，写在 `work/{需求项编号}/{目标库}/`。这些目录不提交进本仓库。

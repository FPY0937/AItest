# Implementation Plan: 规格驱动与资产复用阶段包

**Branch**: `001-asset-reuse-harness` | **Date**: 2026-10-12 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/001-asset-reuse-harness/spec.md`

## Summary

开发组下载一份阶段包，沿用现有八个阶段处理需求，并按模板填写本工程已经存在的事实。是否检索由子项意图决定：新增能力、替换资产、新服务或新的系统间调用才按能力检索；升级已有组件只按已知标识点查。详细设计决定完成方式，程序设计锁定版本，编码只补资产未覆盖的部分。

最终版详细设计是 [详细设计.md](./详细设计.md)，总体架构用该文第 1 节的 Mermaid。八个阶段各自做什么见 [流程设计.md](./流程设计.md)。技术做法见 [research.md](./research.md)：阶段包就是使用方工程中的 `ai-agent/` 规则与技能；远程查阅只有名为 `glue` 的提供方服务。不新增技能，不进入使用方流水线，未知地址保持空白。glue 四个方法的入参出参、磁盘记录和内存表见 [glue-implementation.md](./glue-implementation.md)。HashMap 与 `chapters.yaml` 的文件读取见 [glue-runtime.md](./glue-runtime.md)。

六张图按层下钻，先看图1再往下：

1. 总览（谁在使用方、谁在提供方）：`.archify/architecture-sdd-glue-20261013-092500/review-4/sdd-glue.html`
2. 阶段包交付什么（文件、技能分组、不进包）：`.archify/architecture-pack-20261013-095200/review-2/pack.html`
3. 八个阶段何调查阅（技能职责、工具、确认）：`.archify/workflow-stages-20261013-095200/stages.html`
4. glue 每次返回什么（调用往返）：`.archify/sequence-mcp-20261013-095200/review-4/mcp.html`
5. glue 内部数据与实现（源记录字段、内存表、四个工具读哪张表）：`.archify/architecture-glue-internal-20261013-103300/review-3/glue.html`
6. search 在内存里怎么算（切词、交集、四种返回）：`.archify/workflow-search-algo-20261013-103300/search.html`

原先的数据流图 `.archify/dataflow-index-20261013-095200/index.html` 只画出文件流向，不含字段和内存表，以图5、图6为准。

## Technical Context

**Language/Version**: 阶段包为 Markdown 规约和既有技能。查阅契约为 Markdown。使用方令牌放在环境变量中。业务代码的语言由目标库的编码应用规约决定，本功能不指定语言版本。

**Primary Dependencies**: 既有阶段技能与 `ai-agent/rules/工程规约.md`。提供方托管的查阅服务。服务地址保持空白。

**Storage**: 阶段规则存放在使用方工程的 `ai-agent/` 文件中。资产目录、文档、样例和程序包留在提供方。

**Testing**: [quickstart.md](./quickstart.md) 中的文件检查和桌面推演。不新增测试框架。

**Target Platform**: 使用方工程。代理在 IDE 中按阶段技能调用名为 `glue` 的查阅服务。

**Project Type**: 可下载的阶段规则包，加上一份查阅契约。

**Performance Goals**: 不设吞吐或延迟指标。地址缺失、调用失败或版本不一致时，停止该新服务或该新的系统间调用。

**Constraints**: 章程 1.0.0。不进入使用方合并或构建流水线。不编造地址、确认人、仓库、账号、命令和组件名。不新增技能。一次会话只处理一个阶段。

**Scale/Scope**: 每个开发组一份阶段包。资产条数未提供，不设数量上限。本次只改规则、契约和六份应用规约的填写说明。

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

Phase 0 之前，按章程 1.0.0 逐条核对。没有违规，不需要复杂性豁免。

| 章程条款 | 计划如何满足 | 结果 |
|---|---|---|
| I. 规格决定做什么，材料决定用什么做 | 需求技能不选资产。概要按意图检索，只记下标识、一句话、标签、状态。完成方式只在详细设计。版本只在程序设计锁定 | 通过 |
| II. 沿用既有阶段，不另建产品规格体系 | 交付 `ai-agent/` 阶段包。`.specify/` 不进入阶段包 | 通过 |
| III. 未知事实必须留空 | 地址、确认人、组件名、坐标在研究、数据和契约中为空 | 通过 |
| IV. 检索与决策分开 | 不新增技能。契约落在 `ai-agent/mcp/glue.md`。服务进程不进阶段包 | 通过 |
| V. 人确认，代理不签字 | 计划不增加代理签字步骤。确认只打开下一设计阶段 | 通过 |
| VI. 交付可下载 harness，不进入使用方流水线 | 计划不包含使用方流水线核对步骤 | 通过 |
| VII. 规格保鲜走既有打回 | 漂移分类后回到原阶段。不建同步轨迹 | 通过 |
| 资产复用约束 | 四种查阅结果、停用资产、一次一问、必须复用不回写，写入数据和查阅契约 | 通过 |
| 设计写作边界 | 不把六条工作流做成新系统。不交付文字识别或代码知识图谱 | 通过 |

Phase 1 之后复检：查阅契约、阶段包契约、数据模型和快速开始都遵守上表。未知地址仍为空白。复检通过。

## Project Structure

### Documentation (this feature)

```text
specs/001-asset-reuse-harness/
├── plan.md
├── 详细设计.md              # 本项目详细设计，单独可读
├── 流程设计.md              # 八个阶段各自做什么、留下什么、何时查阅
├── research.md
├── data-model.md
├── glue-implementation.md   # glue 四个方法的入参出参、记录和存放位置
├── glue-runtime.md          # 内存 HashMap 与 chapters.yaml、章正文的文件读取
├── glue-example-readme.md   # 格式示例：一份 README 如何拆进目录（非真实资产）
├── quickstart.md
├── contracts/
│   ├── glue-lookup.md
│   ├── catalog-index.md
│   └── stage-package.md
└── tasks.md             # 由 /speckit-tasks 生成，本命令不创建
```

### Source Code (repository root)

```text
ai-agent/
├── rules/
│   └── 工程规约.md
├── mcp/
│   ├── .gitkeep
│   └── glue.md
├── skills/
│   ├── pipeline/sdd-flow-skill/skill.md
│   ├── requirements/requirement-spec-skill/skill.md
│   ├── design/outline-design-skill/skill.md
│   ├── design/detail-design-skill/skill.md
│   ├── design/program-design-skill/skill.md
│   ├── coding/test-driven-skill/skill.md
│   └── coding/code-review-skill/skill.md
└── agents/
    ├── 01_需求智能体/rules/需求应用规约.md
    ├── 02_设计智能体/01_概要设计/rules/概要设计应用规约.md
    ├── 02_设计智能体/02_详细设计/rules/详细设计应用规约.md
    ├── 02_设计智能体/03_程序设计/rules/程序设计应用规约.md
    ├── 03_编码智能体/rules/编码应用规约.md
    └── 04_测试智能体/rules/测试应用规约.md
```

**Structure Decision**: 实现修改上表已有文件，并新增 `ai-agent/mcp/glue.md`。文件各自写入的句子见 [contracts/stage-package.md](./contracts/stage-package.md)。不新建技能目录，不新建业务源码树。`ai-agent/mcp/` 目前只有 `.gitkeep`。

## Complexity Tracking

章程检查没有违规。本节省略。

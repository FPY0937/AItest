# Quickstart: 规格驱动与资产复用阶段包

本指南用来验收阶段包，不包含实现代码。查阅地址尚未提供，验收不得访问任何自拟地址。

## 前提

- 仓库中已有 `specs/001-asset-reuse-harness/spec.md` 和本目录的计划产物。
- 实现完成后，`ai-agent/mcp/glue.md` 与下表中的规则文件已按 `contracts/stage-package.md` 写好。
- 环境中不必设置 `GLUE_TOKEN`。地址空白时，新服务或新的系统间调用应当停止并询问。

## 1. 发布范围检查

在仓库根目录执行。实现完成前，这些检查会失败，这是预期结果。

```powershell
$forbidden = @('glue-gate','glue-init --check','panda-test','123.12.1.21','com.example.internal')
$roots = @(
  'ai-agent\rules',
  'ai-agent\mcp',
  'ai-agent\skills\requirements',
  'ai-agent\skills\design',
  'ai-agent\skills\coding\test-driven-skill',
  'ai-agent\skills\coding\code-review-skill',
  'ai-agent\skills\pipeline',
  'ai-agent\agents'
)
foreach ($word in $forbidden) {
  Get-ChildItem $roots -Recurse -Filter *.md -ErrorAction SilentlyContinue |
    Select-String -Pattern $word -SimpleMatch
}
```

期望：没有匹配。任一命中即失败。实现完成前，`ai-agent\mcp\glue.md` 尚不存在，这条检查会因规则还没写入而无法代表最终结果。

另外确认这些路径不在准备交给开发组的文件集合里：

- `ai-agent\skills\coding\code-quality-skill\archive\`
- `specs\`
- `.specify\`
- `docs\工程规范_A.md`
- `docs\工程规范_B.md`

## 2. 应用侧模板检查

打开 `data-model.md` 中的六份应用规约。对每个仍为空的事实栏，确认同一节写明了四件事：

1. 填什么。
2. 只填已经存在的事实。
3. 留空时哪一阶段问人。
4. 哪一阶段读取。

期望：六份文件都满足。确认人、服务名、仓库、账号、命令和地址仍为空。这对应成功标准 SC-002。

## 3. 范围内改动可以不查阅资产

桌面推演，不调用查阅服务：

1. 取一条不新建服务、不新建系统间调用的改动。
2. 按需求阶段交接，确认需求正文没有选定资产。
3. 确认不需要使用方流水线增加步骤。

期望：交接可以继续。这对应 SC-001。

## 4. 新建服务的三种查阅结果

对照 `contracts/glue-lookup.md` 各走一遍，只记录规格允许的字段：

| 结果 | 概要设计应留下的记录 |
|---|---|
| 恰好一条在用或试验资产 | 候选标识，标记「待详细设计绑定」 |
| 多条 | 待确认，且只准备一个问题 |
| 零条 | 原变更类型，没有新资产名 |
| 地址空白或调用失败 | 该新服务停止；范围内改动仍可继续 |

期望：四种记录都不含版本和完成方式。这对应 SC-003。

详细设计只在第一种结果上继续。写成既有资产时记下文档名和章节。写成本服务实现时，只允许来自零条结果。这对应 SC-004。

## 5. 确认栏与必须复用

1. 检查需求确认人和架构师确认人仍为空。
2. 完成一次虚构的资产选用记录，只写在该子项的程序设计说明里。
3. 再打开程序设计应用规约的「必须复用」。

期望：确认栏在人签字前为空；「必须复用」的条目数不变。这对应 SC-005 和 SC-007。

## 6. 已有改动阻止继续编码

1. 假定已命名写入位置已有改动，且没有漂移类别。
2. 按 `ai-agent/skills/pipeline/sdd-flow-skill/skill.md` 请求继续编码。

期望：编码不开始，直到人确认类别。类别去向与 `data-model.md` 的漂移表一致。这对应 SC-008。

## 7. 阶段包内容

准备交给开发组的文件集合中，统计组件程序包、组件文档正文和样例文件。

期望：数量为 0。这对应 SC-006。

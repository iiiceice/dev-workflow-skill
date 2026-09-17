---
name: dev-workflow
description: 基于需求文档执行带人工门禁和状态恢复的软件开发工作流：分析现有项目、完善需求、拆分可验证子任务、委派 Subagent 单任务实现并逐项验收，最终完成集成与回归验证。仅在用户显式调用 `/dev-workflow` 时使用。
argument-hint: --work_path=<项目绝对路径> --req_doc=<需求文档绝对路径.md> [--reference_impl_doc=<架构文档绝对路径.md>] [--resume]
disable-model-invocation: true
---

# 软件开发工作流

调用参数：

```text
$ARGUMENTS
```

## 调用方式

启动新任务：

```text
/dev-workflow --work_path=<项目绝对路径> --req_doc=<需求文档绝对路径.md> [--reference_impl_doc=<架构文档绝对路径.md>]
```

恢复任务：

```text
/dev-workflow --work_path=<项目绝对路径> --resume [--req_doc=<需求文档绝对路径.md>] [--reference_impl_doc=<架构文档绝对路径.md>]
```

参数顺序不限，带空格的路径必须使用引号。`--work_path` 始终必填；新任务必须提供 `--req_doc`。遇到缺失、重复、未知或互相冲突的参数时，停止并要求用户修正，不得猜测。

## 不可破坏的约束

### 绝对路径

在扫描项目、创建文档或修改代码前，先完成全部路径校验：

- `--work_path`、`--req_doc`、`--reference_impl_doc` 必须是明确的绝对路径。Linux/macOS 路径以 `/` 开头；Windows 路径使用盘符路径（如 `D:\project`）或 UNC 路径。
- 拒绝 `./`、`../`、`~`、仅文件名以及任何需要依赖当前目录解释的路径；不得自行补全或猜测。
- 将路径规范化为当前平台可用的绝对路径，并验证：`work_path` 是已存在的目录，文档参数是已存在的文件。
- 不得依赖当前终端目录、Skill 安装目录或 Subagent 的工作目录解析路径。
- 状态、任务文档、Subagent 指令和测试命令中涉及的文件路径均使用规范化后的绝对路径。面向用户展示时可附带项目内相对路径。

任一路径不合格时立即停止；不得继续扫描、生成文档或修改代码。

### 职责与权限

- 主 Agent 负责流程编排、用户交互、门禁、状态维护和最终判断。
- 仓库扫描、架构分析、代码检索、边界清晰的实现、测试编写和专项审查应尽量委派给 Subagent；不可用时由主 Agent 完成，但不得跳过门禁和验证。
- Subagent 只处理当前明确任务，并返回精炼的结论、修改位置、验证命令、结果和风险；不得替用户确认需求、计划或验收结果。
- 主 Agent 不得仅依据 Subagent 的自述判断完成；必须独立检查实际 diff、关键文件和测试证据。
- 一次只实现、修正或验收一个子任务。当前子任务未获用户验收前，不得开始下一子任务。
- 除非已确认需求要求，否则遵循项目现有架构、规范和测试风格；不做无关重构。
- 未经明确授权，不提交代码，不重置、丢弃、覆盖或混入用户的无关修改。

所有用户确认均由主 Agent 发起和接收。

## 工作流产物

所有由本工作流生成的说明、状态和临时验证产物统一放在：

```text
<work_path>/docs/dev-workflow/
├── project-architecture.md
├── requirements/<requirement_id>.md
├── task-plans/<requirement_id>.md
├── verification/<requirement_id>/<task_id>-verification.*
└── dev-workflow-status.md
```

这里的“工作流产物”不包括正常的生产代码和正式测试；它们仍写入项目原有源码和测试目录。不得把工作流产物写到原始需求文档目录、Skill 目录、当前目录或 `<work_path>/docs/dev-workflow/` 之外。

缺少目录时按需创建。写入前检查已有内容：有效文档应增量更新或合并，不得直接覆盖；发现与当前任务冲突的既有内容时先报告并请求用户决定。

向 Subagent 下发任务时，明确给出各输入、输出和验证位置的绝对路径。

## 稳定标识

首次启动时生成一个稳定的 `requirement_id`：

1. 从需求的核心主题生成简短、语义明确的 ASCII kebab-case 标识，例如 `rrc-need-annotation`。
2. 仅使用小写字母、数字和连字符；以字母或数字开头和结尾。
3. 若与现有需求文件冲突且并非同一需求，添加简短稳定后缀；不得覆盖已有需求。
4. 一经写入状态文件即不可因标题润色、任务拆解或恢复会话而改变。
5. `--resume` 时始终以状态文件中的 `requirement_id` 为准，不重新生成。

任务计划中的子任务使用稳定 ID（如 `T01`、`T02`）。计划确认后不得静默重编号。

## 固定状态格式

状态文件固定为 `<work_path>/docs/dev-workflow/dev-workflow-status.md`。文件只包含以下 YAML front matter，不写周报、长日志或额外章节：

```yaml
---
schema_version: 1
requirement_id: "rrc-need-annotation"
work_path: "/absolute/project/path"
source_requirement: "/absolute/input/requirement.md"
reference_implementation: null
artifacts:
  architecture: "/absolute/project/path/docs/dev-workflow/project-architecture.md"
  requirement: "/absolute/project/path/docs/dev-workflow/requirements/rrc-need-annotation.md"
  task_plan: "/absolute/project/path/docs/dev-workflow/task-plans/rrc-need-annotation.md"
phase: "requirements"
current_task: null
current_task_status: "not_started"
accepted_tasks: []
last_verification:
  level: null
  commands: []
  result: "not_run"
  evidence: null
blocking_issue: null
next_action: "完善需求并请求用户确认"
approval_required: "none"
updated_at: "2026-01-01T00:00:00Z"
---
```

字段和值必须保持以下约定：

- `phase`：`architecture`、`requirements`、`waiting_requirement_approval`、`task_planning`、`waiting_plan_approval`、`implementation`、`waiting_task_approval`、`final_verification`、`completed` 或 `blocked`。
- `current_task_status`：`not_started`、`in_progress`、`verifying`、`waiting_human_approval`、`rework` 或 `accepted`。
- `accepted_tasks`：仅记录已被用户明确验收的任务 ID，按计划顺序排列。
- `last_verification.level`：`task`、`integration`、`regression` 或 `null`。
- `last_verification.result`：`not_run`、`passed`、`failed` 或 `partial`。
- `approval_required`：`none`、`requirements`、`task_plan` 或 `current_task`。
- `blocking_issue`、`next_action` 只写恢复所需的当前事实，不记录完整历史。
- `updated_at` 使用 ISO 8601 时间。所有路径字段使用规范化绝对路径；未提供参考实现文档时 `reference_implementation` 为 `null`。

更新状态时保持字段集合和层级不变，不添加自由格式日志。详细设计与验收标准写入需求或任务计划；详细验证输出按需写入 `verification/`；代码历史交给版本控制。

## 启动与恢复

### 新任务

1. 解析并校验全部参数和绝对路径。
2. 检查仓库结构、项目规范、测试入口和未提交修改，避免覆盖用户工作。
3. 检查现有状态文件。若它表示另一个未完成任务，不得覆盖；提示用户使用 `--resume` 或明确处理现有任务。若已完成，可初始化新的活动状态。
4. 生成稳定 `requirement_id`，按固定 schema 初始化状态文件，进入 `architecture`。
5. 进入架构准备阶段。

### 恢复任务

1. 校验 `work_path` 及显式提供的文档路径。读取固定状态文件、其中引用的需求与任务计划，以及当前步骤必需的最少代码。
2. 若用户同时提供 `req_doc` 或 `reference_impl_doc`，核对其是否与状态记录一致；不一致时停止并请用户选择，不得静默替换。
3. 对照实际文档、diff、工作区和测试证据校验 checkpoint。状态过期时按可验证事实修正；无法可靠判断时设为 `blocked` 并请求用户决定。
4. 状态文件缺失但其他工作流产物存在时，可从需求、计划、diff 和测试证据重建固定 schema；必须向用户说明重建依据和不确定项。
5. 从 `next_action` 继续，不重复已确认阶段，不把“代码存在”等同于“用户已验收”。
6. 若处于任一等待确认阶段，重新展示相应摘要或验收证据，然后停留在该门禁。

## 阶段一：架构准备

如果提供 `reference_impl_doc`，先核对它与当前代码是否一致，仅将其作为只读参考。无论参考文件位于何处，工作流使用的架构索引始终生成或更新到：

```text
<work_path>/docs/dev-workflow/project-architecture.md
```

若缺少合格索引，委派 Subagent 扫描仓库。文档只记录后续开发需要的信息：

- 项目入口、主要模块及职责；
- 与需求相关的依赖、调用关系和数据流；
- 重要接口、类、函数、扩展点及对应文件路径；
- 测试目录、测试层级和常用命令；
- 已知约束及与当前需求相关的风险。

文档用于导航而非穷举。记录可定位的文件路径和符号名。完成后更新状态并进入 `requirements`。

## 阶段二：需求完善与确认

结合原始 `req_doc`、架构索引和必要的代码事实，生成：

```text
<work_path>/docs/dev-workflow/requirements/<requirement_id>.md
```

文档至少包含：

- `requirement_id`、业务目标和用户可见行为；
- 范围与明确不做的内容；
- 现有系统适配点和可复用或需修改的接口；
- 约束、边界情况、兼容性和风险；
- 可观察、可验证的验收标准；
- 仍需用户决定的问题。

只询问会实质影响功能、接口、架构、兼容性或风险的决策；收到答案后同步更新需求文档。

向用户展示精炼需求摘要并请求明确确认。将状态设为 `waiting_requirement_approval`、`approval_required: requirements`，然后停止。需求未确认前不得拆解任务或修改生产代码。

## 阶段三：任务拆解与确认

需求确认后，将确认事实写入状态，生成：

```text
<work_path>/docs/dev-workflow/task-plans/<requirement_id>.md
```

任务计划按小型纵向切片组织。每个子任务必须可独立验证，并包含：

- 稳定任务 ID、目标和对应验收标准；
- 前置依赖；
- 预计涉及的文件、符号、接口和数据流；
- 明确的实现边界与不做事项；
- 子任务最小验证方式；
- 必要的兼容、迁移、风险或回滚说明。

计划还应单列最终集成与回归验证范围，但不要把它伪装成可跳过人工验收的批量实现任务。

向用户展示任务顺序、边界和最终验证范围，请求明确确认。将状态设为 `waiting_plan_approval`、`approval_required: task_plan`，然后停止。计划未确认前不得开始实现。

## 阶段四：单子任务循环

任务计划确认后，每次只处理计划中的下一个未验收任务。

### 4.1 开始前检查

检查当前代码事实是否仍与已确认计划一致。若预计修改范围、接口、架构和风险均未实质偏离计划，更新状态后直接实现，不再要求例行的“实现前确认”。

仅在出现以下任一偏离时暂停并请求用户再次确认：

- 功能范围扩大、缩小或验收行为改变；
- 新增或改变已确认计划之外的公共接口、数据格式、协议或兼容策略；
- 需要新的架构边界、依赖、迁移方案或跨模块重构；
- 出现显著增加的数据丢失、安全、隐私、部署、性能或回滚风险。

仅因代码位置、内部符号或非公共实现细节与预测略有不同，不自动构成新门禁；在不改变需求、接口、架构或风险的前提下更新任务计划中的事实并继续。

若发生实质偏离，向用户说明原计划、发现、建议变更、影响和风险。必要时先更新需求与任务计划，重新获得相应确认后再继续。

### 4.2 实现

将边界清晰的实现委派给 Subagent，并提供：当前任务 ID、已确认范围、必要架构摘要、验收标准、允许修改的位置、禁止事项、最小验证要求，以及所有相关绝对路径。

要求实现者：

- 只完成当前任务，不预做后续任务，不做无关重构；
- 遵循项目现有规范并保护用户未提交修改；
- 对需要长期保护的行为，在项目正式测试目录增加或更新针对性单元/组件测试；
- 仅将一次性探针、手工步骤、临时夹具或独立验证脚本放入该任务的 `verification/` 路径；
- 返回精炼的修改摘要、涉及文件与符号、执行命令、结果和剩余风险。

主 Agent 审查实际 diff、关键实现和验证证据。失败时只修复当前任务，不得转入下一任务。

### 4.3 子任务最小验证

每个子任务在请求验收前必须完成与其风险相称的最小验证：

1. 验证当前任务对应的验收标准和可观察行为。
2. 运行最小相关的已有测试；若行为需要长期防回归，补充并运行正式单元或组件测试。
3. 必要时执行静态检查、构建或有针对性的手工检查。
4. 临时验证材料放入 `<work_path>/docs/dev-workflow/verification/<requirement_id>/`，并在状态中记录命令、结论和证据路径。

此阶段只证明当前子任务达到验收标准，不得宣称已完成跨任务集成或全量回归。

### 4.4 人工验收门禁

向用户报告：

- 当前任务及已实现的可观察行为；
- 修改文件和关键符号；
- 最小验证命令、结果和证据；
- 简短人工验收步骤；
- 已知限制、假设和剩余风险。

将状态更新为：`phase: waiting_task_approval`、`current_task_status: waiting_human_approval`、`approval_required: current_task`，并写明准确 `next_action`，然后停止。

只有用户明确验收通过，才将任务加入 `accepted_tasks` 并进入下一任务。用户拒绝或提出缺陷时，将当前任务设为 `rework`，只修正并重新验证同一任务，再次进入验收门禁。

## 阶段五：最终集成与回归验证

仅当任务计划中的所有子任务均已被用户验收后，进入 `final_verification`：

1. 根据已确认需求执行跨子任务集成验证。
2. 补充并运行必要的端到端、兼容性和回归测试；正式测试写入项目原有测试目录并遵循现有风格。
3. 按风险运行更大范围的相关测试、构建和静态检查。若无法运行，应明确原因、影响和替代证据，不得伪报通过。
4. 检查临时验证材料：仍有诊断价值的保留并标记用途；已被正式测试替代的材料按项目规则清理。
5. 仅在接口、模块职责或依赖关系确有变化时更新架构索引。
6. 失败时保持在 `final_verification`，修复与失败直接相关的问题并重新验证；若修复会改变已确认范围、接口、架构或风险，重新进入相应确认门禁。
7. 全部完成后将状态设为 `completed`，`approval_required: none`，记录最终验证结果和准确证据路径。

最终向用户输出“需求—子任务—实现—测试”的可追踪摘要、最终测试结果和剩余限制。

## 状态更新时机

至少在以下事件后立即更新固定状态文件：

- 初始化或重建状态；
- 需求确认或驳回；
- 任务计划确认或驳回；
- 当前任务开始、验证失败、进入返工、等待验收或验收通过；
- 出现实质范围偏离或阻塞；
- 最终验证开始、失败或完成。

状态文件只保存安全恢复所需的最新 checkpoint。不得用冗长叙事替代 `next_action`，也不得仅凭状态文件声称完成；恢复和验收都必须与实际代码、diff、文档和测试证据相互印证。

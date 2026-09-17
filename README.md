# dev-workflow-skill
一个可恢复软件开发工作流 Skill。它通过需求确认、任务计划确认、单子任务实现、逐项人工验收和轻量状态恢复，把较长的软件开发任务约束在清晰、可验证、可继续的流程中。

## 功能

- 强制校验项目、需求和参考文档的绝对路径；
- 由主 Agent 负责流程、门禁和最终判断，将仓库分析、实现与专项验证尽量委派给 Subagent；
- 先确认需求，再确认任务计划；
- 一次只实现一个子任务，每个子任务完成后等待人工验收；
- 仅当实现范围、接口、架构或风险偏离已确认计划时，重新触发实现前确认；
- 使用稳定的 `requirement_id` 和固定 YAML 状态文件支持 `--resume`；
- 区分子任务最小验证与最终集成、端到端和回归测试；
- 将工作流文档统一保存到 `<work_path>/docs/dev-workflow/`。

## 仓库结构

```text
dev-workflow-skill/
├── README.md
└── SKILL.md
```

## 安装

### 个人安装

安装到个人 Skill 目录后，可以在所有项目中使用：

```bash
mkdir -p ~/.claude/skills
git clone https://github.com/<your-username>/dev-workflow-skill.git \
  ~/.claude/skills/dev-workflow
```

更新 Skill：

```bash
git -C ~/.claude/skills/dev-workflow pull
```

### 项目安装

如果只希望在一个项目中使用，可以将仓库作为 Git submodule 添加到项目：

```bash
mkdir -p .claude/skills
git submodule add https://github.com/<your-username>/dev-workflow-skill.git \
  .claude/skills/dev-workflow
```

也可以直接把 `SKILL.md` 复制到：

```text
<project>/.claude/skills/dev-workflow/SKILL.md
```

个人 Skill 位于 `~/.claude/skills/<skill-name>/SKILL.md`，项目 Skill 位于 `.claude/skills/<skill-name>/SKILL.md`。更多信息参见 [Claude Code Skills 文档](https://code.claude.com/docs/en/skills)。

## 调用方式

此 Skill 仅在显式调用 `/dev-workflow` 时运行。

### 启动新任务

```text
/dev-workflow --work_path="/absolute/path/to/project" --req_doc="/absolute/path/to/requirement.md"
```

提供参考架构文档：

```text
/dev-workflow --work_path="/absolute/path/to/project" --req_doc="/absolute/path/to/requirement.md" --reference_impl_doc="/absolute/path/to/architecture.md"
```

### 恢复任务

```text
/dev-workflow --work_path="/absolute/path/to/project" --resume
```

所有路径参数必须使用绝对路径。路径包含空格时，请使用引号。

## 工作流程

```text
架构准备
  → 需求完善与确认
  → 任务拆解与确认
  → 单子任务实现与最小验证
  → 人工验收
  → 下一子任务
  → 最终集成与回归验证
  → 完成
```

如果用户拒绝当前子任务的验收，工作流只返工该子任务，不会提前进入后续任务。

## 工作流产物

运行后生成的工作流文档集中保存到：

```text
<work_path>/docs/dev-workflow/
├── project-architecture.md
├── requirements/<requirement_id>.md
├── task-plans/<requirement_id>.md
├── verification/<requirement_id>/<task_id>-verification.*
└── dev-workflow-status.md
```

生产代码和正式测试仍保存在项目原有的源码与测试目录中。

## 适用场景

适合需求较长、需要分阶段确认、可能跨会话执行，或希望保留人工决策权的软件开发任务。对于只修改一两行且无需需求澄清的小任务，直接开发通常更高效。

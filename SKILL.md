---
name: skill-orchestrator
description: |
  Skill 编排器——把多个 Claude Code skill 串成声明式 workflow 的运行时。
  用户的 workflow 配置放在 ~/.claude/workflows/<name>.md，本 skill 负责发现、
  解析、执行——包括 step 派发、mode md 注入、gate 处理、loop feedback、
  state 持久化、崩溃恢复。
  触发语义：用户提到 "跑 XX workflow / 执行 XX 流水线 / /orchestrate / resume <id>"
  时加载本 skill。
---

# Skill Orchestrator

Workflow 配置放在 `~/.claude/workflows/*.md`（**不是** `~/.claude/skills/`，它不是 Claude Code Skill，而是本 skill 读的声明式配置），描述"跑哪些 skill、怎么串"；本 skill 负责"怎么跑"。

## Workflow 发现与激活

| 用户表达 | engine 行为 |
|---|---|
| `/orchestrate`（无参） | 扫 `~/.claude/workflows/*.md` 列出所有可用 workflow + description |
| `/orchestrate <name> k=v ...` | 直接加载 `~/.claude/workflows/<name>.md` 并按 k=v 填充 params |
| "跑 XX workflow" / "执行 XX 流水线" | engine 扫目录按 `name` / `description` 模糊匹配，多个命中时 AskUserQuestion 澄清 |
| "resume <workflow_id>" / "resume <name>" | 定位 `<workspace>/.autoteam/<workflow_id>/state.json` 或按 name 列出活跃 workflow，进入 §3 恢复模式 |

workflow.md 本身**不是 Claude Code Skill**——用户不会单独"加载一份 workflow"；总是先加载本 skill（自动或通过 slash command），再由本 skill 加载 workflow.md。

## 目录布局

```
~/.claude/skills/skill-orchestrator/
├── SKILL.md                   # 本文件（入口 + 索引）
├── reference/                 # 协议与 schema 详细文档（Required reads 按需 Read）
│   ├── workflow-schema.md
│   ├── execution-protocol.md
│   ├── state-schema.md
│   ├── subagent-prompt-template.md
│   ├── autoteam-result-schema.md
│   ├── lint.md
│   ├── failure-modes.md
│   ├── execution-sequence.md
│   └── workflow-example.md
├── modes/                     # 运行时注入的数据层（mode md 模板）
│   ├── subagent-execution.md
│   ├── human-gate.md
│   ├── auto-gate.md
│   ├── retry-with-feedback.md
│   └── loop-feedback.md
└── bin/
    └── lint                   # workflow 结构+语义校验脚本（零依赖 bash）
```

**路径注意**：engine 必须用**绝对路径**访问 `modes/` 和 `bin/`（如 `~/.claude/skills/skill-orchestrator/modes/subagent-execution.md`），不依赖 cwd 相对解析。

## Required reads（按场景）

**不要直接进入执行**。engine 每次处理以下场景前必须先 Read 对应 reference：

| 场景 | 必读 |
|---|---|
| 用户首次触发 workflow，即将进入 §0 预检查 | `reference/workflow-schema.md` + `reference/execution-protocol.md` + `reference/lint.md` |
| 进入 §1 按 steps 循环执行（mode md 注入、gate、goto/retry） | `reference/execution-protocol.md` |
| `mode: subagent` 派发子 agent 构造 prompt | `reference/subagent-prompt-template.md` + `reference/autoteam-result-schema.md` |
| Subagent 返回后解析 autoteam-result 块 | `reference/autoteam-result-schema.md` |
| 初始化或读写 state.json | `reference/state-schema.md` |
| 异常处理 / fallback 决策 | `reference/failure-modes.md` + `reference/execution-protocol.md` §1.5 |
| 用户 `resume <id>` 入口 | `reference/execution-protocol.md` §3 + `reference/state-schema.md` |
| 想对照时序图理解行为 | `reference/execution-sequence.md` |
| 想看完整 workflow 配置示例 | `reference/workflow-example.md`（Evaluator-Optimizer 实例） |

Engine 没 Read 过对应 reference 就进入执行 = 凭部分上下文即兴，是协议破裂的第一源。

## 顶层执行 skeleton

```
§0 预检查（workflow 启动时一次）
├─ 0.1 参数收集
├─ 0.2 lint + YAML parse
└─ 0.3 Workspace init（生成 workflow_id = NNN-<shortid>，建 state.json）

§1 按 steps 顺序循环（对每个 step）
├─ 1.1 准备（inputs / prompt_args / loop feedback 取用）
├─ 1.2 Mode 说明准备（inject_contents 组装）
├─ 1.3 按 mode 执行（main / subagent）
├─ 1.4 Gate 处理（human approve/reject/abort / auto continue/goto/abort）
├─ 1.5 Fallback（执行时异常降级到 fallback_gate）
├─ 1.6 on_fail 处理（abort / retry:N / ask_user）
└─ 1.7 成功完成本 step（清 retry / 清两个 feedback 字段 / 持久化）

§2 全部 steps 完成（status=completed / 清 loop_counts / notify）

§3 恢复模式（resume：workflow_hash 对比 + dirty step 检查）
```

**每段详细语义见 `reference/execution-protocol.md`**——进入 §0/§1.x/§2/§3 之前必须 Read。

## 关键行为规则（常用速查）

1. **Workflow 是配置文件，不是 skill**——放 `~/.claude/workflows/<name>.md`；用户通过 `/orchestrate` 或自然语言触发
2. **engine 的执行/调度不依赖用户 skill**：协议在本 skill 或 mode md；skill args 通过对话文本传，不走特殊序列化；subagent 结果走 engine 定死的 `autoteam-result` schema（通过 mode md 告诉 subagent 怎么写）
3. **Mode md 是数据，不是代码**——加/改模式改 md 即可，engine 不改
4. **一切执行时异常 fallback 到 `fallback_gate`**（MVP 只允许 `human`）——永不 hard fail
5. **结构性错误 ≠ 执行时异常**——结构错直接 abort + notify，不走 fallback（用户要改 workflow 文件）
6. **state.json 原子写入 recipe**：
   - 先 `Write("<state>.tmp", content)` — 避开 shell heredoc 转义
   - 再 `Bash("mv <state>.tmp <state>")` — 原子替换
7. **每次 state 变更都持久化**——支持随时 ctrl+c 恢复
8. **Main 模式身份切换**：加载 skill 后是执行者；skill 完成后立即切回编排者
9. **保证范围**：本地文件产物 / 可重跑 skill；外部副作用 skill crash/retry 可能重复执行，用户自负
10. **Loop cap 统一**：顶层 `max_goto_loops`（默认 10）是所有 goto 分支（auto gate / human reject goto / fallback human goto）共用的上限；超 cap → fallback to human（不 hard fail）。**Cap preflight**：超 cap 时 state 保持完整 pre-goto 态（不 stash history / 不 truncate completed_steps / 不推进 loop_counts / 不 commit `pending_feedback_gate`），仅传 `fallback_context` 给 fallback prompt
11. **Loop feedback 两字段互斥**：`state.pending_feedback`（retry 路径，走 `modes/retry-with-feedback.md`）/ `state.pending_feedback_gate`（goto 路径，走 `modes/loop-feedback.md`）——绝不同时有值。**切换时原子清对方**：§1.4 retry 分支写 `pending_feedback` 时同时 `pending_feedback_gate = null`；Backward goto commit `pending_feedback_gate` 时同时 `pending_feedback = null`。**§1.7 成功通过 gate 后两字段都清**（覆盖正常成功 + max_retries 超限后用户选 skip 分支）。`loop-feedback.md` 的占位符统一 `{{decision_artifact}}`

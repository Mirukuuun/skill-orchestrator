# Skill Orchestrator

**Claude Code skill 的声明式编排引擎——用 markdown 把多个 skill 串成 workflow，自动跑、自动恢复、自动兜底。**

> 把多个 Claude Code skill 串成一条声明式流水线，交给 orchestrator 执行：
> 失败降级、人工卡点、回跳重跑、崩溃续跑，全都帮你处理好。

一句话说清楚：**Skill 是"一件事怎么做"，Workflow 是"多件事怎么串"，Orchestrator 是"这条线怎么跑"。**

---

## 为什么要有它

Claude Code 的 [Skill](https://docs.claude.com/en/docs/claude-code/skills) 把一类任务的做法封装成可复用单元（`~/.claude/skills/<name>/SKILL.md`）：写设计文档、跑代码评审、生成测试……每个 skill 只做好一件事。

但真实工作常常是**多个 skill 的组合**：

```
写设计文档 → 基于设计开发 → 跑代码评审 → 评审不过回到开发重做
```

手工串这条线问题很多：

- 漏步骤 / 顺序乱 / 上下文互相污染
- 没法跨会话恢复，Ctrl+C 之后前功尽弃
- 评审不过要手动把报告贴回开发 prompt
- 失败没有兜底，报错就卡住

**Skill Orchestrator 就是这条线的运行时。** 你用一份 markdown 声明要串哪些 skill、谁依赖谁、哪里需要人工卡点、哪里要循环重跑，orchestrator 负责：

- 按顺序派发 step（主会话或 subagent）
- 在 gate 卡点等人工审核 / 自动读判决字段
- 循环回跳时把上一轮的评审产物作 feedback 自动注入
- 每次状态变更都原子写 `state.json`，随时可以 resume
- 执行时异常统一降级到 fallback gate，**永不 hard fail**
- 结构性错误（workflow 文件写错）精确报错 + 直接 abort

## 30 秒看懂

一份 workflow 长这样（`~/.claude/workflows/coding.md`）：

````markdown
---
name: coding
description: 设计 → 开发 → 评审；评审不过自动回跳重跑
autoteam_version: "0.1"
params:
  feature:
    type: string
    required: true
fallback_gate: human
max_goto_loops: 3
---

# Coding Workflow

## Steps

### design

```yaml
skill: design-gen
mode: main
prompt_args:
  feature: "{{params.feature}}"
outputs:
  - name: design_doc
    path: "{{workspace.root}}/artifacts/design.md"
on_fail: ask_user
gate_after:
  kind: human
  prompt: "设计 OK？"
  on_reject: retry
  max_retries: 3
```

### code

```yaml
skill: code-dev
mode: subagent
inputs:
  - from: design.outputs.design_doc
    as: design_doc
outputs:
  - name: code_changes
    path: "src/**"
on_fail: retry:2
```

### review

```yaml
skill: code-review
mode: subagent
inputs:
  - from: code.outputs.code_changes
    as: code_changes
outputs:
  - name: review_report
    path: "{{workspace.root}}/artifacts/review.md"
on_fail: ask_user
gate_after:
  kind: auto
  decision:
    output_file: "{{outputs.review_report.path}}"
    field: verdict
    values:
      pass: continue
      fail: goto:code
```
````

跑它：

```
/orchestrate coding feature="JWT 认证"
```

orchestrator 的动作：

1. 主会话加载 `design-gen` 写设计文档，停下来让你 review（human gate）
2. 你 approve 后，派一个干净 context 的 subagent 加载 `code-dev`，基于设计开发
3. 再派 subagent 加载 `code-review` 跑评审；评审 markdown 的 frontmatter `verdict: pass` 就收工，`verdict: fail` 自动回跳到 `code` step，把评审报告作 feedback 注入，最多循环 3 轮
4. 中间随时 Ctrl+C，下次 `resume coding` 从断点续跑

## 核心能力

### 两种执行模式

| 模式 | 适用 | 说明 |
|---|---|---|
| `main` | 需要和你交互的 step（设计评审、需求澄清） | skill 在当前主会话里跑，保留讨论延续性 |
| `subagent` | 机械化 step（代码生成、代码评审、测试生成） | 派一个干净 context 的 subagent，隔离 token、防止上下文污染 |

### 两种 gate（卡点）

| Gate | 触发 | 行为 |
|---|---|---|
| `human` | step 完成后停下来等你 | 选 `approve` / `reject` / `abort`；reject 时可以带反馈、选 `retry`（同 step 重跑带反馈）或 `goto:<step>`（回跳上游 step 重做） |
| `auto` | orchestrator 读产出文件的根级判决字段 | 字段值对照 `values` 表：`continue` 前进 / `goto:<step>` 回跳 / `abort` 终止 |

### Evaluator–Optimizer 反馈循环

任何 gate 触发 `goto:<step>` 时，orchestrator 自动把**触发 gate 的产物**（评审报告路径 / 用户反馈文本）作为 `{{decision_artifact}}` 注入 target step 的下一轮——workflow 作者**不需要**在 target step 的 `inputs` 里显式声明这个反馈。

`max_goto_loops`（默认 10）是所有 goto 循环的全局上限，超过就降级到 human gate 让你裁决，**不会 hard fail**。

### 状态持久化 & 崩溃恢复

- 每次状态变更原子写到 `<项目根>/.autoteam/<workflow_id>/state.json`
- `resume <name>` 或 `resume <workflow_id>` 从断点续跑
- Workflow 文件自上次执行后被改过？orchestrator 检测 hash 变化，让你选「按旧 state 续 / 从头重跑新版 / abort」

### Fallback 保证

| 异常 | 处理 |
|---|---|
| Subagent 回复缺 `autoteam-result` 块 / 格式错 | → fallback 到 human gate |
| Auto gate 字段读不到 / 值不在 `values` 表 | → fallback 到 human gate |
| 声明的 outputs 文件未出现（glob 0 匹配） | → fallback 到 human gate |
| Loop 超 `max_goto_loops` | → fallback 到 human gate |
| Workflow 文件结构错（缺字段 / goto 目标不存在 / skill 未安装等） | 直接 abort + 精确定位（**不**走 fallback，因为要你改文件） |

**执行时异常永不 hard fail；结构性错误直接 abort。** 两条线分得清清楚楚。

## 安装

```bash
git clone <this-repo> ~/.claude/skills/skill-orchestrator
```

然后把你的 workflow 放到 `~/.claude/workflows/` 下。Claude Code 在下一次会话自动识别。

`/orchestrate` slash command 的定义在 `~/.claude/commands/orchestrate.md`——仓库里有一份参考，复制过去即可。

## 触发方式

| 用户输入 | 行为 |
|---|---|
| `/orchestrate` | 列出 `~/.claude/workflows/` 下所有 workflow |
| `/orchestrate <name>` | 跑指定 workflow；缺必填参数会问你补 |
| `/orchestrate <name> k1=v1 k2=v2` | 带参跑 |
| 「跑 xxx workflow」 / 「执行 xxx 流水线」 | 自然语言触发，按 name / description 模糊匹配，多命中时问你澄清 |
| `resume <workflow_id>` / `resume <name>` | 从 state.json 断点续跑 |

## Workflow 结构速查

### Frontmatter

| 字段 | 必填 | 说明 |
|---|---|---|
| `name` | ✓ | 必须等于文件名（`coding.md` → `name: coding`） |
| `description` | ✓ | 一句话功能说明 |
| `autoteam_version` | ✓ | schema 版本，当前 `"0.1"` |
| `params` | | workflow 参数声明（每条含 `type` / `required` / `default`） |
| `workspace` | | 工作区配置（`root` / `state_file`，支持 `{{workflow_id}}`） |
| `fallback_gate` | | 目前仅支持 `human` |
| `max_goto_loops` | | goto 循环全局上限（默认 `10`） |

### Step 字段

| 字段 | 必填 | 说明 |
|---|---|---|
| `skill` | ✓ | 对应 `~/.claude/skills/<name>/SKILL.md` 必须存在 |
| `mode` | ✓ | `main` / `subagent` |
| `on_fail` | ✓ | `abort` / `ask_user` / `retry:<正整数>` |
| `inputs` | | `[{from: <step_id>.outputs.<name>, as: <local_name>}]` |
| `outputs` | | `[{name, path}]`；path 三种写法见下 |
| `prompt_args` | | map，传给执行者的参数（支持模板变量） |
| `gate_after.kind` | | `human` / `auto` |
| `gate_after.on_reject` | | human gate 用：`retry` / `goto:<step>` |
| `gate_after.decision` | | auto gate 必填：`{output_file, field, values}` |
| `gate_after.prompt` | | human gate 给用户看的提示文字 |

### Output path 三种写法

| 写法 | 含义 | 展开位置 |
|---|---|---|
| `{{workspace.root}}/...` | workflow 内部中间产物 | `<项目根>/.autoteam/<workflow_id>/...` |
| `src/**`（相对路径） | 项目文件变更 | `<项目根>/...` |
| `/...`（绝对路径） | 逃生门 | 原样使用 |

支持单文件 / 目录 / glob。

### 模板变量命名空间

| 格式 | 作用 |
|---|---|
| `{{params.<name>}}` | 从 frontmatter `params` 取值 |
| `{{workspace.<root\|state_file\|workflow_id>}}` | 从 workspace 配置取 |
| `{{outputs.<output_name>.path}}` | 仅本 step 自己的 outputs（跨 step 必须走 `inputs.from`） |

## 目录布局

```
skill-orchestrator/
├── SKILL.md                          入口 + 导航索引
├── reference/                        协议与 schema 详细文档
│   ├── workflow-schema.md            workflow 配置文件结构
│   ├── execution-protocol.md         执行协议（§0 预检查 / §1 step 循环 / §2 收尾 / §3 resume）
│   ├── state-schema.md               state.json 结构
│   ├── subagent-prompt-template.md   subagent 启动 prompt 模板
│   ├── autoteam-result-schema.md     subagent ↔ engine 结果块契约
│   ├── lint.md                       lint 脚本说明
│   ├── failure-modes.md              失败模式汇总
│   ├── execution-sequence.md         执行时序图
│   └── workflow-example.md           完整 workflow 示例
├── modes/                            运行时注入的 mode md（数据层）
│   ├── subagent-execution.md
│   ├── human-gate.md
│   ├── auto-gate.md
│   ├── retry-with-feedback.md
│   └── loop-feedback.md
└── bin/
    └── lint                          workflow 校验脚本（零依赖 bash）
```

## 设计原则

1. **Workflow 是数据，不是代码。** 编排逻辑在 orchestrator 里；用户只写声明式 markdown。
2. **Orchestrator 不依赖用户 skill 的实现。** 协议只活在 orchestrator 和 mode md 里；skill 参数通过对话文本传，subagent 结果走固化的 `autoteam-result` schema。加新 skill 不需要改 orchestrator。
3. **Mode md 是数据，不是代码。** 要加 / 改执行模式就改 md，orchestrator 不需要重写。
4. **永不 hard fail。** 执行时异常都降级到 `fallback_gate`（human）。
5. **结构错 ≠ 执行错。** Workflow 文件写错直接 abort 让你改文件，不进 fallback。
6. **每次状态变更都持久化。** 随时可以 Ctrl+C，下次接着跑。
7. **路径契约 declared-form 统一。** state 与 subagent 结果里都存 `<path as declared>` 字符串（模板变量已展开，但 glob 不展开、相对路径不绝对化）——只有 engine 内部做 glob 验证时才临时绝对化。

## 局限 & 不保证（MVP）

- **只保证本地文件产物 / 可重跑 skill**。外部副作用（调外部 API、发消息、DB 写入）在 crash / retry / goto 时可能重复执行，用户自担
- `fallback_gate` 当前仅支持 `human`（未来会开放 `abort`）
- Auto gate `decision.field` 只支持根级字段（嵌套路径如 `status.verdict` 推到 v1.x）
- `params` 只支持 block form（不支持 inline flow-map `{...}`）

## 校验工具

Workflow 文件校验器（零依赖 bash）：

```bash
~/.claude/skills/skill-orchestrator/bin/lint ~/.claude/workflows/coding.md
```

- 退出码 `0` = 合法
- 非 `0` = stderr 里是 `LINT ERROR: ...` 精确定位（哪行 / 缺什么字段 / goto 目标不存在 / skill 未安装等）

Orchestrator 启动时（§0.2）也会跑同一个 lint。你也可以接到自己的 CI 里防 workflow 回归。

## FAQ

**Q：Workflow 本身是不是一个 Skill？**

不是。Workflow 是 orchestrator 读的**声明式配置文件**，放在 `~/.claude/workflows/`（而不是 `~/.claude/skills/`）。Claude Code 不会直接「加载一份 workflow」，而是加载 `skill-orchestrator` skill，再由它加载 workflow 文件。

**Q：为什么不让 Claude 直接串多个 skill？**

可以，但你会失去跨会话恢复、自动重试、人工卡点、反馈自动回灌、状态审计这些基础设施。Orchestrator 把这些一次性做好，你只用关心「要串哪些 skill」。

**Q：Workflow 写错了会怎样？**

Lint 和 orchestrator 的预检查阶段会精确报错——哪一行缺哪个字段、goto 目标是否存在、skill 是否安装，一条条列出来。**直接 abort 让你改**，不会进 fallback 流程（因为这不是执行时问题，是配置文件问题）。

**Q：跑到一半崩了 / 误按 Ctrl+C？**

`resume <name>` 就行。state.json 完整记录了到哪一步、每个 step 的产出路径、当前循环计数等，从崩溃点下一步接着跑。如果 workflow 文件自上次执行后被改过，orchestrator 会检测到 hash 变化，让你选「按旧 state 续 / 从头重跑 / abort」。

**Q：Gate 可以写自定义 options（比如 `approve / escalate / defer`）吗？**

不行。Human gate 的 options 固定为 `[approve, reject, abort]`，workflow 不得自定义——这是 engine 的稳定契约。自定义分叉请用 `on_reject: goto:<step>` 配合多个 step 表达。

**Q：一个 step 可以有多个 goto 目标？**

Auto gate 通过 `values` 表可以声明多个分支（`pass: continue`、`fail: goto:code`、`need_redesign: goto:design`）。Human gate 一次 reject 只能走一个 `on_reject` 配置（`retry` 或 `goto:<step>`）。

**Q：外部副作用 step（发邮件、call API）怎么办？**

MVP 不做幂等性标注——这类 step 在 crash / retry / goto 时**可能被重复执行**，用户自己保证幂等性或者在 skill 里做去重。未来版本会考虑引入显式幂等性声明。

## 相关文档

- [SKILL.md](SKILL.md) — skill 入口与导航
- [reference/workflow-schema.md](reference/workflow-schema.md) — workflow 配置文件结构详解
- [reference/execution-protocol.md](reference/execution-protocol.md) — 执行协议完整细节
- [reference/workflow-example.md](reference/workflow-example.md) — 完整可跑的 coding-workflow 示例
- [reference/execution-sequence.md](reference/execution-sequence.md) — 执行时序图

## License

[MIT](LICENSE)

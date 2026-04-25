# Workflow 配置文件结构与支持契约

本文档描述 `~/.claude/workflows/<name>.md` 的结构、frontmatter 字段、body 格式、解析规则，以及 engine 对 workflow 的约束（路径、模板变量、gate options 等）。

Engine 启动时会 Read 本文件来理解 workflow 如何组织；生成 workflow 的 `orchestrator-workflow-creator` skill 也应参考本文档。

---

Workflow 配置放在 `~/.claude/workflows/<name>.md`。

### Frontmatter 字段

**必需**：

| 字段 | 说明 |
|---|---|
| `name` | workflow 标识，**必须等于文件名**（`coding-workflow.md` → `name: coding-workflow`） |
| `description` | 一句话功能说明，给用户看 |
| `autoteam_version` | schema 版本（当前 `"0.1"`） |

**可选**：

| 字段 | 说明 |
|---|---|
| `params` | workflow 参数声明（`name / type / required / default`） |
| `workspace` | 工作区配置（`root` / `state_file`），支持 `{{workflow_id}}` |
| `fallback_gate` | MVP **只允许 `human`**（`abort` 是非法值，0.2 校验拒绝） |
| `max_goto_loops` | **Goto 循环全局上限**（默认 `10`）；所有 goto 分支（auto gate / human reject goto / fallback goto）共用；超限 → fallback to human |

### Body 结构

```markdown
# <Workflow 名>

<业务描述>

## 流程

<可选 ASCII 流程图，engine 忽略>

## Steps

### <step-id-1>

<业务说明>

```yaml
skill: <...>
mode: main | subagent
# ...其他字段
```

### <step-id-2>
...

## 用法 / 依赖 / 恢复

<其他章节，engine 忽略>
```

### Engine 解析规则

1. **Step 列表**：body 中第一个 `## Steps` 二级章节下的 H3 子章节（必须有）
2. **单个 step**：
   - `step.id` = H3 标题（trim 后，必须唯一）
   - `step.config` = 该 H3 下**唯一** YAML fenced block（` ```yaml `）的解析结果
   - 多个 YAML fenced block → 结构性错误（用 ` ```yml-example ` 等非 `yaml` 标签写示例）
3. **其他 body 章节**（`## 流程` / `## 用法` 等）engine 不读

### 结构性错误（直接 abort，不走 fallback）

| 类别 | 示例 |
|---|---|
| 章节缺失 | 缺 `## Steps` |
| YAML 块异常 | H3 下无 `yaml` fenced block；多个 `yaml` fenced block；YAML 语法错 |
| step 标识 | id 重复；goto 目标不存在于 workflow |
| 字段非法 | `mode` 非 `main`/`subagent`；`gate_after.kind` 非 `human`/`auto`；`fallback_gate` 非 `human` |
| 模板变量 | 预检查阶段未解析的 `{{var}}`（见"模板变量命名空间"） |
| skill 引用 | `~/.claude/skills/<skill_name>/SKILL.md` 不存在 |

**结构性错误 ≠ 执行时异常**。结构错了要用户改 workflow 文件；fallback_gate 只处理执行时异常（§1.5）。

---

## 支持契约（engine 对 workflow 的约束）

### 范围约定

**MVP 只保证本地文件产物 / 可重跑 skill**。外部副作用（调外部 API、发消息、DB 写入）**不在 MVP 保证范围**——这类 skill 在 crash/retry/goto 时可能重复执行，用户自担。未来版本可能引入幂等性标注，MVP 不做。

### 模板变量命名空间（唯一合法集）

| 格式 | 作用 |
|---|---|
| `{{params.<name>}}` | 从 workflow `params` 取参数值 |
| `{{workspace.<root\|state_file\|workflow_id>}}` | 从 workspace 配置取 |
| `{{outputs.<output_name>.path}}` | **仅本 step 自己的 outputs**；禁止跨 step 引用（跨 step 必须走 `inputs.from`） |

预检查（§0.2）阶段全量 resolve；任何未解模板变量 → 结构性错误。

### Artifact / Input 规则

#### `inputs[].from` 格式

**唯一**合法格式：`<step_id>.outputs.<output_name>`（引用上游 step 的 outputs）

#### `outputs[].path` 三种写法

engine 按字符串开头判断路径含义：

| 写法 | 含义 | 展开结果 |
|---|---|---|
| `{{workspace.root}}/...` | **workflow 内部 artifact**（design.md / review.md 等中间件） | `<项目根>/.autoteam/<workflow_id>/...` |
| 相对路径（不以 `/` 开头、不带 `{{workspace.root}}`，如 `src/**`） | **项目文件变更** | `<项目根>/...`（按项目根展开） |
| 绝对路径（`/` 开头） | 逃生门 | 直接用 |

支持单文件 / 目录 / glob（如 `src/**`）。

#### 项目根 & workflow_id

- **项目根** = engine 启动时的 Claude Code cwd（用户总在项目目录里触发 workflow，这是自然契约）
- **`workflow_id`** 格式：`NNN-<shortid>`（三位顺序号 + 8 位短 id，例 `001-a3b9c7d2`）
- engine 在 §0.3 初始化时扫 `<项目根>/.autoteam/` 取已存在 workflow 最大序号 + 1 作 NNN；shortid 用 `uuidgen | head -c 8` 或类似

#### Glob 在 state 里的形态

`state.step_results[X].outputs[name]` 存 **"path as declared"** 字符串——模板变量已展开，但 **glob 不展开、相对路径不绝对化**（保留 workflow.md 里的原形态）。例如：

```yaml
# workflow.md
outputs:
  - name: code_changes
    path: "src/**"
```

→ state 里记录（**照抄 declared 字符串**，不预绝对化）：

```json
"outputs": { "code_changes": "src/**" }
```

再例：`path: "{{workspace.root}}/artifacts/design.md"` 模板展开得到 `.autoteam/001-a3b9c7d2/artifacts/design.md`（项目相对），state 里就存这个字符串，也不拼项目根前缀。

下游 step 的 `inputs.from` 拿到 declared 字符串后，**skill 自己按路径三写法规则解析 + glob 展开读文件**（engine 不绝对化，只有 engine 内部 glob 匹配校验 / dirty 清理时临时展开）。

#### Glob 未匹配 / dirty 清理

- **完成度校验**：step 完成时 engine 对每个声明 outputs 的 path 做一次 glob，**0 匹配** = "outputs 未出现" → §1.5 fallback
- **dirty step 清理**（§3）：resume 时若用户选"清 outputs 重跑"，engine 对 declared outputs 做 glob、列清单给用户 AskUserQuestion 再确认，然后删除

### Step 字段清单

每个 step H3 下 YAML block 的合法字段：

| 字段 | 必需 | 说明 |
|---|---|---|
| `skill` | 是 | 要加载的 skill 名（对应 `~/.claude/skills/<name>/SKILL.md` 存在） |
| `mode` | 是 | `main`（主会话） / `subagent`（派子 agent） |
| `on_fail` | 是 | `abort` / `ask_user` / `retry:<正整数>` |
| `inputs` | 否 | 列表，每条 `{from: <step_id>.outputs.<output_name>, as: <local_name>}` |
| `outputs` | 否 | 列表，每条 `{name: <output_name>, path: <path as declared>}`；**单 step outputs 内部 name 不能重复** |
| `prompt_args` | 否 | map，传给执行者的参数（模板变量会展开）；main 模式通过对话文本传，subagent 模式注入启动 prompt |
| `gate_after` | 否 | 若存在 `gate_after.kind` 必填，取值 `human` / `auto` |
| `gate_after.on_reject` | 否 | human gate 用：`retry` / `goto:<step_id>` |
| `gate_after.max_retries` | 否 | human gate `on_reject: retry` 时的上限（N3 与顶层 `max_goto_loops` 互补） |
| `gate_after.mode_md` | 否 | 覆盖 engine 默认的 mode md（指向自定义 md 路径） |
| `gate_after.extra_instructions` | 否 | 字符串，追加到 mode md 末尾 |
| `gate_after.decision` | auto gate 必填 | 含 `output_file` / `field` / `values`；见下方 "Auto gate 字段提取规则" |
| `gate_after.prompt` | 否 | human gate 的提示文字（给用户看） |

### Gate options 固定

`gate_after.kind: human` 的 options **固定为 `[approve, reject, abort]`**，workflow 不得自定义。

### Auto gate 字段提取规则（MVP）

`decision.field` **只支持根级字段 + case-sensitive**：
- Markdown 文件 → 读 frontmatter YAML 的根级字段
- JSON/YAML 文件 → 读根级字段
- 嵌套路径（如 `status.verdict`）、大小写模糊 → 结构性错误（0.2 校验拒绝）

嵌套路径支持推到 v1.x。

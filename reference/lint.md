# Engine 内置 lint

Workflow 结构+语义校验器，**零依赖**（纯 bash + grep + sed + awk）。两个时点调用：

1. **`skill-workflow-creator` 生成 workflow 后**：保证 creator 不产出坏文件
2. **engine §0.2 首步**：保证磁盘上当前文件合法（可能被用户手改或外部 merge 破坏）

### 调用方式

脚本位于 `~/.claude/skills/skill-orchestrator/bin/lint`，由 engine（或 creator）通过 Bash 工具调用：

```
Bash("~/.claude/skills/skill-orchestrator/bin/lint <workflow_path>")
```

退出码 `0` = 合法；非 `0` = 失败，stderr 是逐条 `LINT ERROR: ...` 精确定位。engine 把 stderr 内容原样 notify 给用户然后 abort。

### 覆盖范围

**结构层**：
- 文件存在 + **BOM 自动 strip**（UTF-8 BOM 前缀容忍）
- frontmatter `---...---` 闭合
- 必填顶层字段：`name` / `description` / `autoteam_version`
- `name` 等于文件名（去 `.md`）
- `fallback_gate` 若存在仅 `human`
- `max_goto_loops` 若存在必为正整数
- `## Steps` 章节存在
- 每个 H3 下恰好 1 个 ` ```yaml ` block
- ` ```yaml ... ``` ` 闭合
- step.id 唯一

**语义层**：
- **必填** step 字段：`skill` / `mode` / `on_fail`；若 step 声明 `gate_after` 则 `gate_after.kind` 必填
- `skill` 对应的 `~/.claude/skills/<name>/SKILL.md` 存在
- `mode` ∈ `{main, subagent}`
- `gate_after.kind` ∈ `{human, auto}`
- `on_fail` ∈ `{abort, ask_user, retry:<正整数>}`
- `decision.field` 非嵌套（不含 `.`）
- `goto:<target>` 目标存在于 workflow
- `{{params.X}}` 引用的 X 在 frontmatter `params` **第一级缩进**的 key 里声明（不把 `type` / `required` / `default` 这类嵌套字段误当 params 名）

**不覆盖（交 engine §0.2 LLM parse 时自然发现）**：
- YAML 语法错（缺引号、缩进错、anchors 坏）

**MVP 限制（lint 主动拒绝）**：
- Inline `params: {...}` flow-map 不支持——必须用 block form（`params:` 换行 + 缩进的 `<name>:` + 缩进的字段）。原因：shell 精确 parse inline flow-map 超出 MVP ROI；block form 更可读、也是绝大多数 workflow 的自然写法

### 关系

"启动前明确报错"的保证 = lint（确定性工具，一刀切一大半错误）+ §0.2 LLM parse（兜底 YAML 语法错）。lint **不是**唯一防线，但它是便宜、确定、不吃 token 的第一道。

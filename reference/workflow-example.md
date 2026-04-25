---
name: coding-workflow
description: 设计 → 开发 → 审查 的自动化流水线（设计人工 review；审查不过自动回 code 重跑，最多 3 次）
autoteam_version: "0.1"

params:
  feature_name:
    type: string
    required: true
  target_language:
    type: string
    default: typescript

workspace:
  root: ".autoteam/{{workflow_id}}"
  state_file: ".autoteam/{{workflow_id}}/state.json"

fallback_gate: human
max_goto_loops: 3
---

# Coding Workflow

> **文件位置**：`~/.claude/workflows/coding-workflow.md`
> **本文件不是 Claude Code Skill**，而是 `skill-orchestrator` engine 读取的声明式配置。用户通过 `/orchestrate` 或自然语言触发执行。
>
> **路径语义**（见 engine `§Artifact / Input 规则`）：
> - `{{workspace.root}}/artifacts/*.md` → workflow 内部 artifact，落到 `<项目根>/.autoteam/<workflow_id>/artifacts/*.md`
> - `src/**` → **项目相对路径**（不带 `{{workspace.root}}` 前缀），落到 `<项目根>/src/**`（这是 workflow 对项目文件的变更）
> - `/` 开头 → 绝对路径

自动化的编码流水线：**设计 → 开发 → 审查**，审查不通过自动回到开发重跑。

## 流程

```
design (main, human gate)
   │
   ▼
code (subagent)
   │ ◄─────┐
   ▼       │ (engine 自动把 review.md 作 feedback 注入，C4)
review (subagent, auto gate)
   ├─ verdict=pass → ✓ done
   └─ verdict=fail → goto code  (max 3 loops)
```

## Steps

### design

设计文档生成。在主会话里跑（保留讨论延续性），完成后停下让你 review。

```yaml
skill: design-gen
mode: main
prompt_args:
  feature: "{{params.feature_name}}"
  language: "{{params.target_language}}"
outputs:
  - name: design_doc
    path: "{{workspace.root}}/artifacts/design.md"
on_fail: ask_user
gate_after:
  kind: human
  prompt: "设计文档已完成，请 review。"
  on_reject: retry
  max_retries: 3
```

### code

代码开发。独立 subagent 执行，context 干净。失败自动重试 2 次。

```yaml
skill: code-dev
mode: subagent
inputs:
  - from: design.outputs.design_doc
    as: design_doc
prompt_args:
  feature: "{{params.feature_name}}"
outputs:
  - name: code_changes
    path: "src/**"
on_fail: retry:2
```

> **Loop feedback（engine 自动处理）**：当 review 判 fail 触发 `goto:code` 时，engine 自动把 `review.md` 作为 feedback 追加到 code step 的下次执行 prompt——**workflow 作者不需要在 `inputs` 里声明 review.md**。

### review

代码审查。auto gate 读 `review.md` 的 `verdict` 字段自动判断：pass 完成，fail 回到 code 重跑。

```yaml
skill: code-review
mode: subagent
inputs:
  - from: design.outputs.design_doc
    as: design_doc
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

> **Loop cap**：goto 循环上限由顶层 frontmatter `max_goto_loops` 控制（本 workflow 设为 3），**不在 per-gate 上配**。所有 goto 分支（auto gate / human reject goto / fallback goto）共用这一个计数。

## 用法

无参触发（engine 会问你补填必填 params）：

```
/orchestrate coding-workflow
```

全参触发：

```
/orchestrate coding-workflow feature_name="JWT 认证" target_language=typescript
```

自然语言触发：

```
跑一下 coding-workflow，实现 JWT 认证
```

## 依赖 skill

- `design-gen`
- `code-dev`
- `code-review`

必须已存在于 `~/.claude/skills/`，否则 engine 启动时 abort（§0.2 结构性错误）。

## 恢复

中断后：

```
resume coding-workflow
```

engine 会列出该 workflow 下所有活跃 run 让你选；或直接指定 id：

```
resume coding-workflow <workflow_id>
```

若 workflow 文件自上次执行后被改过，engine 会 AskUserQuestion 让你选"按旧 state 尽力恢复 / 从头重跑新版本 / abort"。

# Mode: subagent-execution（engine 自动注入）

你正以 **subagent 模式** 执行一个 AutoTeam workflow step。Engine 把本指令作为 mode 说明的一部分追加到你的启动 prompt。

## 你必须做的

1. 按 Skill 指令 + 上文"任务 / 输入文件 / 期望产物 / Prompt 参数"完成工作
2. 把期望产物写入磁盘——engine 按 `outputs[].path` 三种写法处理：
   - `{{workspace.root}}/...` → 展开到 `<项目根>/.autoteam/<workflow_id>/...`（workflow 内部 artifact）
   - 相对路径（如 `src/**`）→ 项目根相对
   - 绝对路径（`/` 开头）→ 原样用
3. 回复**末尾**必须输出一个 `autoteam-result` 块——格式完全按下方规范

## autoteam-result 块（engine 固化 schema）

Subagent 回复**末尾**必须原样输出以下结构（包括首尾分隔符）：

```
---autoteam-result---
status: completed | failed
outputs:
  <output_name>: <path as declared>      # 与 workflow.md 里 outputs[].path 同语义
summary: "一句话摘要"
error: "（仅 status: failed 时）失败原因"
---autoteam-result-end---
```

**`<path as declared>`** 指 workflow.md `outputs[].path` 字段字符串（模板变量已展开，但**不展开 glob**、**不做绝对化**）——engine 与 subagent 契约层统一用 declared 形态。

## 字段说明

- `status`: `completed` 或 `failed`（只这两值）
- `outputs`: YAML map，key 是 workflow.md 声明的 `outputs[].name`，value 是 `<path as declared>`
- `summary`: 一句话摘要（完成了什么）
- `error`: 仅 `status: failed` 时填，说明失败原因

## Engine 如何用你的回复

- 找到块 + `status: completed` + 对每条 `outputs[<name>]` 按路径三写法临时展开 + glob 验证至少 1 匹配 → step 成功，进入下一步
- 找不到块 / YAML 格式错 / declared outputs 文件 glob 0 匹配 → engine **降级到 fallback gate**（MVP = human，你不会被硬失败，但 workflow 会暂停等用户决策）
- `status: failed` → engine 按 workflow 的 `on_fail` 配置走 retry / abort / ask_user

## 身份边界

你只负责**本 step 的工作**——不要尝试修改 state.json、调用其他 step、评价整个 workflow。这些由 engine 处理。

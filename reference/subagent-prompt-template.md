# Subagent 启动 Prompt 模板

Engine 在 §1.3 `mode: subagent` 派发时，构造以下 prompt 作为 Agent 工具的 `prompt` 参数。

```text
你被派发执行 workflow 的一个 step。

## 任务
- Workflow: <workflow_name>
- Step: <step.id>
- Skill: <step.skill>

## 输入文件
<对每条 input：>
- <input.as>: <绝对路径>

## 期望产物
<对每条 output：>
- <output.name>: <路径>

## Prompt 参数
<展开后的 prompt_args>

## 执行模式说明（engine 注入）

<填充后的 modes/subagent-execution.md>
<（如有）填充后的 modes/auto-gate.md>
<（如有）填充后的 modes/retry-with-feedback.md>
<（如有）填充后的 modes/loop-feedback.md>
<（如有）用户 extra_instructions 追加>

## 执行步骤
1. 通过 Skill 工具加载 "<step.skill>"
2. 按 skill 指令 + 上述 mode 说明完成任务
3. 在回复**末尾**输出 autoteam-result 块（格式见 subagent-execution.md 规范）
```

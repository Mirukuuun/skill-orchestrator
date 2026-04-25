# autoteam-result 结果块协议

本协议是 **subagent ↔ engine 之间的契约**。Schema 所有权在 engine（本 skill），不放在 skill 产出格式里（C1：engine 不依赖用户 skill）。

Subagent 回复**末尾**必须输出 `autoteam-result` 块（以下 schema 固化在本 skill；mode md `modes/subagent-execution.md` 引用本规范，不反向持有）：

```
---autoteam-result---
status: completed | failed
outputs:
  <output_name>: <path as declared>      # 与 workflow.md 里 outputs[].path 同语义
summary: "一句话摘要"
error: "（仅 status: failed 时）失败原因"
---autoteam-result-end---
```

**`<path as declared>`** 指 workflow.md `outputs[].path` 字段字符串（模板变量已展开，但**不展开 glob**、**不做绝对化**）——engine 与 subagent 契约层统一用 declared 形态；绝对化只在 engine 内部需要时临时做（如完成度 glob / dirty 清理）。

**Engine 处理**：
- 找不到块 / YAML 格式错 → **fallback to `fallback_gate`**
- `status: completed`:
  - 对每条 `outputs[<name>]` 按路径三写法临时展开 + glob 验证（至少 1 匹配）；不存在 → **fallback**
  - 记录 `<path as declared>` 到 `state.step_results[step.id].outputs[<name>]`（不存绝对化结果）
  - 进入 §1.4
- `status: failed` → 进 §1.6 `on_fail`

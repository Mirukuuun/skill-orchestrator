# Mode: auto-gate（engine 自动注入）

本 step 配置了 **auto gate**——engine 在你完成后会读取一个**判决字段**决定 workflow 下一步：前进 / 回跳 / 终止。你必须让产出满足判决字段的要求。

## 必须满足

- 你的产出文件中必须包含 `decision.field` 声明的**根级字段**
- 字段位置按文件类型：
  - Markdown → 写在 **frontmatter YAML** 的根级
  - JSON / YAML → 根级
- 字段值必须是 `decision.values` 表里的**某一个枚举值**（engine 看到未知值会降级到 human gate）

## 判决字段常见值（workflow 会在 `decision.values` 里明确）

- `continue` → workflow 前进到下个 step
- `goto:<step_id>` → engine 回跳到某已完成 step；engine 会自动把**你本次的产出文件路径**作为 `decision_artifact` 带给 target step 的下一轮执行者（触发 loop-feedback.md 注入）
- `abort` → 终止 workflow

## 约束

- 判决字段是 **case-sensitive 根级字段**——不支持嵌套路径（如 `status.verdict`）
- 字段值必须完全匹配 `decision.values` 里列出的字符串（engine 严格相等比较）
- 如果你判断结果介于两个值之间 / 无法裁决，**选保守值**（通常是 `goto` 或 `abort` 而非 `continue`）——engine 总能通过 goto/fallback 继续运转，但错误的 `continue` 会把坏产物传下去

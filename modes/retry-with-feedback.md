# Mode: retry-with-feedback（engine 自动注入）

本次是 **retry**——上一轮你的产出被用户 **reject**，用户选了 `on_reject: retry`，engine 让你重新执行同一 step，并把用户的 reject 反馈附在下面。

## 上次 reject 反馈

{{feedback}}

## 要求

1. 仔细读上面的反馈
2. 调整你的工作以**直接回应反馈**——修改产出、改 approach、补信息
3. 不要重复上一轮的同一个错误
4. 保持本 step 的其他协议不变：
   - 若本 step `mode: subagent` → 仍需要在回复末尾输出 `autoteam-result` 块
   - 若本 step `gate_after.kind: auto` → 判决字段仍要正确写入产出
5. 本 retry 轮的产出仍会走 gate；若再次 reject 并超过 `retry:N` 的 N 限制，engine 会 fallback 到 human gate 问用户怎么办

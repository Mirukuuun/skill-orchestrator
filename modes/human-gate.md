# Mode: human-gate（engine 可选注入 / 参考模板）

本 step 配置了 **human gate**——你的工作完成后，engine 会暂停 workflow 让用户选 `approve` / `reject` / `abort`。本 mode md 不会被 engine 自动注入（human gate 的暂停动作由 engine 自己完成）；它作为 **`gate_after.mode_md` 或 `extra_instructions` 引用时的参考模板** 存在。

## 建议做法

- 完成 step 核心工作后，**明确摘要**你产出了什么、为什么它可以通过审核——让用户能快速决策
- **不要自我预判通过**（别在产出里写"已通过审核"之类）
- 对不确定的点，在 summary 里明确列出，方便用户有针对性审阅

## 用户反应后 engine 如何处理

- **approve** → workflow 前进到下个 step
- **reject** → engine 根据 `on_reject` 配置分叉（两字段互斥，切换时原子清对方）：
  - `on_reject: retry` → engine **原子操作**：写 feedback 到 `state.pending_feedback` + 清 `pending_feedback_gate = null`；下轮本 step 会收到 `retry-with-feedback.md` 注入（带 `{{feedback}}`）
  - `on_reject: goto:<step_id>` → engine 构造 `goto_context = {from_step, gate_kind: "human", decision_artifact: <feedback>}` 传入 Backward goto；cap preflight 通过后 Backward goto 原子 commit `pending_feedback_gate = goto_context` + 清 `pending_feedback = null`；target step 下轮会收到 `loop-feedback.md` 注入（带 `{{decision_artifact}}`）。超 cap 则 fallback 到 human gate（state 保持 pre-goto 完整态，附 `fallback_context`）
- **abort** → workflow 终止

## 身份边界

human gate 的 UI 交互由 engine 处理——你不需要输出"请审核"之类的对话文字，也不需要自己调 AskUserQuestion。专心产出好的工作 + 清晰的 summary。

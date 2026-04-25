# Mode: loop-feedback（engine 自动注入）

本次是 **goto 回跳后的重新执行**——某下游 gate（auto 或 human）作出 `goto:<本 step>` 的决定，engine 回到本 step 重跑。下游 gate 给出的**决策产物**作为参考资料注入给你。

## 上次 gate decision artifact

{{decision_artifact}}

以上是**触发 goto 的 gate 给出的参考资料**——可能是：
- **文件路径**（auto gate 场景：指向评审 / 检查产出的 markdown / yaml / json）
- **文本**（human gate reject-goto 场景：用户的反馈 + goto 意图说明）

## 要求

1. **读懂** artifact 内容——如果是路径，读文件；如果是文本，直接理解
2. 据此调整本 step 的产出——gate 为什么要 goto 回本 step？哪些点要改？哪些前提假设错了？
3. 保持本 step 的其他协议不变（subagent 末尾 `autoteam-result` 块、auto gate 判决字段等）
4. 如果**完全不理解 artifact**（格式异常 / 内容空 / 无法读取），在产出 summary 里明确标注——engine 会感知到 downstream gate 的困惑并降级到 human gate

## Engine 自动行为（你不需要关心）

- engine 已在 §1.1 把 `state.pending_feedback_gate.decision_artifact` 复制到局部 `LOOP_ARTIFACT` 并传入本 mode md；state 字段本身**不清空**（本 step §1.7 成功通过 gate 后才由 engine 清掉，保证 crash 恢复不丢 feedback）
- 循环次数由 engine 用 `state.loop_counts[<target_step_id>]` 跟踪；超 `max_goto_loops`（默认 10）会 fallback 到 human gate
- `pending_feedback` 和 `pending_feedback_gate` 两字段**互斥**——你不会同时收到 retry-with-feedback.md 和 loop-feedback.md

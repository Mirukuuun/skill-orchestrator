# 失败模式汇总

| 场景 | 处理 |
|---|---|
| **结构性错误**（lint / 0.2 任一失败 / unresolved template vars / unknown skill / invalid mode · gate_after.kind · fallback_gate / 嵌套 decision.field / goto target 不存在） | abort + 明确定位 notify（不走 fallback） |
| Workflow hash 变化（resume） | AskUserQuestion |
| Dirty step（resume 时发现残留） | AskUserQuestion |
| Mode md 读取失败 | notify + fallback to human |
| Subagent 无 autoteam-result / 格式错 | fallback to human |
| Subagent status: failed | §1.6 on_fail |
| Auto gate 字段读失败 / 值不在表 | fallback to human |
| Main 模式 outputs 文件未出现 | fallback to human |
| Loop 超 `max_goto_loops` | fallback to human |
| State.json 损坏 | AskUserQuestion（备份 / 从头 / abort） |
| 用户 Ctrl+C | 标记 paused，下次可 resume |

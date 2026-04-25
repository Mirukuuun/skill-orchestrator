# 执行协议

本文档定义 engine 从触发到结束的完整执行流程：mode md 注入机制 + §0 预检查 → §1.1~§1.7 step 循环 → §2 完成 → §3 恢复模式。engine 每次启动/恢复 workflow 时必须 Read 本文档以遵循正确的状态机。

---

## 核心机制：运行时 mode md 注入

Engine 实现 "skill 零感知" 的关键路径。

### 为什么要注入

workflow 的**执行模式约束**（subagent 必须回报 `autoteam-result` 块、auto gate 要求产出满足字段约定、retry 要带 feedback 等）**不应该要求 skill 本身知道**——skill 可能来自 superpowers / awesome-claude-code / 第三方。

解决：本 skill 内置一批标准"模式说明 md"（`modes/*.md`），engine 在每个节点执行前 Read 并注入执行者上下文。skill 本身完全不用改。

### 内置 mode md 清单

```
~/.claude/skills/skill-orchestrator/modes/
├── subagent-execution.md        # subagent 的基础执行契约（含 autoteam-result 块要求）
├── human-gate.md                # 主 agent 完成后停下问用户
├── auto-gate.md                 # 执行者需要在产出里包含判决字段
├── retry-with-feedback.md       # 本次是 retry，上次 reject 原因 {{feedback}}
└── loop-feedback.md             # 本次是 goto 回跳，上次 gate decision artifact {{decision_artifact}}
```

每份 mode md 是参数化模板（`{{variable}}` 占位符）。

### 注入路径

| 条件 | 要注入 | 注入方式 |
|---|---|---|
| `mode: subagent` | `subagent-execution.md` | engine 构造 Agent prompt 时拼入 |
| `gate_after.kind: auto` | `auto-gate.md` | 注入给执行者（让产出满足判断） |
| `retry_counts[step.id] > 0` 且有 feedback | `retry-with-feedback.md` | 追加 |
| `pending_feedback_gate` 非空（goto 回跳后第一次执行 target）| `loop-feedback.md` | 追加（C4 自动机制） |
| `gate_after.mode_md` 指定 | 用指定路径**覆盖**对应默认 | — |
| `gate_after.extra_instructions` | **追加**到 mode md 末尾 | — |

### Mode md 是数据不是代码

加新模式 → 加一份 md；改模式行为 → 改 md；engine 代码不动。

---

## 执行协议

### 0. 预检查（workflow 启动时一次）

#### 0.1 参数收集

- 从触发语句 / `/orchestrate` 参数提取 `params` 声明的所有必填参数
- 缺失必填 → AskUserQuestion

#### 0.2 结构校验 + YAML parse

1. **跑 `lint <workflow_path>`**（见 reference/lint.md）——覆盖结构层 + 绝大多数语义层（skill 存在性 / mode 值 / goto 目标 / 模板变量引用等）；失败 → abort + 原样 notify stderr
2. **YAML parse 每个 step 的 yaml block**（LLM，lint 不做这件事）——失败 → abort + 指出哪个 step 哪一行
3. 解析 steps（按"Engine 解析规则"，见 reference/workflow-schema.md）+ 展开模板变量得到执行时值
4. 任何错误 → abort + 明确定位 + notify user

#### 0.3 Workspace 初始化

- 生成 `workflow_id` = `NNN-<shortid>`（扫 `<项目根>/.autoteam/` 取现有最大 NNN + 1；shortid 用 `uuidgen | head -c 8`）
- 创建 `<项目根>/.autoteam/<workflow_id>/` 目录
- 计算 workflow 文件 sha256 → `state.workflow_hash`（M4：防 resume 时 workflow 被改过导致错配）
- 原子写入初始 `state.json`（见 reference/state-schema.md）

### 1. 按 steps 顺序执行

对每个 step（按 body 中 H3 顺序）：

#### 1.1 准备

1. `state.current_step = step.id`；`state.dirty_step = step.id`（进入 §1.3 前置位，M5：防崩溃后 resume 时不知残留）；`status = running`
2. 解析 `inputs`：从 `state.step_results[<from_step>].outputs[<name>]` 取 path
3. 展开 `prompt_args` 里的模板变量
4. **Loop feedback 取用（N3 两字段互斥）**：若 `state.pending_feedback_gate` 非空 → 把它的 `decision_artifact` 复制到局部 `LOOP_ARTIFACT` 供 §1.2 使用。**不清空 state.pending_feedback_gate**（清空时机推迟到 §1.7）——这样若本 step 在 §1.3/§1.4 之间崩溃，resume 时 pending_feedback_gate 仍在，能重新注入 feedback；step 成功后再 §1.7 清
5. 原子写入 state.json

#### 1.2 Mode 说明准备（注入前置）

按"Mode md 注入路径"表决定要用的 mode md 清单：

```
inject_contents = []
if mode == "subagent":
    inject_contents.append(Read("modes/subagent-execution.md") 填参数)
if gate_after.kind == "auto":
    inject_contents.append(Read("modes/auto-gate.md") 填参数)
if retry_counts[step.id] > 0 and state.pending_feedback:
    inject_contents.append(Read("modes/retry-with-feedback.md") 填 {{feedback}} = state.pending_feedback)
if LOOP_ARTIFACT is not None:    # §1.1 从 pending_feedback_gate 拿的
    inject_contents.append(Read("modes/loop-feedback.md") 填 {{decision_artifact}} = LOOP_ARTIFACT)
if gate_after.mode_md:
    替换对应默认
if gate_after.extra_instructions:
    inject_contents[-1] += "\n\n" + gate_after.extra_instructions
```

**统一模板变量**：`modes/loop-feedback.md` 内部占位符是 `{{decision_artifact}}`（不使用其它别名），engine 传参一致。

mode md 读取失败 → `fallback_gate`。

**两字段互斥（N3）**：`state.pending_feedback` 走 `retry-with-feedback.md` 路径、`state.pending_feedback_gate` 走 `loop-feedback.md` 路径；同一时刻只该有一个为非空（见 §1.4 human gate reject 如何分叉写入）。

#### 1.3 按 mode 执行

##### `mode: main` — 主 agent 模式（C3 对话文本注入参数）

engine 在主会话：

1. 对每份 `inject_contents`：用 **Read 工具**读入主会话（mode md 独立可审计，行为变更改 md）

2. **输出一段对话文本声明执行契约**（关键：不通过 Skill args 传结构化参数）：

   ```
   现在我要执行 workflow step "<step.id>"。
   
   Skill: <step.skill>
   输入文件：
   - <input.as>: <绝对路径>
   期望产物：
   - <output.name>: <绝对路径>
   Prompt 参数：
   - feature: <展开后的值>
   - language: <展开后的值>
   
   按上面注入的 mode md 要求完成任务。
   ```

3. 通过 Skill 工具加载 step.skill：`Skill(skill="<step.skill>", args="")` — args 留空或极简，**不承载结构化数据**

4. skill 加载后主 agent 在对话上下文 + 注入 mode 说明下工作——skill 自然从上下文读参数，**不需要约定特殊序列化格式**

5. skill 完成后：
   - 切回编排者身份
   - 核验 `declared outputs` 文件存在（glob 至少匹配 1 个；否则 §1.5 fallback）
   - 进入 §1.4

##### `mode: subagent` — 子 agent 模式

构造 subagent 启动 prompt（模板见 reference/subagent-prompt-template.md），派发：

```
Agent(
  subagent_type="general-purpose",
  prompt=<构造的 prompt>,
  run_in_background=false         # MVP 纯串行
)
```

等 subagent 返回，解析 **autoteam-result 块**（schema + engine 处理规则见 reference/autoteam-result-schema.md）：
- 找不到块 / YAML 格式错 → §1.5 fallback
- `status: completed` → 按 schema 记录 outputs 到 `state.step_results[step.id].outputs[<name>]`，进 §1.4
- `status: failed` → 进 §1.6 `on_fail`

#### 1.4 Gate 处理

##### `gate_after.kind: human`

options 固定 `[approve, reject, abort]`：

- **approve** → §1.7
- **reject**（N3 按 `on_reject` 分叉写 feedback）：
  1. 收集 feedback 文本（AskUserQuestion 后续展开 text input）
  2. 按 `on_reject` 分叉：
     - `retry`：**原子操作**——写 feedback 到 `state.pending_feedback` + 清 `state.pending_feedback_gate = null`（N3 两字段互斥——进入 retry 路径必清旧的 loop feedback state，否则下轮 §1.2 会同时注入 retry-with-feedback.md + loop-feedback.md）；`retry_counts[step.id]++`；≤ `max_retries` 回 §1.1 重跑本 step；超限 → AskUserQuestion `[retry / skip / abort]`：
       - `retry` → 保留 `pending_feedback`，回 §1.1
       - `skip` → 视为通过本 step，进 §1.7（§1.7 会清 `pending_feedback`）
       - `abort` → `status = aborted`，退出
     - `goto:<step_id>`：构造 `goto_context = {from_step: step.id, gate_kind: "human", decision_artifact: <feedback>}`；然后进入 **Backward goto(target, goto_context)**（见下）。Backward goto 会在 **cap 通过时** 提交 `goto_context` 到 `state.pending_feedback_gate`（走 `loop-feedback.md` 路径）；**不写 `pending_feedback`**（两字段互斥）
- **abort** → `status = aborted`，退出

##### `gate_after.kind: auto`

1. 读 `decision.output_file`：不存在 → **fallback**
2. 解析 `decision.field`（MVP 根级 + case-sensitive）：
   - Markdown → frontmatter 根级字段
   - YAML/JSON → 根级字段
   - 读不到 / 格式错 → **fallback**
3. 值查 `decision.values` 表：
   - `continue` → §1.7
   - `goto:<step_id>`：构造 `goto_context = {from_step: step.id, gate_kind: "auto", decision_artifact: <decision.output_file 的解析后路径>}`；然后进入 **Backward goto(target, goto_context)**
   - `abort` → `status = aborted`
   - 值不在表 → **fallback**

##### Backward goto（C5 state rewind + N4 统一 loop cap）

**前置**：调用方（§1.4 human reject goto / §1.4 auto goto / §1.5 fallback goto）构造 `goto_context = {from_step, gate_kind, decision_artifact}` 作为参数传入 **Backward goto**。Backward goto 只有在 **cap 通过并准备实际 rewind 时**，才把 `goto_context` 提交到 `state.pending_feedback_gate`——确保超 cap fallback 不会留下 orphan feedback state、且 pre-goto state 保持完整态供用户决策（参见本段末"设计说明"）。

```
current = state.current_step        # 触发 goto 的那个 step
target  = <goto target>
# target 必须在当前 generation 的 completed_steps 中（否则是 §0.2 应抓的结构性错误遗漏，abort）

max_cap = workflow.max_goto_loops (默认 10)      # N4: 顶层字段，所有 goto 分支共用
new_generation = state.loop_counts.get(target, 0) + 1

# 1. Cap preflight — 不改任何 state
if new_generation > max_cap:
    → fallback to human gate(
        reason = "Loop 超 max_goto_loops",
        fallback_context = goto_context      # 传 decision_artifact 给 fallback prompt 作局部参考
      )
    # 保持 pre-goto state:
    # - 不 stash history / 不 truncate completed_steps / 不推进 loop_counts
    # - 不 commit goto_context 到 pending_feedback_gate
    return

# 2. Commit loop feedback context + 清 retry feedback（原子操作，cap 通过，后续 state mutation 视为同一事务）
state.pending_feedback_gate = goto_context
state.pending_feedback = null      # N3 两字段互斥：进入 goto 路径必清旧的 retry feedback state（覆盖 retry→fallback→goto 切换路径）

# 3. Stash 旧 step_results 到 history（保留历史以便审计/未来 UI 查看）
for sid in [target, ..., current]:
    if sid in state.step_results:
        state.history.setdefault(sid, []).append({
            "generation": new_generation - 1,
            "result": state.step_results[sid]
        })
        del state.step_results[sid]

# 4. Truncate completed_steps
idx = state.completed_steps.index(target)
state.completed_steps = state.completed_steps[:idx]

# 5. Advance loop counter
state.loop_counts[target] = new_generation

# 6. Reset current / retry / dirty
state.current_step = target
state.retry_counts[target] = 0
state.dirty_step = null

# 7. 原子写入 state.json，回到 §1.1 target step
```

**设计说明（为什么 cap preflight）**：若 cap 检查放在 stash/truncate/loop_counts++ **之后**，超 cap 时 state 会部分 rewind 但 `current_step`/`dirty_step` 未重置——用户在 fallback human gate 看到的是矛盾中间态。改成 **cap preflight + 延迟提交 `pending_feedback_gate`** 后，超 cap 时 state 保持完整 pre-goto 态，goto_context 只作为局部 `fallback_context` 传给 fallback prompt。这让 loop cap 分支与其他 fallback 分支（subagent 无块 / auto gate 字段读不到 / mode md 读不到等本来就不改 state）保持一致。

**C4 Loop feedback**：
`state.pending_feedback_gate` 非空时，§1.1 把 `decision_artifact` 复制到局部 `LOOP_ARTIFACT`（**不动 state**），§1.2 自动追加 `modes/loop-feedback.md` 到 inject_contents（`{{decision_artifact}}` 填成 `LOOP_ARTIFACT`）。**workflow 作者不需要在 target step 声明 loop inputs**——engine 自动带过来。target step **成功通过 gate** 后由 §1.7 清空 `pending_feedback_gate`（保证 crash/resume 不丢 feedback）。

#### 1.5 Fallback 原则（MVP fallback_gate = human）

当执行时异常发生，**降级到 `fallback_gate`**（MVP 只允许 `human`）：

异常清单：
- Subagent 回复无 `autoteam-result` 块 / 格式错
- Auto gate 的 `decision.output_file` 不存在
- Auto gate 的 `decision.field` 读不到 / 值不在表
- Main 模式 skill 跑完后 `declared outputs` 文件未出现（glob 0 匹配）
- Mode md 读取失败
- Loop 超 `max_goto_loops`

Fallback 流程（human gate）：

```
1. Notify: "Step <id> 异常：<原因>，降级到 human gate"
   - 若调用方传入 `fallback_context`（例：Loop 超 max_goto_loops 分支），在 Notify + AskUserQuestion 文本里附 `fallback_context.decision_artifact`（让用户看到触发 goto 的 gate 决策产物，避免 cap 超限 fallback 时上下文丢失）
2. AskUserQuestion:
   options: [
     "继续（当作成功）",
     "重跑当前 step",
     "goto:<step_id>",   # 列出 completed_steps
     "abort"
   ]
3. 按选择走对应路径（4 个 options 的 state 语义明确如下）
   - **"继续（当作成功）"** → 视为本 step 成功，进 §1.7（§1.7 会清**两个 feedback 字段**）
   - **"重跑当前 step"** → 回 §1.1 重跑本 step；`pending_feedback` / `pending_feedback_gate` **保留原值不动**（若本 step 是 goto target，pending_feedback_gate 仍有效，§1.1 继续取用 LOOP_ARTIFACT 注入 loop-feedback.md；若是 retry 的继续，pending_feedback 保留生效）——语义等同用户主动要求再试一次，无 counter 变化
   - **"goto:<step_id>"** → 构造 `goto_context = {from_step: <当前 step.id>, gate_kind: "human", decision_artifact: "本次 fallback 的触发信息 + 用户备注"}`，再进 **Backward goto(target, goto_context)**（Backward goto 会原子 commit pending_feedback_gate + 清 pending_feedback，覆盖 retry→fallback→goto 切换路径）
   - **"abort"** → `status = aborted`，退出
```

**保证**：auto gate 永不 hard fail；LLM 产出不稳定永远不让 workflow 卡死；skill 零感知。

#### 1.6 on_fail 处理

| `on_fail` | 动作 |
|---|---|
| `abort` | `status=failed`，退出 |
| `retry:N` | `retry_counts[step.id]++`；≤ N 回 §1.1；超 N → fallback to human |
| `ask_user` | AskUserQuestion `[retry/skip/abort]` |

#### 1.7 成功完成本 step

- `completed_steps.append(step.id)`
- `current_step = null`；`dirty_step = null`
- `retry_counts[step.id] = 0`（本 step 通过 gate，清 retry）
- **loop_counts 不在此清**（按 target step 索引，workflow 结束时统一清）
- **`pending_feedback_gate = null`**（清空时机从 §1.1 推迟到此，保证 crash 恢复不丢 feedback——若本 step 是 goto 回跳的 target，说明本次 loop feedback 已成功消费）
- **`pending_feedback = null`**（retry feedback 也在本 step 成功通过 gate 后清，保证 N3 两字段互斥不被破坏——若本 step 是 retry 的延续，说明本次 retry feedback 已成功消费；`max_retries` 超限后用户选 skip/continue 同样在此清）
- 原子写入 state.json

### 2. 全部 steps 完成

- `status = completed`；`completed_at` 填充
- 清空 `loop_counts`
- Notify user 完成摘要

### 3. 恢复模式

用户输入 `resume <workflow_id>` 或 `resume <name>`：

1. 定位 state.json（按 id 精确匹配；按 name 列出所有 `status in {running, paused}` 的让用户选）

2. **Workflow hash 对比**（M4）：

   ```
   current_hash = sha256(~/.claude/workflows/<state.workflow_skill>.md)
   if current_hash != state.workflow_hash:
       AskUserQuestion(
         "Workflow 配置自上次执行后已变化。如何处理？",
         options=[
           "按旧 state 尽力恢复（可能跑错）",
           "从头重跑新版本",
           "abort"
         ]
       )
   ```

3. **Dirty step 检查**（M5）：

   ```
   if state.dirty_step is not None:
       AskUserQuestion(
         "上次在执行 step <dirty_step>，可能残留不完整产物。如何处理？",
         options=[
           "清理该 step 声明的 outputs 后重跑",
           "保留残留直接重跑（可能混合新旧文件）",
           "abort"
         ]
       )
   ```

4. 从 `current_step`（非空）或 `completed_steps` 之后的下一 step 重跑

**粒度**：step 级（不尝试 step 内恢复）。

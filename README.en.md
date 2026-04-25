[中文](README.md) · **English**

# Skill Orchestrator

**A declarative orchestration engine for Claude Code skills — chain multiple skills into a workflow with one markdown file. The orchestrator runs it, recovers from crashes, and degrades gracefully on failure.**

In one line: **Skill is "how to do one thing", Workflow is "how to chain many things", Orchestrator is "how to run the chain".**

---

## Why this exists

Claude Code [Skills](https://docs.claude.com/en/docs/claude-code/skills) package one type of task into a reusable unit (`~/.claude/skills/<name>/SKILL.md`): writing design docs, running code review, generating tests, and so on. Each skill does one thing well.

But real workflows are typically **combinations of multiple skills**:

```
write design doc → develop from design → run code review → if review fails, loop back to development
```

Hand-wiring this chain causes problems:

- Missed steps / wrong order / context cross-contamination
- No cross-session recovery — Ctrl+C and lose all progress
- A failed review requires manual copy-paste of the report into the next dev prompt
- No safety net when something breaks

**Skill Orchestrator is the runtime for that chain.** You declare in a markdown file which skills to chain, who depends on whom, where humans need to approve, and where to loop. The orchestrator handles:

- Sequential step dispatch (main session or subagent)
- Gate checkpoints — wait for human approval, or auto-decide based on a verdict field
- Auto-injection of feedback artifacts when looping back
- Atomic state writes to `state.json` — resume anytime
- All runtime exceptions degrade to the fallback gate — **never hard-fail**
- Structural errors (broken workflow file) abort with a precise error location

## 30-second tour

A workflow file looks like this (`~/.claude/workflows/coding.md`):

````markdown
---
name: coding
description: design → code → review; auto-loop back on failed review
autoteam_version: "0.1"
params:
  feature:
    type: string
    required: true
fallback_gate: human
max_goto_loops: 3
---

# Coding Workflow

## Steps

### design

```yaml
skill: design-gen
mode: main
prompt_args:
  feature: "{{params.feature}}"
outputs:
  - name: design_doc
    path: "{{workspace.root}}/artifacts/design.md"
on_fail: ask_user
gate_after:
  kind: human
  prompt: "Design OK?"
  on_reject: retry
  max_retries: 3
```

### code

```yaml
skill: code-dev
mode: subagent
inputs:
  - from: design.outputs.design_doc
    as: design_doc
outputs:
  - name: code_changes
    path: "src/**"
on_fail: retry:2
```

### review

```yaml
skill: code-review
mode: subagent
inputs:
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
````

Run it:

```
/orchestrate coding feature="JWT auth"
```

What the orchestrator does:

1. Loads `design-gen` in your main session, writes the design doc, then pauses for review (human gate)
2. Once you approve, dispatches a clean-context subagent with `code-dev` to implement against the design
3. Dispatches another subagent with `code-review` to audit; `verdict: pass` in the review markdown ends the run, `verdict: fail` automatically loops back to the `code` step with the review report injected as feedback, up to 3 iterations
4. Ctrl+C anytime; `resume coding` next session picks up from the breakpoint

## Core capabilities

### Two execution modes

| Mode | When | What |
|---|---|---|
| `main` | Steps that need interaction with you (design review, requirement clarification) | Skill runs in your current main session — preserves discussion continuity |
| `subagent` | Mechanical steps (code generation, code review, test generation) | Dispatches a clean-context subagent — isolates tokens, prevents context pollution |

### Two gate types

| Gate | Trigger | Behavior |
|---|---|---|
| `human` | Step completes; orchestrator pauses for you | Choose `approve` / `reject` / `abort`. Reject can carry feedback and pick `retry` (re-run the same step with feedback injected) or `goto:<step>` (loop back to an upstream step) |
| `auto` | Orchestrator reads a root-level decision field from the output file | Field value matched against the `values` table: `continue` advances / `goto:<step>` loops / `abort` terminates |

### Evaluator–Optimizer feedback loop

Whenever any gate decides `goto:<step>`, the orchestrator automatically injects the **artifact that triggered the gate** (review report path / human feedback text) as `{{decision_artifact}}` into the target step's next iteration — workflow authors **do not need** to declare this feedback in the target step's `inputs`.

`max_goto_loops` (default 10) is the global ceiling across all goto loops; exceeding it falls back to a human gate, **without hard-failing**.

### State persistence & crash recovery

- Every state change is atomically written to `<project_root>/.autoteam/<workflow_id>/state.json`
- `resume <name>` or `resume <workflow_id>` picks up from the breakpoint
- If the workflow file changed since last execution, the orchestrator detects the hash mismatch and prompts: "continue with old state / restart with new version / abort"

### Fallback guarantee

| Exception | Handling |
|---|---|
| Subagent reply missing the `autoteam-result` block / malformed | → fall back to human gate |
| Auto gate field unreadable / value not in the `values` table | → fall back to human gate |
| Declared output file did not appear (glob 0 matches) | → fall back to human gate |
| Loop exceeds `max_goto_loops` | → fall back to human gate |
| Workflow file structural error (missing field / nonexistent goto target / uninstalled skill, etc.) | abort + precise error (does **not** route to fallback — you need to fix the file) |

**Runtime exceptions never hard-fail; structural errors abort immediately.** Two clean lanes.

## Install

```bash
git clone https://github.com/Mirukuuun/skill-orchestrator.git ~/.claude/skills/skill-orchestrator
```

Then drop your workflows into `~/.claude/workflows/`. Claude Code picks them up next session.

The `/orchestrate` slash command lives in `~/.claude/commands/orchestrate.md` — a reference copy is in this repo; copy it into place.

## Trigger

| Input | Behavior |
|---|---|
| `/orchestrate` | List all workflows in `~/.claude/workflows/` |
| `/orchestrate <name>` | Run the named workflow; prompts you for any missing required params |
| `/orchestrate <name> k1=v1 k2=v2` | Run with arguments |
| "run xxx workflow" / "execute the xxx pipeline" (natural language) | Fuzzy-match by name / description; ask to disambiguate on multi-match |
| `resume <workflow_id>` / `resume <name>` | Pick up from `state.json` |

## Workflow schema cheat sheet

### Frontmatter

| Field | Required | Description |
|---|---|---|
| `name` | ✓ | Must equal the filename without `.md` (`coding.md` → `name: coding`) |
| `description` | ✓ | One-line summary |
| `autoteam_version` | ✓ | Schema version, currently `"0.1"` |
| `params` | | Workflow parameter declarations (`type` / `required` / `default`) |
| `workspace` | | `root` / `state_file`, supports `{{workflow_id}}` |
| `fallback_gate` | | Currently only `human` is supported |
| `max_goto_loops` | | Global goto loop ceiling (default `10`) |

### Step fields

| Field | Required | Description |
|---|---|---|
| `skill` | ✓ | The corresponding `~/.claude/skills/<name>/SKILL.md` must exist |
| `mode` | ✓ | `main` / `subagent` |
| `on_fail` | ✓ | `abort` / `ask_user` / `retry:<positive int>` |
| `inputs` | | `[{from: <step_id>.outputs.<name>, as: <local_name>}]` |
| `outputs` | | `[{name, path}]`; path has three forms (see below) |
| `prompt_args` | | Map passed to the executor (supports template variables) |
| `gate_after.kind` | | `human` / `auto` |
| `gate_after.on_reject` | | Human gate: `retry` / `goto:<step>` |
| `gate_after.decision` | | Required for auto gate: `{output_file, field, values}` |
| `gate_after.prompt` | | Human gate prompt text shown to you |

### Output path: three forms

| Form | Meaning | Resolves to |
|---|---|---|
| `{{workspace.root}}/...` | Internal workflow artifact | `<project_root>/.autoteam/<workflow_id>/...` |
| `src/**` (relative) | Project file change | `<project_root>/...` |
| `/...` (absolute) | Escape hatch | Used as-is |

Supports single files / directories / globs.

### Template variable namespace

| Form | Purpose |
|---|---|
| `{{params.<name>}}` | Pull from frontmatter `params` |
| `{{workspace.<root\|state_file\|workflow_id>}}` | Pull from workspace config |
| `{{outputs.<output_name>.path}}` | Only the current step's own outputs (cross-step references must use `inputs.from`) |

## Repo layout

```
skill-orchestrator/
├── SKILL.md                          Entry point + navigation index
├── reference/                        Protocol & schema details
│   ├── workflow-schema.md            Workflow file structure
│   ├── execution-protocol.md         Execution protocol (§0 preflight / §1 step loop / §2 closing / §3 resume)
│   ├── state-schema.md               state.json structure
│   ├── subagent-prompt-template.md   Subagent boot prompt template
│   ├── autoteam-result-schema.md     Subagent ↔ engine result-block contract
│   ├── lint.md                       Lint script docs
│   ├── failure-modes.md              Failure-mode summary
│   ├── execution-sequence.md         Execution sequence diagrams
│   └── workflow-example.md           Complete workflow example
├── modes/                            Mode mds injected by the engine at runtime (data layer)
│   ├── subagent-execution.md
│   ├── human-gate.md
│   ├── auto-gate.md
│   ├── retry-with-feedback.md
│   └── loop-feedback.md
└── bin/
    └── lint                          Workflow validator (zero-dep bash)
```

## Design principles

1. **Workflow is data, not code.** All orchestration logic lives in the orchestrator; users only write declarative markdown.
2. **The orchestrator does not depend on user-skill implementations.** The protocol lives only in the orchestrator and the mode mds; skill arguments flow as conversation text, subagent results follow the fixed `autoteam-result` schema. Adding a new skill never requires changing the orchestrator.
3. **Mode md is data, not code.** To add or change an execution mode, edit the md — the orchestrator stays untouched.
4. **Never hard-fail.** All runtime exceptions degrade to `fallback_gate` (human).
5. **Structural error ≠ runtime error.** A broken workflow file aborts immediately so you can fix it; it never enters the fallback path.
6. **Every state change is persisted.** Ctrl+C anytime; `resume` picks up.
7. **Path contract uses declared form.** State files and subagent results both store the `<path as declared>` string (template variables expanded, but globs not expanded and relative paths not absolutized) — only the engine internally absolutizes temporarily for glob validation.

## Limitations & non-goals (MVP)

- **Only local file artifacts and re-runnable skills are guaranteed.** External side effects (calling external APIs, sending messages, DB writes) may be replayed during crash / retry / goto — user beware
- `fallback_gate` currently only supports `human` (will open up `abort` later)
- Auto gate `decision.field` only supports root-level fields (nested paths like `status.verdict` deferred to v1.x)
- `params` only supports block form (no inline flow-map)

## Validator

Workflow file validator (zero-dep bash):

```bash
~/.claude/skills/skill-orchestrator/bin/lint ~/.claude/workflows/coding.md
```

- Exit code `0` = valid
- Non-`0` = stderr contains `LINT ERROR: ...` with precise location (line / missing field / nonexistent goto target / uninstalled skill, etc.)

The orchestrator runs the same lint at startup (§0.2). You can wire it into your CI to catch workflow regressions.

## FAQ

**Q: Is a workflow itself a Skill?**

No. A workflow is a **declarative configuration file** read by the orchestrator, located in `~/.claude/workflows/` (not `~/.claude/skills/`). Claude Code does not "load a workflow" directly — it loads the `skill-orchestrator` skill, which then loads the workflow file.

**Q: Why not just let Claude chain multiple skills directly?**

You can, but you lose: cross-session recovery, automatic retry, human checkpoints, automatic feedback re-injection, state audit. The orchestrator handles this infrastructure once, so you only worry about "which skills to chain".

**Q: What if my workflow file is wrong?**

The lint script and the orchestrator's preflight check pinpoint the error: which line is missing which field, whether goto targets exist, whether referenced skills are installed — listed one by one. **Aborts immediately** so you can fix it; does not enter fallback (a broken config file is not a runtime issue).

**Q: What if I crash mid-run / hit Ctrl+C by mistake?**

`resume <name>` is enough. `state.json` records full progress: which step, output paths, current loop counts. The orchestrator picks up from the next step after the crash point. If the workflow file changed since last execution, it detects the hash mismatch and prompts "continue with old state / restart with new version / abort".

**Q: Can gates have custom options (e.g. `approve / escalate / defer`)?**

No. Human gate options are fixed at `[approve, reject, abort]` — workflows cannot customize. This is a stable engine contract. Express custom branching with `on_reject: goto:<step>` plus multiple steps.

**Q: Can a single step have multiple goto targets?**

Auto gate's `values` table supports multiple branches (`pass: continue`, `fail: goto:code`, `need_redesign: goto:design`). Human gate's single `reject` event takes one `on_reject` config (`retry` or `goto:<step>`).

**Q: What about steps with external side effects (sending email, calling an API)?**

The MVP does not provide idempotency annotations — such steps may be replayed during crash / retry / goto. Users must handle idempotency themselves inside the skill, or accept duplicates. Future versions may add explicit idempotency declarations.

## Related docs

- [SKILL.md](SKILL.md) — Skill entry point and navigation index
- [reference/workflow-schema.md](reference/workflow-schema.md) — Workflow file structure detail
- [reference/execution-protocol.md](reference/execution-protocol.md) — Full execution protocol
- [reference/workflow-example.md](reference/workflow-example.md) — Complete runnable coding-workflow example
- [reference/execution-sequence.md](reference/execution-sequence.md) — Execution sequence diagrams

## License

[MIT](LICENSE)

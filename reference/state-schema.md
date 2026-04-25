# State.json Schema

State 文件位于 `<项目根>/.autoteam/<workflow_id>/state.json`（或 workflow frontmatter 里 `workspace.state_file` 指定的路径）。

```json
{
  "workflow_id": "string (格式: NNN-<shortid>，如 001-a3b9c7d2)",
  "workflow_skill": "string (name, 文件名去 .md)",
  "workflow_hash": "sha256 hex (防 resume 错配)",
  "autoteam_version": "string",
  "started_at": "ISO 8601",
  "completed_at": "ISO 8601 | null",
  "params": { "<name>": "<value>" },
  "workspace_root": "string (绝对路径)",
  "fallback_gate": "human (MVP 唯一值)",
  "current_step": "string | null",
  "dirty_step": "string | null (防残留)",
  "completed_steps": ["<step.id>", "..."],
  "step_results": {
    "<step.id>": {
      "started_at": "...",
      "completed_at": "...",
      "mode": "main | subagent",
      "outputs": { "<name>": "<path as declared>" },
      "summary": "string"
    }
  },
  "history": {
    "<step.id>": [
      { "generation": 0, "result": { "...": "旧 step_results 结构" } }
    ]
  },
  "loop_counts": { "<step.id>": 0 },
  "retry_counts": { "<step.id>": 0 },
  "pending_feedback": "string | null",
  "pending_feedback_gate": {
    "from_step": "string",
    "gate_kind": "human | auto",
    "decision_artifact": "string (path or text)"
  } | null,
  "status": "initialized | running | paused | completed | failed | aborted"
}
```

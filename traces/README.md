# Agent traces

[Browse the runs](index.html) locally or use the [web viewer](https://theorchemgroup.github.io/Self-verifiable_CyberScientist/traces/). Each task has a Codex run and a Claude Code run, available as JSON and HTML.

Each run JSON has these fields:

```json
{
  "task": "The complete question",
  "harness": {"name": "Codex", "model": "Selected model"},
  "messages": [
    {
      "messages": [],
      "produced_code": [{"name": "Model.lean", "content": "Lean code"}],
      "user_feedback": "Feedback text or accepted"
    }
  ]
}
```

Each group follows one agent cycle. Messages contain the supplied instructions, question and replies, the agent's text, and its tool calls and results. Produced code contains the submitted proof and retained code snapshots. Feedback is plain text; an accepted submission ends with `accepted`.

## Export saved runs

```sh
uv run vcs-trace-export --source-map /path/to/source-map.json --output traces
```

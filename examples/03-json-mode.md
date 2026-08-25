# Example 3 — JSON mode (parse structured events)

## Scenario

Kimi needs to know **which files `pi` actually changed** (not just trust its self-report), or detect specific failures, or feed events into a follow-up step. JSON mode emits one JSON event per line that Kimi can parse.

## Command

```bash
pi --model minimax-cn/MiniMax-M3 --thinking max \
   --mode json \
   -p "在 src/auth/ 里加 password reset flow（路由 + service + 测试）" \
   > run.jsonl
```

## What `run.jsonl` looks like

Each line is a JSON object. Observed `type` values include:

- `session` — session metadata
- `agent_start` / `turn_start` — lifecycle
- `message_start` / `message_end` — assistant text (final answer is in the last `message_end` with `role: assistant`)
- `tool_call` / `tool_result` — every tool invocation (read/write/edit/bash/grep/find/ls)
- `error` — failures

## Parse the events with `jq`

```bash
# List every file pi created or modified
jq -r 'select(.type == "tool_call" and (.tool.name == "write" or .tool.name == "edit"))
       | .tool.input.path' run.jsonl | sort -u

# Did pi hit any errors?
jq -r 'select(.type == "error")' run.jsonl

# Get the final assistant text
jq -r 'select(.type == "message_end" and .message.role == "assistant")
       | .message.content' run.jsonl | tail -1
```

## Why this is more useful than print mode

| Need | Print mode | JSON mode |
|---|---|---|
| Plain-text summary | ✅ | ✅ (parse last `message_end`) |
| Detect failures | ❌ (have to grep prose) | ✅ (parse `error` events) |
| Enumerate files changed | ❌ (have to trust pi) | ✅ (parse `tool_call` events) |
| Re-feed to follow-up turn | ❌ | ✅ (structured payload) |

## Practical pattern for Kimi

```bash
pi --model minimax-cn/MiniMax-M3 --thinking max --mode json -p @plan.md > run.jsonl
PI_EXIT=$?

# Pull out what changed
CHANGED=$(jq -r 'select(.type == "tool_call" and (.tool.name == "write" or .tool.name == "edit"))
              | .tool.input.path' run.jsonl | sort -u)

# Pull out the final summary
SUMMARY=$(jq -r 'select(.type == "message_end" and .message.role == "assistant")
              | .message.content' run.jsonl | tail -1)

# Kimi can now run its own verification on $CHANGED
npm test
```

## When to use

- Multi-file refactors where you want a verifiable file list
- Pipelines where the output of `pi` feeds the next step
- Audit-style workflows where you need to prove what happened

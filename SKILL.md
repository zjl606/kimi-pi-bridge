---
name: invoke-pi
description: Delegate coding or implementation work to the local `pi` coding agent (a separate CLI on this machine, independent from Kimi). Use this skill whenever the user wants Kimi to design a solution and have `pi` execute it — signals include "让 pi 实现", "交给 pi 编码", "用 pi 写", "implement this with pi", "let pi handle the coding", "把方案交给 pi 跑一下", "Kimi 写方案 pi 写代码", "K3 规划 pi 执行", "让 pi 写代码省额度", "delegate to pi", "spawn pi", "run pi on this", "pi 执行". This skill is the calling contract — it tells Kimi how to launch `pi` with the right mode, prompt format, and result parsing. It does NOT itself write code; it only orchestrates the call to pi.
---

# Invoke pi (delegate coding work to the local `pi` agent)

`pi` is a separate coding agent CLI installed on this machine (independent from Kimi). Kimi is the **planner**; `pi` is the **executor**. This skill is the standard way for Kimi to hand off coding/implementation tasks to `pi` and read back the result.

## When to use this skill

Use this skill when the user asks Kimi to:

- Design a solution AND have it implemented in code
- Hand a written plan/spec to a coding agent for execution
- Run an automated coding task on the local repo
- Spawn a long-running coding worker that Kimi will supervise

Do **not** use this skill if:

- The user wants Kimi itself to write the code (Kimi can just do it directly)
- The user wants a one-off shell command (use the bash tool)
- The task is not coding-related (e.g. just running tests, linting, reading files)

## Task routing (the point of this setup)

The division of labor on this machine: **Kimi (K3) plans, pi executes.** K3 quota goes to high-value work — architecture, plans, specs, review. Code execution is offloaded to pi, which runs on its own provider/model (check with `pi --list-models`) and never touches K3 quota.

Route every task into one of three lanes:

- **Simple work** — small edits, one-off commands, reading files, running tests: Kimi does it directly. No plan, no pi call, no K3-heavy reasoning.
- **Real implementation work** — new features, multi-file changes, refactors: Kimi writes the plan, pi implements (the handoff pattern below).
- **Pure planning/docs** — design docs, ADRs, analysis: Kimi alone; pi is not involved.

If a "delegation" would have Kimi write the whole implementation inside the prompt anyway, the task was simple work — just do it directly.

## Choosing a mode

`pi` supports four modes. Pick by how much control Kimi needs:

| Mode | When Kimi needs ... | Command shape |
|------|---------------------|---------------|
| **Print (`-p`)** | One-shot task, plain-text summary back | `pi -p "<task>"` |
| **JSON (`--mode json`)** | Structured events to react to (tool calls, errors, files changed) | `pi --mode json "<task>" > run.jsonl` |
| **RPC (`--mode rpc`)** | Long-lived session, steer pi mid-execution | `pi --mode rpc` (JSONL frames on stdio) |
| **SDK** | Embed pi in Node.js code | Overkill for orchestration |

**Default: use `pi -p`**. Promote to JSON only when Kimi needs to parse specific events; promote to RPC only when a long-running agentic session is genuinely required.

## 1. Print mode (default — 90% of cases)

```bash
# Pass the plan as a single string
pi -p "实现一个 todo 应用：React + Vite，支持增删改查，localStorage 持久化"

# Or pass a plan file
pi -p @plan.md

# Or pipe via stdin and merge into the prompt
cat plan.md | pi -p "按这份方案实现"

# Pass multiple files for context
pi -p @src/old.ts @tests/old.test.ts "重构并补单元测试"
```

Output is plain text on stdout — Kimi reads it as the agent's reply. Exit code 0 = success.

## 2. JSON mode (when Kimi needs structured feedback)

```bash
pi --mode json "重构 utils.ts，把所有 var 改成 const" > run.jsonl
```

Each line is a JSON event with a `type` field — observed types include `session`, `agent_start`, `turn_start`, `message_start` / `message_end`, `tool_call`, `tool_result`, `error`. The final assistant text is in the last `message_end` event whose message has `"role":"assistant"`. Use this when Kimi wants to:

- Detect failures automatically (parse for `error` events)
- Enumerate changed files (parse `tool_call` events with name `edit`/`write`)
- Re-feed results to a follow-up turn

## 3. RPC mode (long-lived, multi-turn)

```bash
pi --mode rpc
```

Then send JSONL frames on stdin, one record per `\n`. **Important**: split frames on `\n` only — do **not** use Node's `readline`, which also splits on Unicode separators and will corrupt JSON payloads.

Use RPC only when:

- Kimi needs to steer pi mid-execution ("stop, change approach")
- The task spans multiple logical turns with shared state
- Kimi is acting as a supervisor loop

For ordinary plan-and-execute handoffs, `-p` is enough.

## Passing context to pi

`pi -p` accepts:

| Form | Example | Notes |
|------|---------|-------|
| Inline string | `pi -p "fix the bug in foo.ts"` | Best for short, self-contained tasks |
| `@file` reference | `pi -p @src/foo.ts "fix the bug"` | File is read by pi and included in context |
| Stdin pipe | `cat plan.md \| pi -p "按这份方案"` | Merged into the prompt |
| Combined | `pi -p @plan.md @src/main.ts "implement"` | Mix files and prompt |

**Recommendation**: write the plan to a file first (e.g. `plan.md` in the project root), then `pi -p @plan.md`. This keeps the prompt clean and the plan auditable.

## Common options

| Option | Purpose | Example |
|--------|---------|---------|
| `--tools <list>` | Whitelist tools (`read`, `write`, `edit`, `bash`, `grep`, `find`, `ls`) | `--tools read,write,edit,bash` |
| `--exclude-tools <list>` | Disable specific tools | `--exclude-tools bash` |
| `--thinking <level>` | `off` / `minimal` / `low` / `medium` / `high` / `xhigh` / `max` | `--thinking max` |
| `--model <pattern>` | Provider/model, e.g. `minimax-cn/MiniMax-M3.1-Flash-Preview`, `sonnet:high` | `--model minimax-cn/MiniMax-M3.1-Flash-Preview` |
| `-c`, `--continue` | Continue the most recent session in this cwd | `-c -p "继续把首页响应式适配"` |
| `--session <path\|id>` | Resume a specific session | `--session <id> -p "..."` |
| `--no-session` | Don't save this run | `--no-session -p "..."` |
| `--approve` / `--no-approve` | Trust / ignore project-local config for this run | `--approve -p "..."` |
| `--offline` | Skip pi's startup update checks | `--offline -p "..."` |

For all delegations on this machine, **always pin the model to `minimax-cn/MiniMax-M3.1-Flash-Preview` and set thinking to `max`**:

```bash
pi --model minimax-cn/MiniMax-M3.1-Flash-Preview --thinking max -p @plan.md
```

**On the model name** — the MiniMax platform product is **MiniMax-M3.1-Flash**, but the id `pi` uses is `minimax-cn/MiniMax-M3.1-Flash-Preview` (provider `minimax-cn`, `-Preview` suffix). Use the `-Preview` id in the command; the bare `MiniMax-M3.1-Flash` will not resolve. `pi` also accepts a `:<thinking>` suffix, so `--model minimax-cn/MiniMax-M3.1-Flash-Preview:max` is equivalent to adding `--thinking max`.

Confirm the id on this machine before the first run:

```bash
pi --list-models minimax
```

This avoids accidental fallback to another provider/model and keeps K3 quota untouched. MiniMax model reference: https://platform.minimax.cn/docs/guides/text-generation

## Reading the result back

After `pi -p` exits, Kimi gets:

- **stdout**: pi's final text response
- **exit code**: 0 = success, non-zero = pi errored (tool failure, rate limit, etc.)
- **stderr**: usually empty unless pi hit an internal error

When Kimi hands off a task, **state explicitly what to expect back**. For example:

> "Run `pi --model minimax-cn/MiniMax-M3.1-Flash-Preview --thinking max -p @plan.md` and tell me: (1) what files were created or modified, (2) whether `npm test` passed, (3) any open questions pi raised."

This makes the result parseable for both Kimi and the user.

## Recommended handoff pattern

1. **Kimi writes the plan** — a clear, complete specification of what to implement. Save it to `plan.md` in the project root (delete it or gitignore it afterwards if it shouldn't be committed).
2. **Kimi invokes pi** — always with explicit model and max thinking: `pi --model minimax-cn/MiniMax-M3.1-Flash-Preview --thinking max -p @plan.md`, with `cwd` set to the project directory (pi sessions are per-cwd; `-c` only finds the right session if the cwd matches).
3. **Run long tasks in the background** — non-trivial pi runs exceed the foreground shell timeout. Launch pi as a background task and read the output when it completes instead of blocking on it.
4. **Kimi verifies independently** — after pi exits, run the project's tests/checks yourself. Don't take pi's self-report at face value.
5. **Kimi reports the result** to the user: summary of what pi did, files touched, verification outcome, any blockers.
6. **Iterate** — if the user wants changes, Kimi writes a follow-up plan and runs `pi --model minimax-cn/MiniMax-M3.1-Flash-Preview --thinking max -p @plan.md` again, optionally `-c` to continue pi's previous session.

## Prerequisites and failure handling

- pi must be installed and authenticated (its own provider credentials — separate from Kimi). If unsure, smoke-test once with `pi -p --offline --no-session "Reply with exactly: PI_OK"`.
- If `pi -p` fails with an auth/provider error, **stop and tell the user** — don't retry in a loop and don't silently do the implementation with K3 instead (that defeats the quota split).
- Flags documented here were verified against pi v1.3.0; `pi --help` is the authoritative reference if something doesn't match.

## Things to avoid

- **Don't ask `pi` to plan what Kimi already planned** — Kimi is the planner; pi is the executor. Don't say "design and implement X". Say "implement X per this plan: ...".
- **Don't use RPC for one-shot tasks** — the JSONL framing is fragile and `-p` is one line.
- **Don't `--approve` blindly** — only when the user has explicitly trusted the project.
- **Don't pipe secrets into the prompt** — pi will see them in context. Reference env vars or files instead.
- **Don't reuse a session across unrelated tasks** — start a fresh `-p` run, or explicitly fork with `--fork <id>`.

## Quick reference card

```bash
# Default on this machine: MiniMax-M3.1-Flash-Preview, max thinking
pi --model minimax-cn/MiniMax-M3.1-Flash-Preview --thinking max -p @plan.md

# With extra context files
pi --model minimax-cn/MiniMax-M3.1-Flash-Preview --thinking max -p @plan.md @src/main.ts

# Continue the previous pi session in this repo
pi --model minimax-cn/MiniMax-M3.1-Flash-Preview --thinking max -c -p "接着把 dark mode 加上"

# Structured feedback (parse run.jsonl afterwards)
pi --model minimax-cn/MiniMax-M3.1-Flash-Preview --thinking max --mode json -p @plan.md > run.jsonl

# Long-lived, steerable agent
pi --model minimax-cn/MiniMax-M3.1-Flash-Preview --thinking max --mode rpc
```

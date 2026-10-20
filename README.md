# invoke-pi

A reusable Agent Skill that teaches **Kimi Code CLI** to delegate implementation work to the local [`pi`](https://pi.dev) coding agent — so Kimi's `K3` quota stays reserved for the high-value work (architecture, plans, review) and routine implementation gets offloaded to `pi`.

> Kimi plans → `pi` executes. Save K3 quota. Cleaner handoff.

This skill is the **calling contract** — it tells Kimi how to launch `pi` with the right mode, prompt format, and result parsing. It does **not** itself write code; it only orchestrates the call to `pi`.

## Who is this for

You run:

- **Kimi Code CLI** (`~/.kimi-code/`) as your primary coding agent
- **`pi`** (`https://pi.dev`) as a secondary coding agent on the same machine
- A model configured in `pi` that has cheap / generous quota and follows tool instructions reliably — by default **`minimax-cn/MiniMax-M3.1-Flash-Preview`**

And you want Kimi to stop burning its K3 quota on multi-file edits, boilerplate, and test writing — and instead hand those off to `pi`.

If your stack is different (e.g. you're driving `pi` from Claude Code, or your cheap model is something else), this skill still works — see [Using with a different stack](#using-with-a-different-stack) below.

## Why

Kimi Code's `K3` quota is finite. Spending it on multi-file refactors, scaffold, or test-writing is wasteful when `pi` is sitting on the same machine with its own provider and quota. The natural split:

- **Kimi**: architecture, planning, specs, code review
- **`pi`**: writing files, running tests, iterating on the codebase

Without a clear contract, Kimi either does the implementation itself (wastes K3) or spawns `pi` with the wrong flags (partial result, retry loop). This skill makes the handoff deterministic.

## Install

### Quick install (any supported agent)

```bash
# Kimi Code CLI (the default target)
git clone https://github.com/zjl606/kimi-pi-bridge ~/.kimi-code/skills/invoke-pi

# pi itself
git clone https://github.com/zjl606/kimi-pi-bridge ~/.pi/agent/skills/invoke-pi

# Claude Code
git clone https://github.com/zjl606/kimi-pi-bridge ~/.claude/skills/invoke-pi
```

Restart the agent if it caches the skills directory. Then just say:

- "Let `pi` implement this"
- "把方案交给 `pi` 跑一下"
- "Use `pi` to write the code, save K3"
- "Kimi 写方案,`pi` 写代码"

Kimi will route the implementation work to `pi` automatically.

## Using with a different stack

The skill ships with defaults tuned for **Kimi Code + pi + minimax-cn/MiniMax-M3.1-Flash-Preview**. To use it with a different calling agent or default model:

| You want to change | Where to edit |
|---|---|
| Calling agent (Claude Code / Codex / Cursor) | Drop into that agent's `skills/` directory; the description's "Kimi" mentions are cosmetic |
| Default model | Replace `minimax-cn/MiniMax-M3.1-Flash-Preview` with whatever `pi --list-models` reports for your install |
| Default thinking level | Replace `max` with whatever your quota can sustain |
| Planners (call it from a non-Kimi agent) | The "Kimi" wording in the description and prose still applies — it's just the *calling agent*, not the product Kimi |

## Quick start

Once installed, this is the default handoff Kimi will use:

```bash
pi --model minimax-cn/MiniMax-M3.1-Flash-Preview --thinking max -p @plan.md
```

That's it. Kimi writes a plan to `plan.md`, then runs `pi` with that file as the prompt. You get back the final result on stdout, exit code 0 on success.

> **Model id note** — the MiniMax platform product is **MiniMax-M3.1-Flash**, but `pi` resolves it as `minimax-cn/MiniMax-M3.1-Flash-Preview`. The bare `MiniMax-M3.1-Flash` will not resolve — always use the `-Preview` id. Check what your own install has with `pi --list-models minimax`.

## What you get

- **`SKILL.md`** — the full calling contract. The YAML frontmatter triggers skill discovery; the markdown is what the calling agent actually reads.
- **`examples/`** — four worked examples covering print / `@file` / JSON / `continue`.
- **`LICENSE`** — MIT, fork-friendly.

## The 5-minute primer

`pi` has four orchestration modes. Pick by how much control Kimi needs:

| Mode | When | Use case |
|---|---|---|
| `pi -p "<task>"` | One-shot, plain-text result | 90% of delegations |
| `pi --mode json ...` | Need to parse events | Detect failures / enumerate file changes |
| `pi --mode rpc` | Long-lived, steerable | Supervisor loops |
| SDK | Embed `pi` in Node.js | Custom tooling |

Always pin model + thinking for reproducibility:

```bash
pi --model minimax-cn/MiniMax-M3.1-Flash-Preview --thinking max -p @plan.md
```

See [`SKILL.md`](./SKILL.md) for the full reference and [`examples/`](./examples/) for concrete workflows.

## Recommended handoff pattern

1. Kimi writes the plan to `plan.md`.
2. Kimi runs `pi --model minimax-cn/MiniMax-M3.1-Flash-Preview --thinking max -p @plan.md` (background if long).
3. Kimi verifies by running the project's tests/lint.
4. Kimi reports results to the user.
5. Iterate with `-c` to continue the `pi` session, or start fresh.

## Prerequisites

- `pi` installed and authenticated. See [pi.dev](https://pi.dev).
- A model in `pi` that handles tool calls reliably. Check with `pi --list-models`.

## License

[MIT](./LICENSE)

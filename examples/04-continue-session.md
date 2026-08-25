# Example 4 — Continue a pi session (`-c`)

## Scenario

The user has feedback after the first handoff. Rather than re-prompting `pi` with the full plan again, you can continue the existing session and `pi` keeps its context.

## Step 1 — initial handoff

```bash
pi --model minimax-cn/MiniMax-M3 --thinking max \
   -p @plan.md
```

`pi` creates the files, runs tests, reports back. Exit code 0.

## Step 2 — user gives feedback

User: "Looks good, but can you also add input validation for the email field?"

## Step 3 — Kimi writes a follow-up plan and re-invokes with `-c`

```bash
# Optional: write a tiny follow-up plan
cat > followup.md <<'EOF'
Add email validation to the signup form:
- Use a regex like /^[^\s@]+@[^\s@]+\.[^\s@]+$/
- Show inline error message on invalid input
- Add a unit test for the validator
EOF

# Continue pi's previous session in this cwd
pi --model minimax-cn/MiniMax-M3 --thinking max \
   -c \
   -p @followup.md
```

`-c` tells `pi` to find the most recent session in the current working directory. The plan from step 1 is still in `pi`'s context — `pi` knows what files it just created.

## Important: cwd matters

`pi` sessions are per-cwd. `-c` only finds the right session if you run the command from the same directory as the original handoff. If you changed directory, pass `--session <id>` instead:

```bash
# List recent sessions
pi --list-sessions

# Resume by id
pi --model minimax-cn/MiniMax-M3 --thinking max \
   --session ses_abc123 \
   -p "接着把 dark mode 加上"
```

## When to start fresh instead

Don't reuse a session across unrelated tasks. If the follow-up is a different feature, start a fresh `-p` run — the old session's context will mislead `pi`.

To explicitly fork an existing session (preserve history but branch):

```bash
pi --model minimax-cn/MiniMax-M3 --thinking max \
   --fork ses_abc123 \
   -p "try a different approach: ..."
```

## When to use RPC instead of `-c`

`-c` is fine for "one more turn" patterns. If you need to steer `pi` mid-execution (e.g. "stop, change approach, retry") or run a true supervisor loop with multiple back-and-forth turns, use RPC mode instead — see `SKILL.md` §3.

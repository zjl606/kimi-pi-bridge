# Example 1 — Print mode (default, 90% of cases)

## Scenario

You have a small, well-scoped coding task: add a function, fix a typo, rename a symbol. You just want `pi` to do it and tell you what changed.

## Command

```bash
pi --model minimax-cn/MiniMax-M3.1-Flash-Preview --thinking max -p "在 src/utils/format.ts 里加一个 toKebabCase 函数，把任何字符串转成 kebab-case，并写一个单元测试"
```

## What happens

1. `pi` reads the project (via its own read tools).
2. It edits `src/utils/format.ts` and adds `src/utils/format.test.ts`.
3. It runs the test suite.
4. It prints a final summary to stdout.

## What Kimi reads back

Kimi parses stdout as the agent's reply. Exit code 0 means success. Stderr is usually empty.

## When to promote

This is fine for tasks Kimi is confident about. If the task is risky (touches a shared module, runs migrations, deletes files), Kimi should:

- Write a plan to `plan.md` first
- Add `--exclude-tools bash` or `--tools read,write,edit` to limit damage
- Verify by running tests itself after `pi` exits

## Variants

```bash
# With a tool whitelist
pi --model minimax-cn/MiniMax-M3.1-Flash-Preview --thinking max \
   --tools read,write,edit \
   -p "在 src/utils/format.ts 里加一个 toKebabCase 函数"

# Read-only (ask pi to review, not change)
pi --model minimax-cn/MiniMax-M3.1-Flash-Preview --thinking max \
   --tools read,grep,find \
   -p "Review src/utils/format.ts for any obvious bugs"
```

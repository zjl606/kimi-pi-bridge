# Example 2 — File reference (`@file`)

## Scenario

The task is too long to fit cleanly in a single prompt, or you want `pi` to see specific files for context. Use the `@file` syntax — `pi` reads the files and merges them into the prompt.

## Pattern: write plan to a file, hand it to pi

```bash
# Step 1: Kimi writes plan.md
cat > plan.md <<'EOF'
# Refactor: split user-service into auth + profile

## Goal
Reduce coupling between auth and profile logic. Currently both live in `src/services/user-service.ts` (480 lines).

## Steps
1. Create `src/services/auth-service.ts` — extract login, logout, token refresh
2. Create `src/services/profile-service.ts` — extract profile CRUD
3. Keep `src/services/user-service.ts` as a thin re-export shim
4. Update all import sites (`grep -r "from.*user-service" src/`)
5. Run `npm test`

## Constraints
- No behavior change. Existing tests must pass.
- Don't touch the public API of `user-service`.
EOF

# Step 2: Hand plan + context files to pi
pi --model minimax-cn/MiniMax-M3.1-Flash-Preview --thinking max \
   -p @plan.md @src/services/user-service.ts
```

## Why this works better than piping the plan

- The plan is **auditable** — you can read `plan.md` after `pi` finishes to see what was promised vs. what was delivered.
- The prompt stays clean — `pi` sees the plan as a separate document, not embedded in CLI args.
- The plan can be **gitignored** if you don't want it in the repo (or `.git/info/exclude` if it's local-only).

## Multiple files for context

```bash
# Plan + multiple source files for context
pi --model minimax-cn/MiniMax-M3.1-Flash-Preview --thinking max \
   -p @plan.md \
   @src/old-module.ts \
   @tests/old-module.test.ts \
   "按 plan.md 改造，补单元测试"
```

`pi` reads all of them and includes them in the prompt context. Order doesn't matter.

## When to use stdin pipe instead

```bash
# You have a one-off snippet in mind, no plan file
cat <<'EOF' | pi --model minimax-cn/MiniMax-M3.1-Flash-Preview --thinking max -p "按这份方案实现"
# Tiny plan
1. Add a `parseDuration` helper to src/time.ts
2. Export from index
3. Add a test
EOF
```

This is fine for one-liners. For anything multi-step, prefer writing the plan to a file.

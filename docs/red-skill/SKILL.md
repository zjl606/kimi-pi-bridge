---
name: invoke-pi
description: 让 Kimi Code 把编码工作交给本地 pi 代理,K3 额度留给方案和评审,实现交给 pi。当用户说"让 pi 实现""用 pi 写代码""Kimi 出方案 pi 写代码""省 K3 额度""delegate to pi"时触发。
---

# invoke-pi — Kimi Code + pi 分工桥

让 Kimi Code(规划者)把编码工作交给本地 pi 代理(执行者),K3 额度留给架构和评审,实现交给 pi,默认 model 走 `minimax-cn/MiniMax-M3.1-Flash-Preview`(量大听话)。

## 一句话工作流

```
Kimi 出方案 → 写 plan.md → 调用 pi 执行 → Kimi 验证结果
```

## 触发场景

- 多文件重构 / 新功能实现 / 测试补全 — Kimi 出方案,pi 写代码
- 用户说"让 pi 写""省 K3 额度""Kimi 规划 pi 执行" — 直接走这个 skill
- 单文件小改 / 简单命令 — Kimi 自己做,不要走 pi

## 4 种调用模式

| 模式 | 何时用 | 命令 |
|------|-------|------|
| **Print**(默认,90% 场景) | 一次性任务,纯文本返回 | `pi --model minimax-cn/MiniMax-M3.1-Flash-Preview --thinking max -p @plan.md` |
| **JSON** | 需要解析工具调用 / 失败检测 | `pi --model minimax-cn/MiniMax-M3.1-Flash-Preview --thinking max --mode json -p @plan.md > run.jsonl` |
| **RPC** | 长会话,中途需要调整方向 | `pi --model minimax-cn/MiniMax-M3.1-Flash-Preview --thinking max --mode rpc` |
| **续接**(`-c`) | 在 pi 已有会话上继续 | `pi -c -p "接着把 dark mode 加上"` |

## 推荐交接流程

1. **Kimi 写方案**到 `plan.md`(清晰、可审计)
2. **调用 pi**(后台跑):`pi --model minimax-cn/MiniMax-M3.1-Flash-Preview --thinking max -p @plan.md`
3. **Kimi 自己验证**:跑项目的 test/lint,不轻信 pi 的自报
4. **报告结果**:改了哪些文件、验证是否通过、有无遗留问题
5. **迭代**:用户给反馈 → 写新方案 → `pi -c` 续接 或 起新会话

## 上下文传递 4 种方式

| 形式 | 例子 |
|------|------|
| 内联字符串 | `pi -p "fix the bug in foo.ts"` |
| `@file` 引用 | `pi -p @src/foo.ts "fix the bug"` |
| stdin 管道 | `cat plan.md \| pi -p "按这份方案实现"` |
| 组合 | `pi -p @plan.md @src/main.ts "implement"` |

**推荐**:方案先写到 `plan.md` 再 `pi -p @plan.md`,prompt 干净、方案可审计。

## 常用参数

| 参数 | 作用 | 例子 |
|------|------|------|
| `--tools` | 工具白名单 | `--tools read,write,edit,bash` |
| `--exclude-tools` | 禁用工具 | `--exclude-tools bash` |
| `--thinking` | 推理深度 off/max/... | `--thinking max` |
| `--model` | provider/model | `--model minimax-cn/MiniMax-M3.1-Flash-Preview` |
| `-c` | 续接本地最近会话 | `-c -p "继续..."` |
| `--session <id>` | 续接指定会话 | `--session ses_xxx -p "..."` |
| `--no-session` | 不保存会话 | `--no-session -p "..."` |
| `--offline` | 跳过启动更新检查 | `--offline -p "..."` |

**强制默认**:`--model minimax-cn/MiniMax-M3.1-Flash-Preview --thinking max`,避免 fallback + 节省 K3。

> **模型 id 坑**:MiniMax 平台产品叫 **MiniMax-M3.1-Flash**,但 `pi` 里的 id 是
> `minimax-cn/MiniMax-M3.1-Flash-Preview`(provider 是 `minimax-cn`,带 `-Preview` 后缀)。
> 写裸的 `MiniMax-M3.1-Flash` **解析不了**。用 `pi --list-models minimax` 查本机实际可用的 id。
> 另外 `pi` 支持 `:<thinking>` 后缀,`--model minimax-cn/MiniMax-M3.1-Flash-Preview:max` 等价于 `--thinking max`。
> **thinking 档位**:该模型强制 adaptive thinking(`pi` 里配了 `forceAdaptiveThinking`),
> 所以 `--thinking off` **并不会**真的关掉思考 —— 照样能跑,但一点 token 都省不下来。
> `off` / `minimal` / `low` / `medium` / `high` / `xhigh` / `max` 全部实测可用,默认保持 `max`。

## 速查卡

```bash
# 默认推荐
pi --model minimax-cn/MiniMax-M3.1-Flash-Preview --thinking max -p @plan.md

# 续接之前的 pi 会话
pi --model minimax-cn/MiniMax-M3.1-Flash-Preview --thinking max -c -p "接着..."

# 解析结构化事件(JSONL)
pi --model minimax-cn/MiniMax-M3.1-Flash-Preview --thinking max --mode json -p @plan.md > run.jsonl
```

## 5 件不要做的事

1. **不要让 pi 重新规划** — Kimi 是规划者,pi 是执行者,告诉 pi "按这份方案实现" 而非"设计并实现"
2. **不要 RPC 做一次性任务** — JSONL 框架脆弱,`-p` 一行搞定
3. **不要盲目 `--approve`** — 只在用户明确信任项目时
4. **不要把密钥塞 prompt** — pi 会在上下文里看到,用 env var 或文件引用
5. **不要跨任务复用会话** — 无关任务起新 `-p`,相关迭代用 `-c`

## 前置条件

- `pi` 已装好并登录(独立于 Kimi 的 provider 凭证)
- 第一次跑先 smoke-test:`pi -p --offline --no-session "Reply with exactly: PI_OK"`
- 认证失败时**停下来告诉用户**,不要重试循环,更不要用 K3 偷偷做

## 验证脚本(可选,JSON 模式)

```bash
# 列出 pi 改过的文件
jq -r 'select(.type == "tool_call" and (.tool.name == "write" or .tool.name == "edit"))
       | .tool.input.path' run.jsonl | sort -u

# 抓最终输出
jq -r 'select(.type == "message_end" and .message.role == "assistant")
       | .message.content' run.jsonl | tail -1
```

---

完整调用细节、Handoff 模式、错误处理见 GitHub 仓库:`github.com/zjl606/kimi-pi-bridge`

---
name: ljg-push
description: 把 ~/.claude/skills/ljg-* 里所有更新过的 Markdown skills 同步到 github repo (ljg-skills) 的 master 分支。Use when user says '/ljg-push', 'push skills', '推送 skills', '同步 skills', 'sync ljg', or whenever ljg-* skills get updated and need shipping. NOT FOR pushing non-ljg skills or arbitrary git repos.
user_invocable: true
---

# ljg-push: 推送 ljg-* skills

把本地 `~/.claude/skills/ljg-*` 里改过的 Markdown skills，一键同步到 github repo 的 `master` 分支。

## 仓库路径（硬编码）

```
SKILLS_REPO="$HOME/code/ljg-skills"     # 本地工作 repo
SKILLS_LOCAL="$HOME/.claude/skills"      # 本地 skill 源
REPO_URL="git@github.com:lijigang/ljg-skills.git"
```

如果 `$SKILLS_REPO` 不存在，脚本会自动 clone。如果它存在但不是 ljg-skills 的 git repo，脚本会报错退出（不破坏现有目录）。

## 输出约定

| 分支 | 输出格式 | 文件扩展 | 加粗 | 文件头 |
|------|---------|---------|------|--------|
| `master`（默认） | Markdown | `.md` | `**bold**` | YAML frontmatter |

`~/.claude/skills/` 里的 skill 必须已经是 Markdown 风格；脚本只同步，不做格式转换。

## 工作流

按 `Workflows/Push.md` 步骤执行 → 调用 `Tools/Push.sh`。

## README 一致性（硬 gate）

每次 push 前，脚本强制做一件事：*把 README 跟 local skills 对一遍*。

- 列出 `~/.claude/skills/ljg-*` 全部 skill 名
- grep `$SKILLS_REPO/README.md` 里出现的 `ljg-xxx`
- 找出 local 有但 README 没有的——*几乎肯定意味着 README 漏更新*
- 命中 → push 中止，报告差异

每次 push 都是检视 README 的机会。问自己：

1. *新增 skill 了吗*？README 的 skill 清单 / 安装命令需要加一行
2. *删了 skill 吗*？README 对应行要删
3. *某个 skill 的描述大改了吗*？README 的简介可能要同步

确认 README 已审、确实不需要更新时，绕过 gate：

```bash
/ljg-push --skip-readme-check
```

## 同步范围

- 检测 `~/.claude/skills/ljg-*` 与 `$SKILLS_REPO/skills/ljg-*` 的差异
- 对有差异的 skill 执行 `rsync -a --delete --exclude='.git'`
- bump patch version（plugin.json + marketplace.json）
- commit 并 push 到 `master`

## Voice Notification

```bash
curl -s -X POST http://localhost:31337/notify \
  -H "Content-Type: application/json" \
  -d '{"message": "Running Push in ljg-push"}' \
  > /dev/null 2>&1 &
```

输出文本：`Running **Push** in **ljg-push**...`

## Examples

*Example 1: 一键推送*

```
User: /ljg-push
→ 检测 ~/.claude/skills/ljg-* 中跟 repo 有差异的 skills
→ master: rsync + bump version + commit + push
→ 报告：哪些 skills 推了，新版本号
```

*Example 2: 看会推什么但不真推*

```
User: /ljg-push --dry-run
→ 列出会被同步的 skills
→ 不执行 rsync / commit / push
```

## Gotchas

- *README 漂移是最容易被忽略的*——加完新 skill 直接推，README 还停在老清单。脚本现在有硬 gate 拦这一刀；拦下来时不要无脑加 `--skip-readme-check`，先去看一下 README
- *脚本前提是 git credentials 已配好*（ssh key 或 PAT）—— ljg-push 不处理认证，认证失败时直接报错
- *untracked 杂物（如 `assets/measure.js`）会被 rsync 同步到 repo*——如果不想推，先在本地删掉，或加进 `.gitignore`
- *脚本会自动 bump patch version 在 plugin.json + marketplace.json*——如果你想 bump minor / major，先手动改完再跑脚本，脚本只追加 patch
- *如果 master 的远端比本地新（继刚另一台机器推过）*，脚本会先 `pull --rebase`；冲突时报错让用户处理
- *搬迁记录*：repo 历史曾在 `~/.claude.backup-20260502/ljg-skills-repo/`（路径名带 backup 是历史遗留），2026-05-02 搬到 `~/code/ljg-skills/`

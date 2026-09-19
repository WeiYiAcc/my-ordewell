---
name: machine-memory
description: >
  本机跨仓与跨项目的稳定事实：sops/age 密钥纪律、dotfiles 三层（mise track + sops +
  home-manager）、jj 写操作规范、工具约定，以及 ordewell 的模型路由（planner 走 gproxy、
  网关模型由哪些 runner 能跑，以及发现源与重启 daemon 的坑）。当任务涉及"怎么用 sops""dotfiles 在哪"
  "该用哪个 runner/模型""这台机器有什么约定"时使用。
---

# machine-memory（跨仓机器事实）

只记**怎么用**，不记**密钥值**。这个文件的正文会原样发给正在使用的 planner 后端。

## 密钥（sops + age）

- 唯一 age 私钥：`~/.config/sops/age/keys.txt`（0600）。它是根信任，out-of-band 分发，不进任何仓库。
- 加密文件用 sops 就地解密、按需取字段：`sops decrypt --extract '["grp"]["key"]' <file>`；
  不要 `sops decrypt <file>` 全量打印。
- 全量 dump、以及 `chezmoi diff` 这类"打印明文 diff"的操作，历史上都泄露过值。默认假设输出会被记录。
- 读取顺序：先看 key 名（`sed -E 's/=.*/=<redacted>/'`），确认要哪个值再取单个字段。

## dotfiles（三层，chezmoi 已退役）

chezmoi 于 2026-09-12 彻底移除，**不要复活**（历史在 `WeiYiAcc/my-dotfiles-linux`）。现在三层：

1. **配置与脚本** — `mise bootstrap dotfiles`（track 模式，origin 仓 `WeiYiAcc/mise-setup`）。
   文件原地不动，mise 只做 checkpoint + 跨机分发。
   `status` / `paths` 查看，`track <path>` 纳管，`save` + `sync` 发布，`pull` 落地。
2. **密钥** — sops + age（见上）。**不要**把 sops 管的密钥文件塞进 mise track。
3. **包 / systemd unit / sessionVariables** — `WeiYiAcc/home-manager` 仓，改完 jj 提交推送。

## VCS：jj native

所有仓库写操作走 jj：`jj describe -m "..."` → `jj new`；推送 `jj git push --bookmark main`。
禁止 `git add/commit/push/checkout/rebase/stash`（尤其 `git add -A`）。git 只读命令可用。
例外：git worktree 只能用 git 管理（`git worktree remove`、`git branch -d`），jj 没有等价操作。

## 工具约定

- 查文件 `fd`，搜内容 `rg`；**禁止 `find`**（WSL 下扫大目录会卡死）。
- `rg` 默认不跟随 symlink：查 symlink 农场（如 skills 目录）要加 `-L`，否则是假阴性。
- 环境里没有 pyyaml；YAML 校验/转换用 `jet --from yaml --to json`。
- shell 带 `http_proxy`：诊断网络先 `env | grep -i proxy`，再用 `curl --noproxy '*'` 复测。
- SSH 到远程跑 nix/mise 装的命令必须 `ssh host 'bash -l -c "..."'`，否则 command-not-found。

## ordewell 模型路由（2026-09-20 验证）

- **Planner** 走 gproxy（OpenAI 兼容网关），配置在 `~/.ordewell/.env`：
  `AI_PROVIDER=openai_compatible` + base URL + `ORCHESTRATOR_MODEL`。
- **Runner 的模型目录来自各 runner 自己的 CLI**，不是 planner 的 provider。planner 能用 `gproxy/*`，
  不代表某个 runner 能跑。
- 网关模型（`gproxy/default` 等）由**能把 CLI 指向该网关的 runner**执行。本机有两条路：
  1. **claude-code**：Claude Code 本身指向网关（`ANTHROPIC_BASE_URL` + `CLAUDE_CODE_ENABLE_GATEWAY_MODEL_DISCOVERY`），
     可用 id 记在 `~/.claude/settings.json` 的 `modelPicker.options`（本机含 `gproxy/default`）。
  2. **opencode**：`~/.config/opencode/opencode.json` 里配了 gproxy provider，凭证在
     `~/.local/share/opencode/auth.json`。
- **历史坑（已修）**：ordewell 的发现源（Anthropic Models API、`claude --help`）看不到网关模型，
  于是 `ordewell task-model <id> gproxy/default` 报 `... was not discovered for claude-code`。
  本仓（`WeiYiAcc/my-ordewell`）的 ADR-0013 让发现改为读 runner 自己 settings 里的 model 行
  （`PluginModelDiscovery.settingsModels`）。**用了这个修法就必须用 fork 的构建**
  （`ordewell` 指向 `~/ghq/github.com/WeiYiAcc/my-ordewell/packages/cli/dist/main.js`，经 `npm link`），
  并且**重启 daemon**——运行中的 daemon 保留旧的发现缓存。
- opencode 那条路不需要这套配置：`ordewell runners opencode on` + `ordewell refresh` 即可。
- **坑**：daemon 的 cwd 若已被删除（例如从 `/tmp` 临时目录启动后又删掉），所有 runner 任务都会失败，
  日志里是 `getcwd: cannot access parent directories` / `mise WARN Current directory does not exist`。
  修法：从仍然存在的目录重启 daemon（`ordewell stop --server`，再在有效目录里跑任意 ordewell 命令）。
- **坑：claude-code 在"没见过的目录"会卡在信任对话框**。Claude Code 首次进入某目录会弹
  `Security guide / Yes, I trust this folder / No, exit`，光标默认停在 `No, exit`；
  `--dangerously-skip-permissions` **不覆盖**它（该对话框只在非交互 `-p` 下跳过，而 ordewell 在 tmux 里开交互 pane）。
  症状：runner 任务永远 `in_progress`，工作区无产物。修法：启动前预登记信任——
  在 `~/.claude.json` 的 `projects["<绝对路径>"]` 写 `hasTrustDialogAccepted: true`
  （新 worktree 还要 `hasClaudeMdExternalIncludesApproved` / `hasClaudeMdExternalIncludesWarningShown`）；
  firstmate 的 `bin/fm-claude-trust.sh` 就是干这个的。
- **跑本仓测试的坑**：如果 shell 里 export 了 `GEMINI_API_KEY` / `GEMINI_BASE_URL`，
  `packages/web` 和 `packages/vscode` 各会挂一个用例（provider 探测解析成 google）。
  用 `env -u GEMINI_API_KEY -u GEMINI_BASE_URL` 跑就通过。

## Ordewell 记忆纪律

`ORDEWELL.md` 只有 8000 字符且静默截断，**不是**记忆库。记忆分层与各使用方式（CLI / TUI / API / VS Code）
的落点见 `docs/agent-memory.md`（在 `WeiYiAcc/my-ordewell` 仓里）。

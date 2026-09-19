# ORDEWELL.md

> Planner 的 always-on 前置上下文。Ordewell 只把这一个文件塞进 planner 系统提示词，并且**硬截断 8000 字符**
> （`ContextCollector.ts`，静默、UTF-16 单位）。所以：这里只放**每次规划都必须先看到**的东西。

## 硬规则

1. **规划期只读**：不要编辑/创建/删除文件，不要跑会改动工作区的命令（探索信封，ADR-0008）。
2. **仓库写操作一律 jj**：`jj describe -m "..."` → `jj new`；推送 `jj git push --bookmark main`。
   禁止 `git add/commit/push/checkout/rebase/stash`（git 只读命令可用）。
3. **密钥永不进入任何会被发给模型的文本**：本文件、`AGENTS.md`、skills 都会原样发给当前 planner 后端。
   sops/age 只记"怎么用"，绝不记"值"。明文密钥只允许在宿主进程内解密后注入。
4. **正文用中文**；命令注释、临时变量名可用英文。

## 记忆放在哪（本仓约定）

| 内容 | 位置 | 为什么 |
|---|---|---|
| 跨仓 / 机器事实（sops、dotfiles、mise、jj、模型路由） | 技能 `machine-memory`，用 `/machine-memory` 调用 | 全局技能无长度上限，且可跨仓复用 |
| 本仓事实（架构、命令、约定） | `AGENTS.md`（索引）+ `docs/`（细节） | planner 提示词第一条就命令它先读 `AGENTS.md` |
| 每次规划都要先看到的少数几条 | 本文件 | 唯一 always-on，但只有 8000 字符 |

**不要把 ORDEWELL.md 当记忆库**：超 8000 字符会被静默切掉，没有提示。设计说明见
[docs/agent-memory.md](docs/agent-memory.md)。

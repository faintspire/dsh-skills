---
name: local-asset-git-snapshot-sync
description: 把本机 DSH 可携带资产（技能包、~/.mnemon 记忆、MCP 配置）按 target 分仓快照同步到各自的私有 git 仓库：清空重建受管子目录、每仓生成 index.json 清单、无变化时不产生提交，并核对远端落地与幂等；用户说「把技能/记忆/MCP 备份到私有仓库」「快照同步到 git」「每天自动推一次也能手动推」「本地删掉的远端也要消失」「memory 目录里到底存了什么」时使用。
whenToUse: 用户要求把本地技能包、用户/项目记忆或 MCP 配置备份到私有 git 仓库；要求多 target 分仓、互不混仓；要求删除能传导到远端、没变化不产生空提交；要求先跑不联网的端到端验收再推真实远端；或要求解释 ~/.mnemon 的 memory 里到底存了什么、是不是「个人记忆」。仅做远端地址/HTTPS 与 SSH 预检时不需要本技能。
---

# 本地资产分仓 git 快照同步

插件本体在 `C:\Users\liuzk\Documents\deepseek-harness\default-workspace\dsh-vault`（`index.js` / `vault.json` / `test/verify.mjs` / `scripts/install.mjs`），远端 `git@github.com:faintspire/dsh-vault.git`。
数据按 target 分三个仓库，每个 target 只被管理一个子目录：

| target | 数据仓库 | 本地 workdir | 被管理子目录 | 源 |
|---|---|---|---|---|
| skills | `git@github.com:faintspire/dsh-skills.git` | `C:\Users\liuzk\.dsh\vault\dsh-skills` | `skills/` | `~/.dsh/skills` + `~/.agents/skills` |
| memory | `git@github.com:faintspire/dsh-memory.git` | `C:\Users\liuzk\.dsh\vault\dsh-memory` | `memory/` | `~/.mnemon/runtime` + `~/.mnemon/documents` |
| mcp | `git@github.com:faintspire/dsh-mcp.git` | `C:\Users\liuzk\.dsh\vault\dsh-mcp` | `mcp/` | 以 `vault.json` 里 mcp target 的配置为准 |

## Steps

1. 先读配置，不要凭记忆猜源路径。编辑 `index.js` 前必须先 `read` 一次，否则写入会被拒（`file has not been read`）。
   ```powershell
   cd "C:\Users\liuzk\Documents\deepseek-harness\default-workspace\dsh-vault"
   Get-Content vault.json
   ```
   逐个确认 `targets[].repo.url / branch / workdir` 与源根目录；缺哪条就以 `vault.json` 为准补齐，不要自造路径。

2. 取证 memory target 的真实构成（对外解释必须先有数字）：
   ```powershell
   Get-ChildItem "C:\Users\liuzk\.mnemon\runtime" -File | Select-Object Name, Length, LastWriteTime
   node -e "const j=require('C:/Users/liuzk/.mnemon/runtime/memories.json'); const e=j.entries; const by={}; for (const x of e) { const k=x.scope||x.kind||x.target||'?'; by[k]=(by[k]||0)+1 } console.log('entries:', e.length); console.log('scope 分布:', JSON.stringify(by))"
   ```
   再数 `~/.mnemon/documents/active/*.md` 的篇数与总字节。解释口径写在下面「memory target 的实际构成」一节，别临场发挥。

3. memory target 只拷贝这些内容，**不**包含 `trajectories` / `ledger` / `evals`：
   - `memory/runtime/USER.md`（个人记忆，本次 0 字节也照常上传）
   - `memory/runtime/MEMORY.md`（项目记忆）
   - `memory/runtime/memories.json`（上面两份的事实源）
   - `memory/documents/active/*.md`（成篇项目档案）
   - `memory/documents/index.json`

4. 每次同步先清空再整体重建受管子目录，因此本地已删条目在远端同步消失；仓库根的 `README.md` / `LICENSE` 等手写文件不属于受管子目录，不会被覆盖。仓库根放 `.gitattributes`（`* text=auto eol=lf`）消掉 CRLF 警告。

5. 先跑离线端到端验收，不访问网络：
   ```powershell
   node test\verify.mjs
   ```
   期望一次列出全部 PASS：两个/三个仓库都首次推送、计数、空 target 仍推、技能包内附加文件一并上传、无 `SKILL.md` 的目录被跳过、互不混仓。任一 FAIL 先修再谈真实远端。

6. 真实首推（全量或单 target）：
   ```powershell
   node index.js sync --config vault.json
   node index.js sync --config vault.json --target memory
   ```
   输出形如 `[vault] 已推送到 git@github.com:faintspire/dsh-memory.git (origin/main): sync: 2026-11-02 09:52:34 · 2 memory`。这里的 `2 memory` 是「runtime + documents 两个分组」，不是文件数；文件面另算（实测 66 文件 / 488 KB）。`mcp` 当前没有条目也要照常提交推送到自己的仓库，不要跳过。

7. 推送后立刻核对远端落地：
   ```powershell
   $m = "C:\Users\liuzk\.dsh\vault\dsh-memory"
   git -C $m log --oneline
   git -C $m ls-tree --name-only HEAD
   git -C $m ls-tree -r --name-only HEAD | ForEach-Object { ($_ -split '/')[0..2] -join '/' } | Group-Object | Select-Object Count, Name
   ```
   顶层应只有 `README.md` / `index.json` / `memory/`；`index.json` 逐项记 `name` / `description` / `sha256` / 文件数 / 字节数。

8. 给每个数据仓补根 README：说明受管子目录每次同步整体重建、**不要手改**，并给出恢复到本机的步骤（技能拷回 `~/.dsh/skills/`；MCP 配置拷回对应配置路径后重启 DSH；记忆按 runtime/documents 结构放回）。写完 `git add -A` → `commit` → `push origin main`，随后再跑一次同步确认手写文件不被覆盖。

9. 给插件本体仓打 annotated tag 并推送：
   ```powershell
   git tag -a v0.3.0 -m "dsh-vault 0.3.0: 技能 / 记忆 / MCP 三 target 分仓同步"
   git push origin v0.3.0
   git ls-remote --tags git@github.com:faintspire/dsh-vault.git
   ```
   远端应看到 `refs/tags/v0.3.0` 与 `refs/tags/v0.3.0^{}` 两行。

10. 收尾一次性打印交付面，并跑二次同步验幂等：
    ```powershell
    git log --oneline -2
    foreach ($d in @("dsh-skills","dsh-memory","dsh-mcp")) { $p = "C:\Users\liuzk\.dsh\vault\$d"; "$d : " + (git -C $p log --oneline -1) }
    node index.js sync --config vault.json
    ```
    二次同步必须输出 `skills: 无变化 / memory: 无变化 / mcp: 无变化`，且三个数据仓的提交号与上一步一致（没有空提交）。

## memory target 的实际构成（对外解释口径）

**`memory` 不等于「个人记忆」。** 它是三层 + 一个事实源：

| 文件 | 是什么 | 本机实测 |
|---|---|---|
| `runtime/USER.md` | 个人记忆（你是谁、偏好、习惯） | **0 字节** → 目前没有任何个人记忆条目，但仍照常上传 |
| `runtime/MEMORY.md` | 项目记忆（工作约定、踩坑、决策） | 10 节 / 5154 B |
| `runtime/memories.json` | 上面两份的事实源 | 11 条，`scope` 分布 `{"memory":11}`，全部是 `target=memory` |
| `documents/active/*.md` | 成篇项目档案（调研、方案） | 59 篇 / 476 KB |
| `documents/index.json` | 档案索引 | 1 个 |

对应解释模板：「这里存的不是个人记忆，共三层——`USER.md` 是个人记忆，`MEMORY.md` 是项目记忆，`documents/active` 是成篇项目档案，`memories.json` 是上面两份的事实源；你现在 `USER.md` 是 0 字节，`memories.json` 里 11 条全是 `target=memory` 的工作记忆，所以严格说一条个人记忆都没有。」整个 memory 仓同步产物实测 66 文件 / 488 KB；`trajectories` / `ledger` / `evals` 不入库。

## Pitfalls

- **别去 `~/.dsh` 里找记忆**。记忆在 `~/.mnemon`（`runtime` + `documents`）；在 `~/.dsh` 递归搜 `MEMORY.md`/`USER.md` 只命中 `.git` 对象等噪声，而且全盘扫很慢——定位到就 `job_kill`，别等它跑完。
- **别把整个 memory 目录笼统叫「个人记忆」**。按上面的三层表说，并给出实测数字（`USER.md` 0 字节、11 条全 `target=memory`）。
- **远端仓库不存在时不要自己造**。`git ls-remote --exit-code <url>` 返回 `exit=128` + `ERROR: Repository not found` 表示仓库还没建，交给用户决定；返回 `exit=2` 表示仓库在但还没有 `main`，此时同步会打 `[vault] pull 失败，继续用本地副本: fatal: couldn't find remote ref main`，这是正常的，继续用本地副本推送即可，不要中止。
- **改文件先读**。`write`/`edit` 直接覆盖没读过的 `index.js` 会失败（`file has not been read`）；先 `read` 再改。
- **网络：全程走 SSH**。本机 HTTPS 连 GitHub 会 `Recv failure: Connection was reset`，`web_fetch` 打仓库页也会 `TypeError: fetch failed`；用 `git@github.com:` 地址，用 `git ls-remote` 验证仓库，不要用网页验证。
- **别让远端残留已删条目**。受管子目录必须每次先清空再重建；手写 README/License 放仓库根，别放进受管子目录。
- **空 target 不跳过**。`mcp` 当前没有配置条目，仍要生成 `index.json`、提交并推送，否则仓库永远不存在。
- **计数别读错**。同步日志里的 `· 2 memory` / `· 38 skills` 是分组/条目数，不是文件数与体积；文件面单独统计。
- **改名或改 target 后要重建 profile 链接**：`node scripts/install.mjs` 更新 `dependencies` 的 `link:` 条目、`dsh.profile.bundles` 与 `node_modules` 链接，再用 `node -e "import('file:///C:/Users/liuzk/.dsh/profiles/desktop/node_modules/dsh-vault/index.js')"` 验证能加载。

## Verification

- `node test\verify.mjs` 全部 PASS：首次推送、各 target 计数、空 target 仍推、无 `SKILL.md` 目录被跳过、附加文件一并上传、`index.json` 存在、MCP 不混进技能仓、记忆与技能互不混仓。
- 每个数据仓 `git -C <workdir> ls-tree --name-only HEAD` 顶层只有 `README.md` / `index.json` / 受管子目录；`ls-tree -r` 里 memory 仓无 `trajectories` / `ledger` / `evals`。
- 记忆面数字对得上：首推时记录文件数与体积（实测 66 文件 / 488 KB），其中 `USER.md` 0 字节、`documents/active` 59 篇 / 476 KB、`MEMORY.md` 10 节。
- 二次 `node index.js sync --config vault.json` 输出三行「无变化」，`git log -1` 提交号不变，`git status --porcelain` 干净。
- 插件仓 `git log --oneline -2` 显示最新提交已被推上 `origin/main`，`git ls-remote --tags` 含 `v0.3.0` 与 `v0.3.0^{}`；收尾打印的三个数据仓最后提交与各自 `git log -1` 一致。

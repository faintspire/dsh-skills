---
name: mind-garden-note-write
description: "把技术讨论整理成 mind-garden 工作笔记并发布：读仓库 AGENTS.md、grep 全仓查重并当场判定是修订既有笔记还是新建、对齐既有笔记的 frontmatter 与结构、在脏工作区下用 git fetch 加 git rev-list 判定是否需要同步（不 stash、不强行 rebase）、写 work/项目/YYYY-MM-DD-标题.md、在 work/README.md 索引表插入指向新笔记的行并与笔记放在同一个提交、只暂存新笔记与 README（其余既存改动保持未暂存）、用 note(scope): 提交并推送到 origin/main；同一请求里还含代码改动时先备份代码、评估可行性，笔记只记结论与可验证理由。当用户说「记笔记」「记一下」「整理成工作笔记」「同步笔记」「写进 mind-garden」「发布笔记」，或要求把一段排障/方案讨论落盘到笔记仓库并推送时使用。"
whenToUse: "用户要求把技术讨论、排障过程、方案设计、代码配置整理成可长期修订的仓库笔记并提交推送到 origin/main 时使用；也用于笔记仓库工作区有他人未提交改动、pull --rebase 被挡住而仍需新增笔记并推送的场景；以及同一请求里既要改代码又要写笔记（先备份代码、评估可行性、笔记只记结论）的复合任务。"
---

# mind-garden 工作笔记落盘与发布

把「记录发生了什么」写成「记录想明白了什么」，落到 `mind-garden` 仓库并在脏工作区下安全提交推送。本地路径 `C:\Users\liuzk\Desktop\mind-garden`（git-bash 下为 `/c/Users/liuzk/Desktop/mind-garden`），远端 `origin/main`。

## Steps

1. 请求里若还含代码改动，先备份、再评估可行性（这一步必须在写笔记之前完成）：
   ```bash
   cd /c/Develop/project/biovisart-server
   mkdir -p .scratch/backup/$(date +%Y%m%d)-<主题>
   cp <待改/待评估的每个文件> .scratch/backup/$(date +%Y%m%d)-<主题>/
   ls -l .scratch/backup/$(date +%Y%m%d)-<主题>/
   ```
   回复里说明回滚方式（把文件拷回原路径即可）与 `.scratch/` 不进编译。判定可行性前先读下游方法签名与调用点，而不是只看上传端：
   ```bash
   grep -rn "<下游方法名>" --include=*.java . | head
   ```
   例：`AiGalleryServiceImpl` 那条链路的下游返回 `BufferedImage`，内存直传在该链路就不成立。笔记只写结论 + 可验证理由（`类:行号`、关键签名），不贴实现代码。

2. 读仓库约定与结构（每次进仓库第一件事）：
   ```bash
   cd /c/Users/liuzk/Desktop/mind-garden && ls && cat AGENTS.md
   cat work/README.md && ls templates && cat templates/note.md
   ```
   确认归类（工作笔记进 `work/`）、目录层级 `<项目或主题>/`、命名 `YYYY-MM-DD-<标题>.md`、脱敏要求。

3. 全仓查重，并当场决定「修订既有」还是「新建」：
   ```bash
   cd /c/Users/liuzk/Desktop/mind-garden && date +%F
   grep -ril "<关键词1>\|<关键词2>\|<英文同义词>" . --include=*.md | head -20
   ```
   命中时读它（`sed -n '1,40p' <命中笔记>`）后再决定，并在回复里给出明确结论：「已命中 `work/<项目>/<文件>.md`，本次在它上面修订」或「无同主题笔记，新建 `work/<项目>/YYYY-MM-DD-<标题>.md`」。不要停在「若已有同类笔记则修订」这类条件句上——查重必须落到具体动作。

4. 看待提交前的仓库状态与分支：
   ```bash
   git status --short && git branch --show-current
   ```
   记下已存在的未暂存改动（他人或自己以前留下的）与未跟踪目录（如 `.obsidian/`），后面绝不能碰。

5. 对齐既有笔记的写法（frontmatter + 结构）：
   ```bash
   sed -n '1,60p' "work/<项目>/<最近同类笔记>.md"
   tail -25 "work/<项目>/<最近同类笔记>.md"
   ```
   沿用 `title` / `date` / `category` / `tags` / `status` 五项，正文以「一句话结论」开头，再展开链路、配置、取舍。

6. 判定与远端差距（脏工作区禁止直接 `git pull --rebase`）：
   ```bash
   git fetch origin main
   git rev-list --left-right --count HEAD...origin/main
   git diff --stat
   ```
   读输出两列：`0\t0` 表示与远端一致，不需要同步，直接写笔记；第一列是本地领先数，第二列是远端领先数。落后时不要 stash、不要强推，先把自己的改动提交后再同步，同步方式与时机向用户说明并让其确认。

7. 写笔记文件，路径 `work/<项目>/YYYY-MM-DD-<标题>.md`，写完立刻回读确认真的落盘：
   ```markdown
   ---
   title: 接口鉴权链：Sa-Token 拦截器配置
   date: 2026-10-30
   category: work
   tags: [sanger, 鉴权, sa-token, spring-mvc]
   status: draft
   ---
   # 接口鉴权链：Sa-Token 拦截器配置
   ## 一句话结论
   ...
   ```
   用 tools.write / tools.edit 直接写绝对路径 `C:\Users\liuzk\Desktop\mind-garden\work\<项目>\YYYY-MM-DD-<标题>.md`，不要手工拼 heredoc 里的 frontmatter；写完回读：
   ```bash
   cd /c/Users/liuzk/Desktop/mind-garden && wc -l "work/<项目>/YYYY-MM-DD-<标题>.md" && head -12 "work/<项目>/YYYY-MM-DD-<标题>.md"
   ```
   结论写取舍与原因（例如预检请求为什么必须放行 `OPTIONS`、为什么必须放行 `/error`），不要把代码原样贴一遍。

8. 读 `work/README.md` 的「## 索引」区域，插入指向新笔记的行（与笔记同一提交）：
   ```bash
   grep -n '^## 索引' work/README.md
   sed -n '14,60p' work/README.md
   ```
   表头为 `| 日期 | 项目 | 标题 | 类型 |`；用 edit 工具把新行插在该表最后一行之后，格式 `| <date> | <项目> | [<标题>](<项目>/YYYY-MM-DD-<标题>.md) | <方案设计/机制说明/复盘> |`，插完回读确认（`grep -n "<标题>" work/README.md`）。索引行和笔记必须一起进同一个提交，不能只提交其中一个。

9. 只暂存自己的文件，绝不用 `git add -A` / `git add .`：
   ```bash
   git add "work/<项目>/YYYY-MM-DD-<标题>.md" work/README.md
   git status --short
   ```
   确认输出里是 `A  work/<项目>/…笔记.md` 和 `M  work/README.md`，而他人的未暂存改动仍显示为 ` M …`、`.obsidian/` 仍是未跟踪。

10. 提交并推送，提交信息格式 `note(<scope>): <摘要>`，scope 取 `work`：
    ```bash
    git commit -m "note(work): 新增 <标题> 笔记" && git push origin main
    ```

11. 复核落地结果：
    ```bash
    git log --oneline -1 --stat
    git fetch origin main && git rev-list --left-right --count HEAD...origin/main
    git status --short
    ```

## Pitfalls

- `git pull --rebase origin main` 在有未暂存改动时直接失败（`cannot pull with rebase: You have unstaged changes.`）。不要因为这条报错就停下，也不要 stash 或丢弃别人/work 里已有的改动，改用第 6 步的 `git fetch` + `git rev-list --left-right --count HEAD...origin/main` 判定。
- 多用户共写的仓库里 `git add -A` 会把 `.obsidian/` 和他人未提交改动一起带进提交。逐个路径 add（笔记 + `work/README.md` 两个路径），提交后 `git status --short` 仍应看到那些改动是未暂存/未跟踪。
- 中文/含空格的文件名在 git 命令里可能被转义成八进制显示，`git add` 时用双引号包住完整路径。
- 日期用当天真实日期（从环境时间取 `date +%F`），不要照抄参考笔记的日期；同一天多篇笔记就追加多行索引。
- 查重只报「命中了哪些文件」不够，必须当场判定修订还是新建并说出文件名；否则等于没查重。
- 笔记只贴代码等于没写：`一句话结论` 必须说清为什么这样选、不这样选会踩什么坑；尾巴可加「顺带发现的坑」。
- 答应改代码前先读下游方法签名/调用点：只要链路下游要的是 `BufferedImage` 这类内存对象，硬推「内存直传」就是伪方案；早点 grep 到它，比写完方案再被推翻便宜。
- 复合任务里笔记是结论载体：先备份（第 1 步）再评估，笔记里只留结论与 `类:行号` 理由，不搬大段实现。
- 本会话 workspace 是代码仓（如 `C:\Develop\project\biovisart-server`），`mind-garden` 在 workspace 之外，写文件可能被沙箱拒绝。被拒时不要停在回复里的代码块草稿：改用 tools.win_bash 在 git-bash 里写同一路径（`/c/Users/liuzk/Desktop/mind-garden/work/...`）并回读；两条路都被拒就明确报告沙箱限制并给出可执行命令，不要声称「已整理好」。
- 没有真正 `git add` → `git commit` → `git push` 就汇报，视为未完成：用户要的是仓库里多一篇笔记并推到 `origin/main`，不是一份待执行计划。
- 仓库按公开标准脱敏：不写客户名、内网地址/域名、密钥、真实数据，项目名用代号。

## Verification

- `git log --oneline -1 --stat` 显示 `note(work): ...`，且同一次提交里同时列出新笔记与 `work/README.md`；`git push` 无报错；
- `git fetch origin main && git rev-list --left-right --count HEAD...origin/main` 为 `0\t0`，证明已与远端一致；
- `git status --short` 只剩提交前就存在的未暂存改动和未跟踪目录（如 `.obsidian/`），没有残留未提交的新笔记或 README；
- `ls -l "work/<项目>/YYYY-MM-DD-<标题>.md"` 文件真实存在（不是只出现在回复代码块里），`head -12` 显示完整 frontmatter（title/date/category/tags/status）与「一句话结论」段；
- `grep -n "<标题>" work/README.md` 能定位到索引表里指向该笔记的那一行。

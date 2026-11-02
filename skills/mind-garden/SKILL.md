---
name: mind-garden
description: 把工作笔记、专业知识、AI 学习内容写入用户桌面上的 mind-garden 笔记仓库，并提交推送到 GitHub。当用户说「记一下」「记笔记」「工作笔记」「学习笔记」「同步笔记」「整理笔记」或要求整理个人知识库时使用。
---

# mind-garden 笔记入库

把用户口述或会话中产生的知识，落盘到个人笔记仓库 `mind-garden`，并提交推送到远端。

## 仓库位置

- 本地：`C:\Users\liuzk\Desktop\mind-garden`
- 远端：`git@github.com:faintspire/mind-garden.git`（分支 `main`）
- 仓库有自己的 `AGENTS.md`，文件格式、命名、提交规范的**最终依据在那里**，本技能只做入口和流程。

## 归类规则

| 内容 | 位置 |
|---|---|
| 具体项目、团队、业务、线上问题、方案复盘 | `work/<项目>/YYYY-MM-DD-<标题>.md` |
| 可复用的流程、检查清单、自动化 | `skills/<name>/SKILL.md` |
| AI / LLM 相关 | `ai/<子领域>/<标题>.md` |
| 八股、面试问答卡片 | `knowledge/qa/<主题>.md` |
| 其余通用技术知识 | `knowledge/<领域>/<标题>.md` |

判断不了先问一句，不要新建顶层目录。

## 步骤

1. 进入仓库：`cd C:\Users\liuzk\Desktop\mind-garden`
2. 拉取最新：`git pull --rebase origin main`
3. 按模板写笔记（`templates/note.md`、`templates/qa-card.md`、`templates/work-log.md`），补全 front matter：`title / date / category / tags / status`
4. 更新对应目录 `README.md` 的索引表
5. 检查：`git status --short`，确认没有意外文件
6. 提交：`git add -A && git commit -m "note(<scope>): <摘要>"`
7. 推送：`git push`
8. 汇报：新增/修改的文件、提交信息、推送结果

## 写作要求

- 中文，短句，先给结论
- 每篇至少一条可复用结论，没有就标 `status: draft`
- 代码块标语言；引用其他笔记用相对路径
- 八股卡片五段：问题 / 一句话答案 / 展开 / 追问 / 我踩过的坑

## 安全

- 不写密钥、token、密码、客户名、内网域名、IP、真实数据
- 可疑内容先停下，指出文件并给出脱敏建议，等用户确认

## 失败处理

- 冲突：解决后 `git rebase --continue`，不要 `--abort`
- push 被拒：先 `git pull --rebase` 再推，**不要 force push**
- SSH 报错：停下报告，让用户执行 `ssh -T git@github.com` 检查
- 当前环境沙箱若拦截 git 网络操作，说明需要用放宽权限的方式重试一次

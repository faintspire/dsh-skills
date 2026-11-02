# dsh-skills

DSH 技能备份仓库，由 [dsh-vault](https://github.com/faintspire/dsh-vault) 自动维护。

- `skills/<name>/` —— 技能包（含 `SKILL.md` 及同目录的脚本、参考资料）。每次同步整体重建，**不要手改**。
- `index.json` —— 每个技能的名称、描述、sha256、文件数、字节数。

## 恢复到本机

把需要的 `skills/<name>/` 整个目录复制到 DSH 的技能根目录之一：

- `~/.dsh/skills/` —— 界面里的「用户」分组
- `~/.agents/skills/` —— 「用户 · agents」分组

重启 DSH 后即可在「技能管理」里看到。

## 手动更新

```powershell
node index.js sync --config vault.json --target skills
```
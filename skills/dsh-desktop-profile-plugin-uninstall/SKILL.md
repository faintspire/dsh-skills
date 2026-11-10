---
name: dsh-desktop-profile-plugin-uninstall
description: 在没有可用 dsh CLI 的 DSH 桌面端（桌面 profile，Windows/PowerShell）手动卸载一个插件包：先确认唯一生效 profile、按 package.json 的 name/version 摆出名字相近的候选并提问定死「卸哪个 + 是否留数据」、带时间戳备份、同步删 dependencies 与 dsh.profile.bundles、把 cordis.patch.yml 里引用该模块的条目整段注释掉、再删 node_modules 下的 scope 目录并用 node 解析 JSON/YAML 验收；当用户说「卸载某个插件」「把插件市场装的包删掉」「web-ui 全家桶不要了」「卸载插件但数据都留着」「dsh plugin 命令不存在怎么卸」时使用。
whenToUse: 用户在 DSH 桌面端要卸载/移除某个已安装插件包（如 @linxin666/dsh-web-all），或 dsh CLI 不在 PATH 无法用 CLI 卸载插件时使用。不适用于安装插件、切换 profile、或清理 ~/.dsh 里的会话/记忆数据。
---

# 卸载 DSH 桌面端 profile 里的插件包（手改路线）

CLI 不可用时，插件卸载等于在**一个 profile 目录**里同步改三处并删一个目录。三处必须一致，否则启动时找不到模块。

下方命令以 `$dshHome = $env:USERPROFILE\.dsh`、profile 名 `desktop` 为例；实际路径与 profile 名以第 1 步的读数为准（本技能来自 `<user>\.dsh\profiles\desktop` 的真实操作）。

## Steps

1. **列出 DSH 主目录与 profiles，确认唯一生效 profile**

   ```powershell
   $dshHome = Join-Path $env:USERPROFILE ".dsh"
   Get-ChildItem $dshHome -Force | Select-Object Mode,Name | Format-Table -AutoSize
   "--- profiles ---"
   if (Test-Path "$dshHome\profiles") { Get-ChildItem "$dshHome\profiles" -Force | Select-Object Name | Format-Table -AutoSize }
   ```

   只有一个 profile（实测为 `desktop`）就直接用；有多个则先让用户确认是哪一个，不要凭目录名猜。

2. **读 profile 的 package.json 与 node_modules，摆出真实安装的候选包**

   ```powershell
   $p = "$dshHome\profiles\desktop"
   Get-ChildItem $p -Force | Select-Object Mode,Name | Format-Table -AutoSize
   Get-Content "$p\package.json" -Raw
   ```

   列出该 scope 下每个包的真实 `name` / `version` / `description`，再据此区分名字相近的候选：

   ```powershell
   $base = "$p\node_modules\@linxin666"
   Get-ChildItem $base -Directory | ForEach-Object {
     $j = Join-Path $_.FullName "package.json"
     if (Test-Path $j) { $o = Get-Content $j -Raw | ConvertFrom-Json
       [PSCustomObject]@{ dir=$_.Name; name=$o.name; version=$o.version; description=$o.description } } } | Format-Table -AutoSize -Wrap
   ```

   同时读 `$p\cordis.patch.yml`：里面的 `id` 条目会按 `name: "@scope/xxx"` 引用该包带进来的子插件，先记下要处理哪几条。

3. **探测 CLI，只探一次**

   ```powershell
   $c = Get-Command dsh -ErrorAction SilentlyContinue; if ($c) { $c.Source } else { "dsh not on PATH" }
   ```

   不在 PATH 就是**手改路线**：不要接着跑 `dsh plugin remove` / `dsh plugin --help` 反复碰壁（实测直接报「无法将 dsh 项识别为 cmdlet」），也不要为卸载去装 CLI。

4. **一次问清两件不可逆的歧义**，用 `ask_user_question` 一个调用里放两个问题：
   - 卸载哪一个（把第 2 步的真实包名+版本+一句话影响摆成选项，例如「@linxin666/dsh-web-all（Web UI 全家桶）卸掉后面板和工具一起消失」）
   - 是否保留数据（推荐「只卸插件，数据全留着」）

   拿到答案再动手；破坏性操作前不要自己替用户选。

5. **备份两个文件，带时间戳**

   ```powershell
   $ts = Get-Date -Format "yyyyMMdd-HHmmss"
   Copy-Item "$p\package.json" "$p\package.json.bak-$ts"
   Copy-Item "$p\cordis.patch.yml" "$p\cordis.patch.yml.bak-$ts"
   "backup ts: $ts"
   ```

   同一目录里已有的旧备份不要覆盖或复用，每次生成新的时间戳。

6. **确认 scope 目录是不是符号链接**

   ```powershell
   Get-ChildItem "$p\node_modules\@linxin666" -Force | Select-Object Name,LinkType,LinkTarget | Format-Table -AutoSize
   ```

   是真实目录才可以直接整目录删；是符号链接/链接进 `.pnpm` 虚存时按链接实际指向处理，别盲删目标。

7. **改 `package.json`：两处同步删同一行**（用 edit 逐处改，别整文件重写）
   - `dependencies` 里删 `"@scope/xxx": "^x.y.z",`
   - `dsh.profile.bundles` 里删同名条目 `"@scope/xxx",`

   只删 dependencies 不删 bundles，启动时会找不到模块。

8. **`cordis.patch.yml`：把引用该模块的条目整段注释掉，保留原文**

   对每条 `- id: <...>` + `name: "@scope/xxx"` 的条目，逐行加 `#`（含其 `config:` 子块），并在整段前插入注释头写明原因与恢复步骤：

   ```yaml
   # 2026-11-07：@linxin666/dsh-web-all（Web UI 全家桶）已卸载，
   # 下面这些条目引用的插件包已不存在，整段注释掉，避免启动时找不到模块。
   # 要装回来：恢复 package.json 里的依赖与 bundles，并把这里的注释去掉。
   # - id: web-ui-remote-bridge
   #   disabled: true
   ```

   **不要直接删条目**：删掉会丢掉原 config 和回滚路径，而注释方式同样能解除「引用不存在的模块」这一强断言。

9. **删 `node_modules` 下的 scope 目录并验收**

   ```powershell
   $target = "$p\node_modules\@linxin666"
   $sz = (Get-ChildItem $target -Recurse -Force -File -ErrorAction SilentlyContinue | Measure-Object Length -Sum).Sum
   "size: {0:N1} MB" -f ($sz/1MB)
   Remove-Item -LiteralPath $target -Recurse -Force
   "exists after: $(Test-Path $target)"
   ```

   记下 MB 数，后面汇报「释放了多少」；`exists after` 必须是 `False`。只在该 scope 目录已被整个卸载包占据时才整目录删，同 scope 下还有别的在用包就只删对应子目录。

10. **残留引用扫描（预期只剩注释行）**

    ```powershell
    Select-String -Path "$p\*.json","$p\*.yml","$p\*.yaml" -Pattern "@scope/xxx" -ErrorAction SilentlyContinue | Select-Object Filename,LineNumber,Line | Format-Table -AutoSize -Wrap
    ```

11. **硬验收：同时解析 JSON 与 YAML，打印剩余依赖/bundles/条目数**

    ```powershell
    cd $p
    node -e "const fs=require('fs');const YAML=require('yaml');const j=JSON.parse(fs.readFileSync('package.json','utf8'));const y=YAML.parse(fs.readFileSync('cordis.patch.yml','utf8'));console.log('package.json ok, deps:',Object.keys(j.dependencies||{}).join(', '));console.log('bundles:',((j.dsh&&j.dsh.profile&&j.dsh.profile.bundles)||[]).join(', '));console.log('patch entries:',Array.isArray(y)?y.length:Object.keys(y).length)"
    ```

    必须在 profile 目录下跑，`require('yaml')` 才会解析到该 profile 的 `node_modules`。条目数应从改造前的值下降（实测 12 → 9）。

12. **汇报**：给出「位置 / 操作」两列改动表（package.json 两处、cordis.patch.yml 注释了哪几条 id、node_modules 删了什么+释放空间），附校验输出（剩余 bundles、条目数），并明确声明用户数据（会话、记忆等 `~/.dsh` 下其它目录、profile 里的 `.plugin-manager`/数据文件）**未被删除**，符合第 4 步用户所选项。

## Pitfalls

- **在 CLI 上反复试错**：`dsh` 不在 PATH 时 `dsh plugin ...` 只会报无法识别；探一次即判定手改路线。
- **只改一处**：`dependencies` 与 `dsh.profile.bundles` 必须同步删；漏删 bundles 启动即找不到模块。
- **删 patch 条目而不是注释**：直接删会丢失原 `config` 与恢复步骤，回滚就没有依据；整段注释并写明原因。
- **漏掉 patch 里引用子插件的条目**：包删了但 patch 仍 `name: "@scope/xxx"`，启动即崩；先读 patch 再动手。
- **不问就动手**：名字相近的包（`dsh-web-all` / `dsh-web-app` / `dsh-client-ui-*`）影响差别很大，卸错会让整套面板消失；用一次问答把「卸哪个 + 是否留数据」定死。
- **把「保留数据」理解成别删文件**：删 `node_modules\@scope` 是必须的；别去碰 `~/.dsh` 下的 `attachments`、`dsh-session-archive`、记忆目录。
- **备份缺失/覆盖**：目录里可能已有旧 `.bak-<数字>` 备份，用 `yyyyMMdd-HHmmss` 新建，别复用。
- **符号链接盲删**：先看 `LinkType`/`LinkTarget`，再决定删目录还是删链接。

## Verification

- `Test-Path $target` 返回 `False`，且报告了删除前的 MB 数。
- `node -e` 同时 `JSON.parse(package.json)` 与 `YAML.parse(cordis.patch.yml)` 成功，打印的 deps / bundles 里都不再含被卸包名，patch 条目数下降。
- `Select-String` 扫描结果里该包名只出现在第 8 步写入的注释行。
- `package.json.bak-<ts>` 与 `cordis.patch.yml.bak-<ts>` 存在，可据此回滚（恢复两个文件 + 删掉 patch 注释前缀）。
- 回复里明确说明用户数据未被删除。

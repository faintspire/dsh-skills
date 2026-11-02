---
name: ssh-remote-bash-via-base64
description: 从 Windows PowerShell 用 ssh 在远程 Linux 上稳定执行多行 bash：单引号 here-string 存脚本 → UTF8 Base64 编码 → 远端 `base64 -d | bash`，并为递归 grep 超时与远程无权限（docker.sock permission denied、sudo 需要 tty、接口 403）给出「限域重跑 / 记录限制转只读取证」的处置；当用户说 ssh 多行脚本报 unexpected EOF、syntax error near `done'、括号被吃掉，或远程 grep -r 跑了两分多钟没结果、取证时撞上 permission denied / 403 时使用。
whenToUse: 需要经 ssh 在远程 Linux 上跑带引号、括号、for/while 循环、管道的多行 bash；或远程递归 grep 迟迟不返回需要缩小范围重跑；或远程取证被权限挡住（docker.sock、sudo、Nacos/HTTP 403）需要决定是否继续。
---

# 用 Base64 把多行 bash 送到远程执行

本地是 PowerShell、远端是 Linux 时，`ssh "..."` 里的引号、括号、`$` 会被**本地** shell 先解析一遍，多行脚本几乎必然在远端变成 `unexpected EOF` / `syntax error near unexpected token 'done'`。确定可行的写法是：脚本原样写进单引号 here-string → UTF8 Base64 → 远端解码后交给 bash。

## Steps

1. **停止在 ssh 参数里拼脚本。** 只要脚本里出现引号、`(` `)`、`$`、`for … done` 或管道，就不要写成 `ssh host "for f in …; do …; done"`。

2. **把脚本写进单引号 here-string。** 用 `@'` 开头、`'@` 结尾；`@'` 之后不能跟任何字符，结尾 `'@` 必须顶格（前面不能有缩进）。单引号 here-string 内不做任何变量展开，`$JAR`、`"$f"`、`'…'`、`for` 循环都保持原样。

   ```powershell
   $s = @'
   set -eu
   JAR=/path/to/app.jar
   echo "=== config entries ==="
   unzip -l "$JAR" | grep -Ei 'BOOT-INF/classes/(application|bootstrap)' | head
   for f in application.yml application-prod.yml; do
     echo "########## $f ##########"
     unzip -p "$JAR" "BOOT-INF/classes/$f"
   done
   '@
   ```

3. **脚本开头加 `set -u`（或 `set -eu`）。** 用于让未定义变量、命令失败立即暴露，而不是静默跑完给出一堆空结果。

4. **用 UTF8 转 Base64。** 必须显式指定 `[Text.Encoding]::UTF8`，否则脚本里的中文/非 ASCII 会乱码：

   ```powershell
   $b = [Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes($s))
   ```

5. **在远端解码后交给 bash 执行。** Base64 只含 `A-Za-z0-9+/=`，可安全放在内置的单引号里；外层用双引号让 PowerShell 展开 `$b`：

   ```powershell
   ssh -o BatchMode=yes <user>@<host> "echo '$b' | base64 -d | bash"
   ```

   `-o BatchMode=yes` 用来保证认证失败时直接报错退出，而不是卡在密码提示上。脚本以 `echo "=== … ==="` 之类的分隔标记开头，方便把输出和本地日志区分开。

6. **递归 grep 一律限域。** 不要在 `/`、`/home` 或整个数据目录上直接 `grep -r`。固定用「指定子目录 + `--include` 限扩展名 + 跳过二进制 + `head`」：

   ```bash
   grep -rIn --include='*.yml' --include='*.yaml' --include='*.conf' --include='*.json' --include='*.properties' --include='*.sh' \
     -E '<pattern>' <子目录1> <子目录2> 2>/dev/null | head -20
   ```

   子目录只列这次真正需要的那几个；需要看全部文本文件时用 `grep -rIl … | grep -v '\.jar$'` 先拿文件清单，再逐个看。

7. **超时就终止，不要干等。** 命令跑满约 120s 被转后台时，直接 `job_kill(job_id=…, reason="Too slow, narrowing scope")`，然后用第 6 步的写法缩小范围重跑（实测由超时变为秒级返回命中清单）。

8. **远程无权限时立刻停止索取，改只读取证。** 遇到 `docker.sock: permission denied`、`sudo: a terminal is required`、接口返回 403（如 Nacos 配置接口 `{"status":403,"message":"user not found!"}`）时：不索要密码、不重复重试、不换路径反复撞，直接在结论里写明「该取证路径不可用 + 原因」，并把证据面切到只读渠道（`docker-compose.yml` 里 grep 端口/服务名、`unzip -l` / `unzip -p` 读 jar 内配置、`/proc/net/tcp` 看连接状态等）。

## Pitfalls

- **双引号内联脚本**：`ssh host "…"` 中 `(application|bootstrap)` 的括号被本地解析掉，远端报 `syntax error near unexpected token '('`。→ 用 here-string + Base64。
- **只换成单引号仍不够**：外层改成单引号后，脚本里的 `$f`、`"$JAR"` 又要靠反斜杠转义，`for … done` 依旧报 `syntax error near unexpected token 'done'`。→ 别再手工转义，走 Base64。
- **here-string 写坏**：`@'` 后面多一个空格、或结尾 `'@` 缩进，PowerShell 会直接语法报错/少收集内容。
- **忘记 UTF8**：用默认编码转 Base64，远端解码出的中文注释与中文路径变乱码。
- **把密码或 sudo 交互留在远程**：`BatchMode=yes` + 只读命令；需要提权的操作直接放弃并记录，别把凭据写进脚本。
- **全盘 grep 当默认手段**：`grep -rIl` 扫大目录会超时转后台，浪费时间且输出被截断。→ 先限子目录与扩展名。
- **超时后继续等**：转后台的命令往往还要跑很久，`head` 也截不住扫描本身。→ 直接 kill 再限域重跑。
- **403 / permission denied 反复重试**：同一路径重试不会变成有权限，只会消耗回合。→ 一次失败即记录限制并切只读证据。

## Verification

- 远端输出里出现脚本自己的 `=== … ===` 标记且无任何 `bash: -c: line 0:` / `unexpected EOF` / `syntax error` 行，退出码为 0，即写法生效。
- 首次在新主机上使用时，先跑一个最小脚本（`set -eu` + `echo ok` + 一个 `for` 循环）确认链路，再上真正的取证脚本。
- 限域 grep 应在秒级返回并给出 `文件:行号` 命中清单；若仍超时，说明子目录或 `--include` 还不够窄，继续收窄而不是等待。
- 收尾时确认：结论里已列出所有「因无权限/403 未能取证」的项与其原因，且没有靠猜测补空。

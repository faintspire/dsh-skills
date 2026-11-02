---
name: maven-multimodule-compile-verify-windows
description: >-
  在 Windows 下用 Maven 对本仓 biovisart-server 的 Java 多模块改动做编译验收，以 BUILD SUCCESS
  与 Reactor Summary 各模块 SUCCESS 判定结果；开跑前先探测本会话实际可用的 shell 工具（pwsh /
  win_bash / bash）再套同一条 mvn 命令，遇到 clean 阶段删不掉各模块 target 产物时不反复重试
  clean 而改为只跑 compile。当用户说编译验收、编译验证一下、帮我 mvn 编译、确认模块能编译、
  多模块改完帮我编译，或遇到 No plugin found for prefix、target 目录拒绝访问、Failed to clean
  project 时使用。
whenToUse: >-
  用户刚改完 biovisart-server 下 common/biz/ai/netty 的 Java 代码要求编译验收时；在 PowerShell 里跑
  mvn -pl ... compile 报错（No plugin found for prefix、~/.m2 拒绝访问、120s 超时）需要给出正确命令
  与判定标准时；mvn ... clean compile 报 Failed to clean project 加 Failed to delete 某模块 target
  产物，需要决定是重试 clean 还是跳过 clean 时；或调用 shell 工具直接拿到 unknown tool 需要换执行入口时。
version: 4
---

# Windows 下多模块 Maven 编译验收

固定环境事实（引自本仓实测）：

- 仓库根目录：`C:\Develop\project\biovisart-server`
- 本地 Maven 仓库：`C:/Develop/apache-maven-3.9.11/repository`（工作区外，必须显式指定）
- 模块：`biovisart-common`、`biovisart-ai`、`biovisart-netty`、`biovisart-biz`、`biovisart-server`
- **执行入口按会话探测，不要硬编码**：先确认本会话实际可用的 shell 工具（`pwsh` / `win_bash` / `bash`），把结果记下来全程复用。实测两种会话都出现过：一种只有 `win_bash`（参数 `command` / `workdir` / `timeoutMs`），另一种没有 `win_bash`、只有 `pwsh`（参数 `command` / `description`，命令里自带 `Set-Location` 与管道）。命令本体（Maven 参数、引号、正斜杠）与工具名无关，探测完套同一条命令即可。
- 对着本会话不存在的工具名调用只会拿到 `unknown tool "bash"` 这类报错，那不是构建失败，只是探测结论。
- 前台 `run_code` 有约 120s 默认执行 deadline；入口支持内联超时（`win_bash` 的 `timeoutMs`，如 400000）就先内联跑完，不必强制转后台。
- 会话文件策略为 workspace-write：模块 `target` 下的旧产物未必删得掉，`clean` 因此可能先于编译失败
- 审批/提权在本会话默认不可用；不要把「放宽文件权限」当成首选解法

## Steps

1. 先探测 shell 入口，再摸改动面，别凭印象决定编译范围。

   - 探测：看本会话的工具清单里有没有 `pwsh` / `win_bash` / `bash`。拿不准就发一条无害命令验证（例如 `echo noop` 或 `Write-Output ok`）；拿到 `unknown tool` 只说明该工具不存在，换下一个，**不要据此判定构建失败，也不要每步重新试错**。
   - 探测结果决定后续每条命令的形态：走 `win_bash` 用 `command` / `workdir` / `timeoutMs`，走 `pwsh` 用 `command` / `description` 并在命令里 `Set-Location` 到仓库根。
   - 摸改动面（在 `workdir: C:\Develop\project\biovisart-server`，或 pwsh 形态下先 `Set-Location`）：

     ```powershell
     git status --short
     ```

     用 `grep` 把改动文件路径映射到模块（`biovisart-common/`、`biovisart-ai/`、`biovisart-netty/`、`biovisart-biz/`）。改动落在哪几个模块，`-pl` 就覆盖哪几个。

2. 按改动面选口径（三种都实测可用，不要为了统一而收紧成一种）：

   - **单模块改动**：直接 `-pl <模块> -am`。`-am`（also-make）只把该模块**依赖的上游模块**一并加进 reactor，不会牵动无关模块；实测 `-pl biovisart-biz -am` 一次 `BUILD SUCCESS`。
   - **common + biz 同改**（上游 snapshot 已安装）：用显式逗号列表 `-pl biovisart-common,biovisart-biz compile`，**不加** `-am`，避免把已安装 snapshot 的上游模块重复构建、拉长耗时。
   - **下游模块组合**（如 `biovisart-biz` + `biovisart-netty`，或 `biovisart-ai` + `biovisart-biz`）且要连带校验上游：用显式逗号列表并补 `-am`，即 `-pl biovisart-biz,biovisart-netty -am compile`、`-pl biovisart-ai,biovisart-biz -am compile`（两条均实测通过，Reactor Build Order 里带上 common 等上游模块）。

   只有在想确认 snapshot 是否已安装（决定多模块场景要不要 `-am`）时才做这一步抽查：

   ```powershell
   $repo='C:\Develop\apache-maven-3.9.11\repository\com\sangerbox'
   if (Test-Path $repo) { Get-ChildItem -Recurse -Directory -Depth 1 $repo | ForEach-Object { $_.FullName.Replace($repo,'') } } else { "missing: $repo" }
   ```

3. 单模块编译（改动只在 `biovisart-biz` 时的实测通过版本，一次跑完约需 300s 级，故内联 `timeoutMs: 400000`）：

   ```js
   await tools.win_bash({
     command: 'mvn -B -DskipTests "-Dmaven.repo.local=C:/Develop/apache-maven-3.9.11/repository" -pl biovisart-biz -am compile',
     workdir: "C:\\Develop\\project\\biovisart-server",
     timeoutMs: 400000
   });
   ```

   硬性要点两条，缺一即失败：整个 `-Dmaven.repo.local=` 参数用双引号包住、路径用正斜杠。`-am` 与内联 `timeoutMs` 在单模块场景可放心同用。

4. 多模块编译（显式列表，模块顺序与 reactor 无关，Maven 自己排序）。

   `win_bash` 形态：

   ```js
   await tools.win_bash({
     command: 'mvn -B -DskipTests "-Dmaven.repo.local=C:/Develop/apache-maven-3.9.11/repository" -pl biovisart-biz,biovisart-netty -am compile',
     workdir: "C:\\Develop\\project\\biovisart-server",
     timeoutMs: 400000
   });
   ```

   `pwsh` 形态（没有 `win_bash` 的会话，本轮实测通过；命令里直接带 `Select-String` 抽摘要，避免整份日志灌进上下文）：

   ```powershell
   Set-Location 'C:\Develop\project\biovisart-server'
   mvn -B -DskipTests "-Dmaven.repo.local=C:/Develop/apache-maven-3.9.11/repository" -pl biovisart-ai,biovisart-biz -am compile 2>&1 | Select-String -Pattern 'BUILD|SUCCESS \[|FAILURE|ERROR' | Select-Object -Last 40
   ```

   对应工具调用参数是 `{ command, description }`，上面两行合成一条命令字符串（用 `;` 连接 `Set-Location` 与 `mvn` 也可以）。摘要里应能看到 `[INFO] Reactor Build Order:` 与各模块 `SUCCESS` 行。

   common + biz 且上游 snapshot 已安装时换成 `-pl biovisart-common,biovisart-biz compile`（不带 `-am`）；要覆盖四个模块就用 `-pl biovisart-common,biovisart-ai,biovisart-netty,biovisart-biz compile`。同样按探测到的入口传超时（`win_bash` 用内联 `timeoutMs`）。

5. 决定要不要 `clean`（本轮最重要的分歧点，最多判断一次）：

   - 想要「旧 class 不算通过」的严格验收、且 `target` 可写时，可以用 `clean compile`；此时日志里必须出现 `Compiling N source files with javac` 一类实际编译行，出现 `Nothing to compile` 就不算验收（产物是旧的）。
   - 一旦日志出现下面这条，说明挂在 `maven-clean-plugin` 上、编译根本没开始：

     ```text
     [ERROR] Failed to execute goal org.apache.maven.plugins:maven-clean-plugin:3.4.1:clean (default-clean) on project biovisart-common: Failed to clean project: Failed to delete C:\Develop\project\biovisart-server\biovisart-common\target\...
     ```

     **不要重试 clean，也不要先想着放宽文件权限** —— 实测放宽权限后重跑同一条 `clean compile` 仍围着 clean 打转。直接去掉 `clean`，改跑不带 clean 的命令，一次到位：

     ```powershell
     mvn -pl biovisart-biz,biovisart-netty -am compile 2>&1 | Select-String -Pattern 'BUILD|SUCCESS|FAILURE|ERROR' | Select-Object -Last 15
     ```

     对应两种入口形态（命令字符串直接带管道抽摘要）：

     ```js
     await tools.win_bash({
       command: 'mvn -B -DskipTests "-Dmaven.repo.local=C:/Develop/apache-maven-3.9.11/repository" -pl biovisart-biz,biovisart-netty -am compile 2>&1 | Select-String -Pattern "BUILD|SUCCESS|FAILURE|ERROR" | Select-Object -Last 15',
       workdir: "C:\\Develop\\project\\biovisart-server",
       timeoutMs: 400000
     });
     ```

     ```powershell
     # pwsh 入口：同一条 mvn 命令，前面补 Set-Location
     Set-Location 'C:\Develop\project\biovisart-server'
     mvn -B -DskipTests "-Dmaven.repo.local=C:/Develop/apache-maven-3.9.11/repository" -pl biovisart-biz,biovisart-netty -am compile 2>&1 | Select-String -Pattern 'BUILD|SUCCESS|FAILURE|ERROR' | Select-Object -Last 15
     ```

   - 只有在「不带 clean 的 compile 阶段本身」也报 `target` 拒绝访问、且当前会话确实允许提权时，才考虑放宽文件权限；本会话审批默认禁用，带 `sandbox_permissions` 的请求会被自动拒绝，先按无提权路径走。全程不要靠换构建入口（IDE 构建、`install`、单独 `javac`）绕开。

6. 超时处理：探测到的入口支持内联超时（`win_bash` 的 `timeoutMs: 400000` 已在单模块 `-am` 场景实测够用）就先顶在前台；**只有**命令仍被 `execution deadline reached` / 超时截断时，才改用后台 job：`run_in_background: true` 拿 `jobId`，之后用 `tools.job_output({ job_id })` 取 `j.text`。超时截断是运行器侧限制，本身不代表编译失败，不要据此判定 `BUILD FAILURE`。（TODO：`pwsh` 入口的超时参数名本轮未实测，按当前入口文档传；取不到就走后台 job。）

   ```js
   const t = j.text || "";
   const lines = t.split(/\r?\n/);
   lines.filter(l => /ERROR|BUILD FAILURE|SUCCESS|Reactor Summary|Could not|Cannot find|symbol|error:/i.test(l));
   ```

   每次只打印尾部（如 `t.slice(-2500)`），避免整份 45k 日志灌进上下文。

7. 判定：在日志里同时找到 `BUILD SUCCESS` 和 `Reactor Summary`，且该 Summary 中每个参与构建的模块都是 `SUCCESS [ xx s]`，才算验收通过。把 Summary 行原样贴进结论，并说明 reactor 里实际包含哪些模块（用了 `-am` 时要指出额外带上的依赖模块）。

8. 只有 `exit=0` 不够，只有「生成 class」也不够（实测 `exit=1` 时日志里已有 `Compiling 56/19/10 source files with javac`）。若 `BUILD FAILURE`，抓 `[ERROR]` 首行定位原因：若是 clean 删不掉产物就按第 5 步去掉 clean 重跑；若是编译错误就改代码后重跑同一条命令，不要改用别的构建入口绕开。

9. 收尾复核也走同一个探测到的入口。`pwsh` 形态下实测可用：

   ```powershell
   Set-Location 'C:\Develop\project\biovisart-server'; git status --short; git diff --stat -- biovisart-ai biovisart-biz/src/main/java
   ```

   需要清掉 `target/classes` 下的陈旧 class（例如已删除的 Feign 接口残留产物）时用 `Remove-Item -LiteralPath <path> -Force`，先 `Test-Path` 判断存在与否再删，别直接 `clean`。

## Pitfalls

- **把执行入口写死成单一工具就会第一步失败。** 实测同一台机器上，一份会话只有 `win_bash`，另一份会话没有 `win_bash`、调用 `bash` 直接返回 `unknown tool "bash"`，全程改用 `pwsh` 才跑通（`mvn ... -pl biovisart-ai,biovisart-biz -am compile` 与 `git status` / `git diff --stat` 都是 pwsh 跑成的）。正确做法是先探测可用 shell 工具并复用探测结果，而不是凭记忆或上一轮经验假定工具名；`unknown tool` 是入口问题，不是构建结果。
- **`clean` 删不掉 target 就要立刻放弃 clean。** 实测 `mvn -q -pl biovisart-biz,biovisart-netty -am clean compile` 在 `maven-clean-plugin:3.4.1:clean` 上直接中断（`Failed to clean project: Failed to delete ...\biovisart-common\target\...`），编译一步都没走到；随后放宽文件权限重跑同一条 `clean compile` 仍是围着 clean 转，最后改成不带 clean 的 `-pl ... -am compile` 才拿到 `BUILD SUCCESS`。多模块 target 上的 clean 失败按固定处置：**一次不重试，去掉 clean 只跑 compile**。
- **默认 `~/.m2` 会被沙箱拒写。** 不带 `-Dmaven.repo.local`（或参数没生效）时出现 `java.io.FileNotFoundException: C:\Users\<user>\.m2\repository\...\resolver-status.properties（拒绝访问）`，日志里可能已 `Scanning for projects...` 走了一段。看到这条直接把本地仓库指到工作区外的 `C:/Develop/apache-maven-3.9.11/repository`。
- **未加引号的反斜杠路径被 PowerShell 拆开。** `-Dmaven.repo.local=C:\Develop\...` 会报 `No plugin found for prefix '.repo.local=C' in the current project and in the plugin groups [org.apache.maven.plugins, org.codehaus.mojo]`。根因不是缺插件，而是参数被按 `\` / `:` 拆解后当成插件前缀；修正是给整个 `-D` 参数加双引号并把路径换成正斜杠。换 shell 入口不改这条要求。
- **`-am` 不是禁忌，只是会扩大 reactor。** 它只把目标模块的依赖模块一起构建，不会牵动无关模块，但模块数变多、耗时变长；单模块场景用 `-pl <模块> -am`，`-pl biovisart-biz,biovisart-netty`、`-pl biovisart-ai,biovisart-biz` 这类组合需要连带校验上游时也补 `-am`，而 `biovisart-common,biovisart-biz`（上游 snapshot 已安装）用显式列表不带 `-am`。判断 reactor 里到底进了哪些模块，看 Reactor Summary，不要凭经验断言。
- **不要靠提权/审批解决拒写。** 本会话审批被禁用，带 `sandbox_permissions` 的请求会被自动拒绝；只有「引号 + 正斜杠」这条路径实测可用。遇到 clean 删不掉时，正确修正是去掉 clean，而不是提权重试。
- **`code run failed (abort): sandbox mode changed` 不是构建失败。** 这是运行器侧的 abort，构建没开始或被打断；原样重试同一条命令（必要时再放后台），并在最终结论里说明该 abort 与构建结果无关。
- **不要把编译成功的中间模块当成整体通过。** 多模块 reactor 中前面模块编译出 class、后面模块失败，整体仍是 `BUILD FAILURE`；只认 Summary。

## Verification

- 日志含 `[INFO] BUILD SUCCESS`。
- 日志含 `[INFO] Reactor Summary for ...`，且 reactor 里每个模块都是 `SUCCESS [ ... s]`，没有 `FAILURE`（意外 `SKIPPED` 要说明原因）。
- 若中途出现过 `Failed to clean project: Failed to delete ...target...`，最终那次命令字符串里不含 `clean`，并且摘要里仍有 `BUILD SUCCESS` 与各模块 `SUCCESS`；结论里说明「clean 跳过是因为沙箱下 target 删不掉，编译本身通过」。
- 命令字符串自查：`-Dmaven.repo.local=` 前有 `"`、路径是 `C:/...`、不出现 `C:\` 形式的仓库路径；`-pl` 是显式逗号列表，单模块与下游组合带 `-am`，common + biz 不带 `-am`。
- 执行入口自查：本轮先探测了可用 shell 工具（`pwsh` / `win_bash` / `bash`）并在后续步骤复用同一入口，没有对着不存在的工具名反复调用；走 `win_bash` 时带 `command` / `workdir` / `timeoutMs` 三个参数，走 `pwsh` 时带 `command` / `description` 且命令里 `Set-Location` 到仓库根。
- 结论里写明验收依据是 BUILD SUCCESS + Reactor Summary，而不是 exit code；若有 abort/timeout/clean 重试/入口切换，注明处置动作与最终那次的结果。

---
name: maven-multimodule-compile-verify-windows
description: 在 Windows 下对本仓 Java 多模块改动做 Maven 编译验收：用 tools.win_bash 执行 mvn，-Dmaven.repo.local 用引号包裹的正斜杠路径，单模块改动可直接 `-pl <模块> -am` 并配内联 timeoutMs，多模块改动用显式逗号 -pl 列表，最终以 BUILD SUCCESS 与 Reactor Summary 判定而非 exit code。当用户说"编译验收""编译验证一下""帮我 mvn 编译""确认模块能编译"，或遇到 No plugin found for prefix 需要修正命令时使用。
whenToUse: 用户刚改完 biovisart-server 下 common/biz/ai/netty 的 Java 代码要求编译验收，或在 PowerShell 里跑 mvn -pl ... compile 报错（No plugin found for prefix、~/.m2 拒绝访问、120s 超时）需要给出正确命令与判定标准时。
version: 2
---

# Windows 下多模块 Maven 编译验收

固定环境事实（引自本仓实测）：

- 仓库根目录：`C:\Develop\project\biovisart-server`
- 本地 Maven 仓库：`C:/Develop/apache-maven-3.9.11/repository`（工作区外，必须显式指定）
- 模块：`biovisart-common`、`biovisart-ai`、`biovisart-netty`、`biovisart-biz`、`biovisart-server`
- 执行入口是 `tools.win_bash({ command, workdir, timeoutMs })`（本会话没有 `tools.pwsh`，别调用它）
- 前台 `run_code` 有约 120s 默认执行 deadline；把 `timeoutMs` 显式传大（如 400000）即可在前台一次跑完，不必强制转后台
- 审批/提权在本会话默认不可用

## Steps

1. 先摸改动面，别凭印象决定编译范围。在 `workdir: C:\Develop\project\biovisart-server` 跑：

   ```powershell
   git status --short
   ```

   再用 `grep` 把改动文件路径映射到模块（`biovisart-common/`、`biovisart-ai/`、`biovisart-netty/`、`biovisart-biz/`）。改动落在哪几个模块，`-pl` 就覆盖哪几个。

2. 按改动面选口径（两种都实测可用，不要为了统一而收紧成一种）：

   - **单模块改动**：直接 `-pl <模块> -am`。`-am`（also-make）只把该模块**依赖的上游模块**一并加进 reactor，不会牵动无关模块；本轮实测 `-pl biovisart-biz -am` 一次 `BUILD SUCCESS`。
   - **多模块改动**（如 common + biz）：用显式逗号列表 `-pl biovisart-common,biovisart-biz compile`，**不加** `-am`，避免把已安装 snapshot 的上游模块重复构建、拉长耗时。需要连带校验全部上游时才补 `-am`。

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

4. 多模块编译（改动在 common + biz 时的实测通过版本，显式列表、不带 `-am`）：

   ```js
   await tools.win_bash({
     command: 'mvn -B -DskipTests "-Dmaven.repo.local=C:/Develop/apache-maven-3.9.11/repository" -pl biovisart-common,biovisart-biz compile',
     workdir: "C:\\Develop\\project\\biovisart-server",
     timeoutMs: 400000
   });
   ```

   要覆盖四个模块就把 `-pl` 换成 `biovisart-common,biovisart-ai,biovisart-netty,biovisart-biz`（仍不带 `-am`，同样用内联 `timeoutMs`）。

5. 超时处理：内联 `timeoutMs` 先顶在前台（400000 已在单模块 `-am` 场景实测够用）；**只有**命令仍被 `execution deadline reached` / 超时截断时，才改用后台 job：`run_in_background: true` 拿 `jobId`，之后用 `tools.job_output({ job_id })` 取 `j.text`。超时截断是运行器侧限制，本身不代表编译失败，不要据此判定 `BUILD FAILURE`。

   ```js
   const t = j.text || "";
   const lines = t.split(/\r?\n/);
   lines.filter(l => /ERROR|BUILD FAILURE|SUCCESS|Reactor Summary|Could not|Cannot find|symbol|error:/i.test(l));
   ```

   每次只打印尾部（如 `t.slice(-2500)`），避免整份 45k 日志灌进上下文。

6. 判定：在日志里同时找到 `BUILD SUCCESS` 和 `Reactor Summary`，且该 Summary 中每个参与构建的模块都是 `SUCCESS [ xx s]`，才算验收通过。把 Summary 行原样贴进结论，并说明 reactor 里实际包含哪些模块（用了 `-am` 时要指出额外带上的依赖模块）。

7. 只有 `exit=0` 不够，只有「生成 class」也不够（实测 `exit=1` 时日志里已有 `Compiling 56/19/10 source files with javac`）。若 `BUILD FAILURE`，抓 `[ERROR]` 首行定位原因，改参数后重跑同一命令，不要改用别的构建入口绕开。

## Pitfalls

- **默认 `~/.m2` 会被沙箱拒写。** 不带 `-Dmaven.repo.local`（或参数没生效）时出现 `java.io.FileNotFoundException: C:\Users\<user>\.m2\repository\...\resolver-status.properties（拒绝访问）`，日志里可能已 `Scanning for projects...` 走了一段。看到这条直接把本地仓库指到工作区外的 `C:/Develop/apache-maven-3.9.11/repository`。
- **未加引号的反斜杠路径被 PowerShell 拆开。** `-Dmaven.repo.local=C:\Develop\...` 会报 `No plugin found for prefix '.repo.local=C' in the current project and in the plugin groups [org.apache.maven.plugins, org.codehaus.mojo]`。根因不是缺插件，而是参数被按 `\` / `:` 拆解后当成插件前缀；修正是给整个 `-D` 参数加双引号并把路径换成正斜杠。
- **`-am` 不是禁忌，只是会扩大 reactor。** 它只把目标模块的依赖模块一起构建，不会牵动无关模块，但模块数变多、耗时变长；所以单模块场景用 `-pl <模块> -am` + 内联 `timeoutMs: 400000`，多模块场景仍优先显式逗号列表且不带 `-am`。判断 reactor 里到底进了哪些模块，看 Reactor Summary，不要凭经验断言。
- **不要靠提权/审批解决拒写。** 本会话审批被禁用，带 `sandbox_permissions` 的请求会被自动拒绝；只有「引号 + 正斜杠」这条路径实测可用。
- **`code run failed (abort): sandbox mode changed` 不是构建失败。** 这是运行器侧的 abort，构建没开始或被打断；原样重试同一条命令（必要时再放后台），并在最终结论里说明该 abort 与构建结果无关。
- **不要把编译成功的中间模块当成整体通过。** 多模块 reactor 中前面模块编译出 class、后面模块失败，整体仍是 `BUILD FAILURE`；只认 Summary。

## Verification

- 日志含 `[INFO] BUILD SUCCESS`。
- 日志含 `[INFO] Reactor Summary for ...`，且 reactor 里每个模块都是 `SUCCESS [ ... s]`，没有 `FAILURE`（意外 `SKIPPED` 要说明原因）。
- 命令字符串自查：`-Dmaven.repo.local=` 前有 `"`、路径是 `C:/...`、不出现 `C:\` 形式的仓库路径；单模块场景是 `-pl <模块> -am`，多模块场景是逗号显式列表且不带 `-am`。
- 执行入口是 `tools.win_bash` 且带 `command` / `workdir` / `timeoutMs` 三个参数，未调用 `tools.pwsh`。
- 结论里写明验收依据是 BUILD SUCCESS + Reactor Summary，而不是 exit code；若有 abort/timeout 重试，注明重试次数与最终那次的结果。

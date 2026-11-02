---
name: java-dependency-api-forensics-offline
description: 在没有 javap 的 Windows 机器上离线取证第三方 Java 依赖的真实 API：从本地 Maven 仓库定位 <artifact>-<version>-sources.jar、解压后直接读源码，用 Select-String 取真实方法/字段签名并给出「文件:行号 + 原文行」的证据；当用户说 javap 用不了、没有 javap、离线确认某个依赖到底有哪些方法/字段、不要装反编译工具时使用。
whenToUse: 用户要在 Windows 上确认某个第三方 Java 库（如 langchain4j）某个版本的类里有哪些字段/方法/builder 校验，而这台机器上 javap 报「无法将"javap"项识别为 cmdlet」、或用户明确说「不装反编译工具」「只给源码位置和行号的证据」时使用。不用于编译验收（见 maven-multimodule-compile-verify-windows），也不用于改造本地源码。
---

# 无 javap 时离线取证第三方 Java 库真实 API

目标：不装 JDK、不装 decompiler、不改仓库任何文件，只用本地 Maven 仓库里的 `-sources.jar` 拿到真实签名，并且每条结论都能被「源码文件:行号 + 原文行」复核。

前置约束（先复述给用户）：本轮只读，零代码改动；所有解压产物只写到 `$env:TEMP`。

## Steps

1. 先定位本地 Maven 仓库根目录。默认是 `$env:USERPROFILE\.m2\repository`；本机若自定义过，以 `settings.xml` 的 `<localRepository>` 为准（案例机器为 `C:/Develop/apache-maven-3.9.11/repository`）。用 groupId 的点号换成路径、再接 artifactId 父目录：

   ```powershell
   $repo = 'C:/Develop/apache-maven-3.9.11/repository'   # 本机实测值，其他机器按上面的规则替换
   $ga = "$repo/dev/langchain4j"                          # dev.langchain4j -> dev/langchain4j
   Get-ChildItem -LiteralPath $ga -Directory | Select-Object -ExpandProperty Name
   ```

2. 列出该 group 下目标构件与版本目录，确认版本号（案例为 `1.18.1`）：

   ```powershell
   Get-ChildItem -LiteralPath "$ga/langchain4j-core" -Directory | Select-Object -ExpandProperty Name
   Get-ChildItem -LiteralPath "$ga/langchain4j-core/1.18.1" -Filter '*.jar' | Select-Object Name, Length
   ```

   若项目中已有依赖声明，用 `Select-String -Path 'pom.xml' -Pattern 'langchain4j' -Context 0,2` 交叉确认版本；两者不一致时以项目声明的版本为准并在结论里写明。

3. 确认二进制 jar 与 sources jar 都在，路径写进后续变量：

   ```powershell
   $ver = '1.18.1'
   $bin = Join-Path "$ga/langchain4j-core/$ver" "langchain4j-core-$ver.jar"
   $src = Join-Path "$ga/langchain4j-core/$ver" "langchain4j-core-$ver-sources.jar"
   Test-Path -LiteralPath $bin; Test-Path -LiteralPath $src
   ```

   两者都要为 `True`；`-sources.jar` 不存在就停在这一步报告「该构件未发布 sources」，不要转去反编译。

4. javap 最多试一次，并且把环境判定一起做完（一次失败即切换，不重复试参数）：

   ```powershell
   javap -classpath $bin 'dev.langchain4j.model.chat.request.ChatRequest$Builder' 2>&1 | Select-String -Pattern 'tool'
   "JAVA_HOME=$env:JAVA_HOME"
   Get-ChildItem -LiteralPath $env:JAVA_HOME -ErrorAction SilentlyContinue | Select-Object -ExpandProperty Name
   ```

   判据：失败信息形如 `javap : 无法将"javap"项识别为 cmdlet、函数、脚本文件或可运行程序的名称`；且 `JAVA_HOME` 指向 `...\Common Files\Oracle\Java\javapath\java.exe`、目录里只有 `java.exe/javaw.exe/javaws.exe` 这类转发器——说明它不是 JDK 主目录，`javap` 不会因此出现。到此为止：不找 JDK、不装 decompiler、不再重试 javap。

5. 解开 sources jar 到临时目录（先清旧目录，避免读到上一次的解压结果）：

   ```powershell
   $out = "$env:TEMP\lc4j-src"
   if (Test-Path -LiteralPath $out) { Remove-Item -LiteralPath $out -Recurse -Force }
   Expand-Archive -LiteralPath $src -DestinationPath $out -Force
   Get-ChildItem -LiteralPath $out -Recurse -Filter 'ChatRequest.java' | Select-Object -ExpandProperty FullName
   ```

   若 `Expand-Archive` 拒绝非 `.zip` 扩展名，改用同一机制的 .NET 调用：`Add-Type -AssemblyName System.IO.Compression.FileSystem; [System.IO.Compression.ZipFile]::ExtractToDirectory($src, $out)`。

6. 用 `Select-String -Path` 在解压出的源码里逐条取真实成员（这类包都是包路径 = 文件路径）：

   ```powershell
   Select-String -Path "$out/dev/langchain4j/model/chat/request/ChatRequest.java" -Pattern 'public .*(toolChoice|toolSpecifications|messages|temperature|maxOutputTokens)'
   Select-String -Path "$out/dev/langchain4j/data/message/AiMessage.java" -Pattern 'public .*toolExecutionRequests|public boolean hasToolExecutionRequests|public static AiMessage from'
   ```

   注意嵌套类：`ChatRequest$Builder` 在 `ChatRequest.java` 内部，源码里没有单独的 `Builder.java`，别按文件名去找。

7. 关键行为（builder 校验、默认值、判空）要读全文件或带上下文读，不能只看命中的那一行：

   ```powershell
   Select-String -Path "$out/dev/langchain4j/model/chat/request/ChatRequest.java" -Pattern 'builder\.' -Context 3,3
   ```

8. 按「每条结论 = 一条证据」输出：`<类文件>.java:<行号>: <原文行>`，例如 `ChatRequest.java:32: || builder.toolChoice != null`、`AiMessage.java:58: public AiMessage(List<ToolExecutionRequest> toolExecutionRequests) {`。行号与原文直接取自 `Select-String` 的输出，不手抄、不补写。

## Pitfalls

- **不要把 `JAVA_HOME` 当 JDK 存在性的证据。** 本案例 `JAVA_HOME=C:\Program Files\Common Files\Oracle\Java\javapath\java.exe`，那只是只有 `java.exe` 转发器的目录；据此判断「有 JDK、再试一次 javap」是本技能里最容易踩的坑。
- **javap 失败后不要升级手段。** 不 `where.exe javap` 递归、不扫盘找 JDK、不装 decompiler、不换 `-p` / `-c` 等参数反复重试；一次失败即走 sources jar。
- **不许凭记忆写 API。** 没在解压源码里出现的成员，结论里要写「源码中不存在」，而不是用印象补；也不要用二进制 jar 的字符串猜测签名。
- **`Select-String` 只返回命中行，多行签名会被截断。** 参数换行的方法会只看到第一行，用 `-Context 0,6` 或读全文件补全括号内的参数列表。
- **区分「读到的」与「推断的」。** 例如 `AiMessage` 是否总有 `toolExecutionRequests`、是否 `hasToolExecutionRequests()` 才代表可执行工具调用，必须由源码行决定，推断部分单独标注。
- **只写 `$env:TEMP`。** 解压目录、临时 zip 一律在临时目录；仓库里不新增、不修改任何文件（包括不往 `src` 里拷源码）。

## Verification

1. 每条成员结论都有 `<Xxx.java>:<行号>: <原文本行>` 三件套，且同一行号在文件里能再被 `Select-String` 命中一次（可复跑）。
2. 证据用的版本号与 `Test-Path` 通过的 `-sources.jar` 文件名一致（案例 `langchain4j-core-1.18.1-sources.jar`），并与项目 pom 声明一致或已说明差异。
3. 交付里明确写了 javap 不可用的原因（命令不存在 / `JAVA_HOME` 只是 javapath），没有出现「再试一次 javap」「已安装 decompiler」这类动作。
4. 在仓库根目录跑 `git status --porcelain`，输出为空（本轮零改动）；解压产物只出现在 `$env:TEMP\<prefix>-src`。

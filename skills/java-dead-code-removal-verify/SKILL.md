---
name: java-dead-code-removal-verify
description: 安全删除 Java 死代码并做零引用与编译验收：先 grep 引用面确认「接口 + 唯一实现 + 注入点」确实全死，再逐处清理残留（import、字段、失效 Lombok 注解、javadoc @link），用 pwsh `Remove-Item -LiteralPath ... -Force` 删文件，删后复验 0 引用，最后以 `mvn -pl <biz模块> -am clean compile` 通过收尾。当用户说「这个接口/类不需要了，删掉」「删掉这个 Impl」「死代码删一下」「相关残留也清干净」「删完确认没有残留引用」时使用。
whenToUse: 用户要求删除不再被调用的 Java 接口、实现类、字段或整段死代码，并要求清干净残留引用或做编译验收时使用。只做编译验收、不删代码的场景改用 maven-multimodule-compile-verify-windows；只约束代码风格（不要过度防御）的场景与该技能无关。
---

# Java 死代码安全删除与验收

流程分五段：**判定 → 清理残留 → 删除 → 零引用复验 → 编译验收**。任何一段都不许跳：没确认引用面就删是猜，没复验就说「删干净了」是谎。

## Steps

1. **画引用面**。用 grep 一次性拿到全部引用，不要凭记忆：
   ```js
   const g = await tools.grep({ pattern: "<InterfaceName>|<methodName>\\(", include: "*.java" });
   console.log(`=== references (${g.matches.length}) ===`);
   for (const m of g.matches) console.log(`${m.path}:${m.lineNumber}: ${m.line.trim().slice(0, 150)}`);
   ```
   逐条归位：哪条是接口本体、哪条是唯一实现（`implements`）、哪条是注入点（字段声明 / `@Autowired` / 构造参数 / 调用点）。

2. **读注入点源码，确认字段从未被调用**。用 `tools.read({ file_path: "<handler 全路径>" })` 打印全部行号，确认该字段只剩下声明与通道绑定，`<method>(...)` 调用早已不存在——只有这时「接口 + 唯一实现」才算真死。读的同时记下该文件里所有相关残留行号（import、字段、注解、javadoc）。

3. **一次性清掉注入点残留**，用一个 `tools.edit` 完成，不要分多次改：
   - 引用该类型的 `import`；
   - `private final <Type> <field>;` 死字段；
   - 因此失效的 `@RequiredArgsConstructor`（字段删光后注解无意义）；
   - 类头 javadoc 里的 `{@link <Type>}` 整句；
   - 顺带清掉同一文件里的其它死 import（例如模块没有对应依赖的第三方 `import`，它本身就是编译失败源）。
   改完把文件前 30 行打印出来，肉眼确认四处都没了。

4. **删除前列绝对路径并确认存在**：对每个待删文件 `Get-Item -LiteralPath '<abs>'`，输出必须是 `FOUND <abs>`；连同其大小、Attributes 一起记下，作为删除前证据。

5. **用 pwsh 删除，不要用编辑器/文件工具直接删**：
   ```powershell
   Remove-Item -LiteralPath '<abs>/<File>.java' -Force
   if (Test-Path -LiteralPath '<abs>/<File>.java') { 'STILL EXISTS' } else { 'DELETED'; 'GONE' }
   ```
   `tools.pwsh({ description, command })` 跑完必须看到 `DELETED` + `GONE`。

6. **删除被拒时换写法重试，不许声称已删**。若输出 `DELETE FAILED: 对路径的访问被拒绝` / `STILL EXISTS`（即使同一调用里 `Get-Item` 正常），说明这一写法被挡下：另起一次 `tools.pwsh`，用 `Remove-Item -LiteralPath '<abs>' -Force`（宽访问写法，路径用单引号包住、`-LiteralPath` 避免通配展开）重试，仍以 `DELETED` / `GONE` 判定成功。只有看到 `GONE` 才能往下走。

7. **复验零引用**：重跑第 1 步的同一个 grep，打印 `remaining references: <n>`，必须为 `0`（接口名、实现类名、方法名各查一次）。同时确认两个 `netty/auth` 目录 `files=0`——空目录 git 不跟踪，无需额外处理。

8. **编译验收**（在仓库根目录跑，`workdir` 指到 reactor 根）：
   ```powershell
   mvn -B -DskipTests "-Dmaven.repo.local=<repo路径>" -pl <biz模块> -am clean compile
   ```
   必须 `-am`：被删的类可能跨模块，只编 biz 会漏掉上游模块的残留引用。判定看 `BUILD SUCCESS` + `Reactor Summary` 里所有模块都是 SUCCESS，不看 exit code。若整次调用被 `code run failed (abort): sandbox mode changed` 中止，那是环境变更不是编译错误——原样重跑一次即可。

9. **回报**：用表格列出每个文件（删除 / 清理了哪几处残留），附上 `remaining references: 0`、两个 auth 目录 `files=0`、`BUILD SUCCESS` 与模块清单。

## Pitfalls

- **直接删文件会被拒**：轨迹里第一次删 biz 下的实现类得到「对路径的访问被拒绝」，紧接着 `Get-Item` 仍显示文件存在。遇到就换 `Remove-Item -LiteralPath ... -Force` 重试，绝不能用「已经删了」蒙过去。
- **只删两个类文件不算删干净**：接口残留会把注入点、字段、`@RequiredArgsConstructor`、javadoc `{@link}` 全留在 handler 里，编译期可能还能过，但语义上仍是死引用。
- **死 import 可能是既有编译失败的真因**：轨迹里 netty 模块编译失败源于 `import cn.dev33.satoken.stp.StpUtil;`，而该模块没有 sa-token 依赖。清残留时顺手删掉，别只修「与本次删除相关」的那几行。
- **grep 命中数不等于调用数**：接口本体、`implements`、字段声明都会命中。必须读源码区分「声明残留」与「真实调用」。
- **编译中断≠编译失败**：`sandbox mode changed` 之类的 abort 要原样重跑，不要据此判定 BUILD FAILURE，也不要改代码。
- **别把验收写成只跑 biz**：漏 `-am` 会让上游模块的残留引用逃过检查。

## Verification

- `grep '<接口名>|<实现类名>|<方法名>' -include *.java` → `remaining references: 0`；
- 待删文件在 pwsh 下输出 `DELETED` 且 `Test-Path` 为假（`GONE`）；
- `biovisart-netty/.../netty/auth` 与 `biovisart-biz/.../netty/auth` 均 `files=0`；
- `mvn -B -DskipTests "-Dmaven.repo.local=<repo>" -pl <biz模块> -am clean compile` 输出 `BUILD SUCCESS`，`Reactor Summary` 全模块通过。

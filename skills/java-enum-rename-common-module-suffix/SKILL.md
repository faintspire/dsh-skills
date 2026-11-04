---
name: java-enum-rename-common-module-suffix
description: 在 Maven 多模块仓里把一个 Java 枚举改名并下沉到 common 模块的 enums 包（类名统一加 Enum 后缀），同步 *.java 引用与 *.md/*.sql 注释，再用旧名负向前瞻残留扫描、新名 .class 就位和多模块 mvn compile 退出码验收；当用户说「枚举统一放 common」「以 Enum 为后缀」「这个枚举改名/下沉 common」「全仓引用和文档 SQL 一起改」「改完保证多模块编译通过」时使用。
whenToUse: 用户下达长期命名约定（枚举统一放 common、类名统一加 Enum 后缀）并要求一次性落地到某个具体枚举，或要求在保留 git 重命名历史的前提下把类型跨包下沉并同步全仓引用、中文文档与 SQL 注释。不适用于拆模块/搬整目录/改 pom/删死代码。
---

# 枚举改名 + 下沉 common 模块 + 统一 Enum 后缀

一次可复跑的重命名流水线：先取证引用面与落位现状，用 `git mv` 一次完成改名和跨包下沉，再同步 `*.java` 引用与 `*.md`/`*.sql` 注释，最后以「旧名负向前瞻 0 命中 + 新名 `.class` 就位 + 编译 EXIT=0」验收。不动 pom 结构、不拆模块、不删目录——这里没有死代码，全部是活引用改名。

## 前置：先确认目标仓与可用 shell

1. 用 `pwd` 确认当前会话工作区；目标仓很可能**不是**会话工作区（证据里会话在 `biovisart-server`，实际要改的是 `far-manu-server`）。之后每条命令都以绝对路径 `cd` 进目标仓再执行。
2. 探测 shell：Windows 下用 `pwsh`。`win_bash` 会直接报 `unknown tool "win_bash"`——不要重试、不要换着猜工具名，记录一次后全程固定 `pwsh`。
3. 内置的 `grep`/搜索工具只能看到会话工作区，在目标仓里会返回 `No matches found` 的假阴性；**判定引用面一律以目标仓里的 `pwsh` 命令输出为准**。

## Steps

### 1. 取目标仓状态与引用面（含拼写变体）

```powershell
cd C:\path\to\target-repo
git status --short
git grep -n -E "[Pp]re[Cc]heck" -- "*.java" "*.md" "*.sql"
```

> 示例中的 `PrecheckServiceId` / `Precheck` / `PreCheck` 是按用户给的类名替换的占位；整条流水线换成实际类名，正则变体按真实拼写扩展。

若 `git grep` 不可用，退回 `Get-ChildItem` + `Select-String`，必须排除 `target`、`.git`、`.idea`：

```powershell
Get-ChildItem -Recurse -File | Where-Object { $_.FullName -notmatch '\\(target|\.git|\.idea)\\' -and $_.Extension -in '.java','.md','.sql' } | Select-String -Pattern 'PrecheckServiceId' | ForEach-Object { "$($_.Path.Replace((Get-Location).Path + '\','')):$($_.LineNumber)  $($_.Line.Trim())" }
```

### 2. 判断落位与后缀现状

```powershell
"=== 枚举目录 ==="; Get-ChildItem -Recurse -Directory -Filter enums | ForEach-Object { $_.FullName }
"=== common enums ==="; Get-ChildItem far-manu-common\src\main\java\com\sangerbox\common\enums -File -ErrorAction SilentlyContinue | Select-Object Name
```

用 `read` 打开目标枚举（限 20 行足够看 package 与类声明）以及 1–2 个同类枚举，确认三件事：是否已在 common、package 与目录是否一致、是否只缺 `Enum` 后缀。别在没看文件前就假定它需要搬包。

### 3. 清点全仓命中并留清单

把命中按类型分组——类型引用/import、`*.md` 说明行、`*.sql` 注释行（例如 `service_id 对应 PrecheckServiceId.raw`）。这份清单就是后面「同步了哪些类与文档」的汇报底稿。

### 4. 用 `git mv` 一次完成改名 + 跨包下沉

```powershell
git mv far-manu-common\src\main\java\com\sangerbox\common\enums\PrecheckServiceId.java far-manu-common\src\main\java\com\sangerbox\common\enums\PrecheckServiceIdEnum.java
git mv far-manu-business\src\main\java\com\sangerbox\business\integration\model\PrecheckParentStatus.java far-manu-common\src\main\java\com\sangerbox\common\enums\PrecheckParentStatus.java
git status --short   # 期望看到 R（renamed），不是 A + D
```

改名与下沉要**在同一条 `git mv` 里完成**，不要先复制再删除、也不要 `mv` 后补 `git add`——历史断掉就无法在评审里看到 rename。多个枚举一次改完，各自一行。

### 5. 修正被移动文件的 package 声明

先 `read` 再 `edit`。顺序反了会被直接拒绝：`cannot modify "...": file has not been read`。对每个搬包的文件把 `package com.sangerbox.business.integration.model;` 改成实际所在包（本例 `package com.sangerbox.common.enums;`），并检查文件内自引用/静态导入是否指向旧包。

### 6. 同步 `*.java` 引用

对步骤 1/3 清单里的每个 java 文件：`read` 命中区段 → `edit` 替换 import 与类型引用。特别注意：

- import 里改包名的（被下沉的类型），不只改类名。
- 与枚举常量比较的 switch/`valueOf` 字符串字面量以外的普通常量引用，只改类型名，不改常量名。
- 改完再看一遍是否漏掉同文件多处的第二、第三处引用。

### 7. 同步 `*.md` 与 `*.sql`（按字节读写，保留 BOM/编码）

中文文档整文件重写会制造巨大噪音 diff。必须按字节读、检测 BOM、用负向前瞻替换、按原编码写回，并打印 bom 标记：

```powershell
$files = @('docs\支付设计方案.md','docs\项目信息文档.md','sql\init.sql','sql\pay_init.sql')
foreach ($f in $files) {
  $p = Join-Path (Get-Location) $f
  if (-not (Test-Path -LiteralPath $p)) { "MISSING $f"; continue }
  $bytes = [IO.File]::ReadAllBytes($p)
  $hasBom = $bytes.Length -ge 3 -and $bytes[0] -eq 0xEF -and $bytes[1] -eq 0xBB -and $bytes[2] -eq 0xBF
  $offset = if ($hasBom) { 3 } else { 0 }
  $text = [Text.UTF8Encoding]::new($false, $true).GetString($bytes, $offset, $bytes.Length - $offset)  # strict：非 UTF-8 会抛错
  $updated = [regex]::Replace($text, 'PrecheckServiceId(?!Enum)', 'PrecheckServiceIdEnum')
  $out = [Text.UTF8Encoding]::new($false).GetBytes($updated)
  if ($hasBom) { $out = [byte[]](0xEF,0xBB,0xBF) + $out }
  [IO.File]::WriteAllBytes($p, $out)
  "updated $f (bom=$hasBom, changed=$($text -ne $updated))"
}
```

- 严格解码抛错说明该文件不是 UTF-8，**停下来确认它的原编码再写回**，不要强行按 UTF-8 覆盖。
- 替换用 `PrecheckServiceId(?!Enum)`，避免把已经带后缀的地方变成 `...EnumEnum`。
- 写完用 `git diff --stat <file>` 核对：行数变化应≈命中行数，出现整文件重写就是编码没保住。

### 8. 残留扫描（负向前瞻，必须 0 命中）

```powershell
"=== 剩余未加后缀的引用 ==="
Get-ChildItem -Recurse -File | Where-Object { $_.FullName -notmatch '\\(target|\.git|\.idea)\\' -and $_.Extension -in '.java','.md','.sql' } | Select-String -Pattern 'PrecheckServiceId(?!Enum)' | ForEach-Object { "$($_.Path.Replace((Get-Location).Path + '\','')):$($_.LineNumber)  $($_.Line.Trim())" }
"=== 新枚举位置 ==="
Get-ChildItem far-manu-common\src\main\java\com\sangerbox\common\enums | Select-Object Name
```

0 命中才算引用同步完成；同时确认新名文件在 common/enums 下、旧名文件已不存在。再补一次旧包路径扫描，防止被移动类型的 import 没跟上：

```powershell
Get-ChildItem -Recurse -File -Include *.java | Where-Object { $_.FullName -notmatch '\\(target|\.git)\\' } | Select-String -Pattern 'com\.sangerbox\.business\.integration\.model\.(PrecheckServiceId|PrecheckParentStatus)'
```

### 9. 编译验收

```powershell
"=== modules ==="; (Select-String -Path pom.xml -Pattern '<module>' | ForEach-Object { $_.Line.Trim() }) -join ' '
mvn -B -pl far-manu-common,far-manu-business,far-manu-system -am -DskipTests compile 2>&1 | Select-Object -Last 25
"EXIT=$LASTEXITCODE"
```

`-pl` 用**受影响模块**（common + 引用它的业务模块 + 依赖链），`-am` 带上依赖；记录 `EXIT=0`。把完整输出落日志文件再取尾部，避免管道截断丢失构建结论。

### 10. 核对编译产物（不能只信 EXIT=0）

```powershell
"=== 编译产物 ==="; Get-ChildItem far-manu-common\target\classes\com\sangerbox\common\enums -ErrorAction SilentlyContinue | Select-Object Name
"=== 旧名 class 残留 ==="; Get-ChildItem -Recurse -Filter 'PrecheckServiceId.class' -ErrorAction SilentlyContinue | ForEach-Object { $_.FullName }
```

要求：`PrecheckServiceIdEnum.class` 存在、旧名 `.class` 全仓 0 个。若此时看到 `Nothing to compile - all classes are up to date` 而产物不满足，说明这是增量编译复用旧产物，**该输出不能当验收证据**——需要强制重编受影响模块后再核对一次。

### 11. 汇报

按四块收口：改了什么（哪个枚举 → 新名、落到哪个包、还顺带下沉了谁）/ 同步了哪些类与文档（文件清单）/ 残留扫描结果（`PrecheckServiceId(?!Enum)` 命中 0）/ 编译结论（命令、EXIT、产物核对）。并明确点出**目标仓路径与本会话工作区可能不是同一个**。

## Pitfalls

- **`win_bash` 不存在**：报 `unknown tool "win_bash"` 后立刻改用 `pwsh` 重跑同一条命令，别在同一会话里反复试旧工具名。
- **开会话工作区的 grep 当证据**：内置搜索在目标仓里返回 `No matches found` 是假阴性（目标是另一个仓库），所有引用取证必须在目标仓的 `pwsh` 里做。
- **`edit` 未读先改**：`cannot modify "...": file has not been read`。恢复顺序固定为 `read` → `edit`，对 `git mv` 后的新路径文件也一样。
- **新建+删除代替改名**：会丢重命名历史，评审里看不到 rename；`git status` 必须是 `R`。
- **整文件重写中文文档**：直接 `Set-Content`/`Out-File` 会改编码、丢 BOM，diff 变成全文替换；必须按字节读写并打印 `bom=`。
- **替换把已有后缀再叠一层**：裸 `Replace('PrecheckServiceId', ...)` 会把 `PrecheckServiceIdEnum` 变成 `...EnumEnum`，一律用负向前瞻正则。
- **忘了 SQL/文档注释**：`*.sql` 注释、`*.md` 设计说明里的旧类名同样要改，否则残留扫描不会过。
- **下沉只改名不改 package**：文件搬到 common 但 package 还是旧包，编译或运行期都会出问题；目录与 package 声明必须一致。
- **拿增量编译当验收**：第二次跑出 `Nothing to compile - all classes are up to date` + `EXIT=0` 什么都没证明，必须核对 `target/classes` 产物与旧 `.class` 残留。
- **顺手改了不该改的**：本流水线不动 pom 结构、不删模块、不删目录、不处理死代码；超出范围的改动另开任务。

## Verification

全部满足才算完成：

1. 新枚举落在 `far-manu-common/src/main/java/com/sangerbox/common/enums/<NewName>.java`，`package com.sangerbox.common.enums;` 与目录一致。
2. `git status --short` 中该文件是 `R`（重命名），不是 `A` + `D`。
3. `*.java`/`*.md`/`*.sql` 上 `<OldName>(?!Enum)` 残留扫描 0 命中，且旧名文件已不存在、旧包路径 import 0 命中。
4. `mvn -B -pl <受影响模块> -am -DskipTests compile` 退出码 0。
5. `target/classes/.../enums/` 下存在 `<NewName>.class`，全仓无 `<OldName>.class` 残留。
6. `git diff --stat` 中中文文档/SQL 只有目标行级改动，没有整文件重写。

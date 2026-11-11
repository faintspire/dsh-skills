---
name: java-enum-constant-removal-codetable-sync
description: 删除 Java 状态枚举项、收窄状态码表并让代码/SQL/文档全仓一致时使用：先用「DRAFT|草稿」这类 pattern 全仓 grep 画出 java+sql+docs 引用面，再删枚举项与状态机 ALLOWED_TRANSITIONS 转换边、同清端口接口+适配器实现+全部调用点、改建表 DEFAULT 与 COMMENT、补迁移校验 SQL 与文档流程图，最后复扫残留逐条判定并跑 mvn clean compile 验收；用户说「这个状态不要了」「没有草稿这一说，码表只从 1 开始」「草稿态去掉，代码 SQL 文档都要一致」时使用。
whenToUse: 也适用于文档里的 ASCII 时序图/流程图 edit 报 old_string was not found、需要先 read 精确区间取原文再逐字符重试的场合；不适用于整类死代码删除（走 java-dead-code-removal-verify）或枚举改名下沉 common（走 java-enum-rename-common-module-suffix）。
---

# 删除 Java 状态枚举项并全仓同步码表

适用对象：状态枚举里的一个码值（如 `DRAFT(0, "草稿")`）被产品推翻，码表整体收窄并重新起算（如从 1 开始）。

三条硬边界：

- 本流程**不删任何文件**，只删枚举项、方法声明/实现、转换边和注释文案。
- 残留扫描**不追求 0 命中**：迁移脚本里的历史背景注释与「无草稿态」说明是应当保留的，必须逐条判定并写明理由。
- 分模块契约要三处同清：端口接口 + 适配器实现 + 全部调用点，任何一个留下都是死方法或编译错误。

## Steps

1. 全仓画引用面（改之前做，清单驱动后面每一步）：用 grep 工具，`pattern=DRAFT|草稿`，不限定扩展名、不限定目录。记下命中总数（本次基线 36 处），并按 **java / sql / docs 三类分桶**列成待办清单。
2. 顺状态推进方法名定位三处：grep `pattern=advanceBToPending|advanceCToPending`（换成实际方法名，本次 7 处命中）。命中会落在三处：
   - 端口接口声明：`far-manu-business/src/main/java/com/sangerbox/business/adapter/TaskRepositoryPort.java`（`void advanceBToPending(Long parentTaskId);` 之类）；
   - 适配器实现：取 grep 结果里端口接口之外的那个命中文件（**以 grep 命中的真实路径为准，不要凭记忆假定实现类文件名**）；
   - 全部调用点：`far-manu-business/src/main/java/com/sangerbox/business/app/TaskAppService.java`（本次 B 端提交、C 端提交、支付后创建共三处）。
3. 改枚举本体 `far-manu-business/src/main/java/com/sangerbox/business/enums/TaskStatus.java`，一次改三件事：
   - 删掉 `DRAFT(0, "草稿")` 枚举项；
   - 改类 javadoc：写明「没有草稿态，码值从 1 开始」，并把新码表 `<pre>` 链路列出来（1=待检测 → 2=检测中 → 3=已完成 / 4=检测失败，5=部分完成仅主任务）；
   - 在 `private static final Map<TaskStatus, Set<TaskStatus>> ALLOWED_TRANSITIONS = Map.of(...)` 里删掉所有涉及被删态的转换边（如 `DRAFT→PENDING`），其余边原样保留。只删枚举项不删转换边 → 编译失败或留下不可达状态。
4. 创建处直接置新起始态：`TaskAppService` 里 `bo.setStatus(TaskStatus.DRAFT.getCode())` → `bo.setStatus(TaskStatus.PENDING.getCode())`。
5. 接口与实现成对删方法：`TaskRepositoryPort` 删 `advanceBToPending` / `advanceCToPending` 的 `void` 声明（含其上 javadoc），适配器实现删对应方法体。留一半会变成未实现/未调用的残留。
6. 清调用点并改走异步提交：`taskRepositoryPort.advanceBToPending(parentTaskId);` → `taskAsyncAppService.submitBPrecheckTask(parentTaskId, request.getObjectKey(), selections);`；C 端 → `taskAsyncAppService.submitCPrecheckTask(parentTaskId, objectKey, selections);`；支付成功后的创建处同款处理。
7. SQL 建表与注释（`sql/init.sql`），两处都容易漏：
   - `status SMALLINT NOT NULL DEFAULT 0` → `DEFAULT 1`，edit 带 `replace_all=true`（主任务表与子任务表都有这一列）；
   - COMMENT 文案去掉 `0=草稿`：主任务一套（`任务状态（主任务与子任务统一码表）：1=待检测, ...`）、子任务一套（`任务状态（与主任务同一码表）：1=待检测, ...；5=部分完成 仅主任务使用`）各做一次 edit，均 `replace_all=true`。
8. 迁移脚本 `sql/migrate_unify_task_item_status.sql`：
   - 更新 `COMMENT ON COLUMN`，`c_inspect_task_item.status` 与 `b_inspect_task_item.status` 两条一起改；
   - 注释里说明「主任务表仅刷新列注释（码表无草稿态，从 1 开始），不修改任何数据」；
   - 补人工校验语句并写明期望值域：
     ```sql
     -- 迁移后校验（人工执行）：子任务表不应再有 status=0，值域应为 1/2/3/4
     -- SELECT status, COUNT(*) FROM c_inspect_task_item GROUP BY status ORDER BY status;
     -- SELECT status, COUNT(*) FROM b_inspect_task_item GROUP BY status ORDER BY status;
     ```
9. VO/DTO 与实体注释同步：grep `@Schema` 与 `status` 交叉定位（或直接用第 1 步清单里的 java 命中逐条过），把 VO/DTO 的 `@Schema(description = ...)` 和实体字段 javadoc（如 `CInspectTaskEntity` 的 status 字段，可 `read` 带 `offset`/`limit` 只取该段）改成新码表，去掉「0=草稿」。
10. 文档同步（`docs/*.md`）：命中会分布成「表格 / 要点句 / ASCII 时序图」三类，三类都要改。
    - 表格与要点句：直接 edit，如 `| 主任务与子任务共用 TaskStatus 统一码表 | ... 无草稿态、码值从 1 开始 ... |`；要点句如「建任务与建订单在同一事务」→「下单只落订单与任务入参快照；任务在**支付成功事件**里创建（直接为待检测，无草稿态）并立即提交」。
    - ASCII 时序图 edit 报 `old_string was not found` 时，固定动作只有一条：**read 该文件精确区间取回原文**（本次 `docs/支付设计方案.md`，`offset=304, limit=22`），再用原文**逐字符**（含 `│ ─ ►` 框线与对齐空格）作为 `old_string` 重试。
    - 该处禁止：改用 write 整文件覆盖、跳过不改、凭印象重拼 `old_string`。图里的两处（下单事务「不建任务，入参快照存订单 payload」与「8. AFTER_COMMIT：创建任务(待检测) → 提交 Precheck」）都要改到。
11. 复扫：用**同一个** pattern `DRAFT|草稿` 全仓再跑一次。Java 代码必须 0 命中；剩余命中逐条判定，例如迁移脚本里的「ParentTaskStatus(0=草稿, ...)」历史码表演进注释、「码表已取消 0=草稿」说明、`0=草稿、5=部分完成` 之类的历史背景都属可保留，并写下判定理由。
12. 编译验收：`mvn clean compile`（workdir 仓库根，如 `C:\Develop\project\far-manu-server`，`timeoutMs=600000`），以 Reactor Summary 各模块 `SUCCESS` / `BUILD SUCCESS` 判定。
13. 回报：一张 `文件 / 改动` 两列表格逐条列改动，外加新码表链路一行（如 `1=待检测 → 2=检测中 → 3=已完成 / 4=检测失败（/ 5=部分完成，仅主任务）`）。

## Pitfalls

- **凭印象拼 ASCII 图 old_string 必然失败**。本次第 54、55 两次 edit 连续 `old_string was not found`，原因是框线字符与对齐空格和记忆不符。正确交付是「重读区间 → 逐字符重试」（第 61→62/63 步），不是放弃文档同步，也不是 write 整文件覆盖。
- **只删枚举项，忘删 `ALLOWED_TRANSITIONS` 的转换边**：会编译不过或留下不可达状态；这一步必须和删枚举项一起做。
- **接口删了实现没删（或反之）**：端口接口 + 适配器实现 + 调用点三处要同批清，否则留下未实现方法或死调用。
- **`sql/init.sql` 的 `DEFAULT 0` 漏改**：码表从 1 开始却仍默认 0，是本类改动最常见的一处残留；多张表都有该列，必须 `replace_all=true`。
- **残留扫描追求 0 命中**：迁移脚本里「无草稿态」说明与历史背景注释是**应当保留**的；本类技能与整类死代码删除不同，判据是「逐条判定 + 说明理由」，不是「归零」。
- **只改代码不改文档**：表格、要点句、时序图三处会分散命中，漏一处就是文档与实现不一致；也不要把「删不掉的图」当成「图不用改」。
- **改类名/动模块归属**：本流程不改类名、不动模块归属，那是 `java-enum-rename-common-module-suffix` 的范围。

## Verification

- 同一 pattern 复扫：Java 侧 `DRAFT` 0 命中；非 Java 命中逐条给出「可保留 / 已改」判定与理由。
- 枚举文件：`DRAFT` 不在枚举列表，`ALLOWED_TRANSITIONS` 无涉及被删态的边，javadoc 写明「无草稿态、码值从 1 开始」。
- `sql/init.sql`：`status ... NOT NULL DEFAULT 1`，COMMENT 无 `0=草稿`；迁移脚本含 `SELECT status, COUNT(*) ... GROUP BY status` 校验语句。
- 文档：ASCII 时序图两处已是新流程，要点句已去掉「同一事务建任务」的旧前提，复扫后无遗留旧流程表述。
- `mvn clean compile` 输出 Reactor Summary 各模块 `SUCCESS` 且末行 `BUILD SUCCESS`。
- 交付回复含 `文件 / 改动` 表格与新码表链路。

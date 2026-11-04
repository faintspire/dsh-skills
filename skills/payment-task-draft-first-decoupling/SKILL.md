---
name: payment-task-draft-first-decoupling
description: 在 far-manu-server 把 C 端支付改成「先建任务草稿 → 带 taskId 下单 → 支付成功事件驱动履约」的固定改法：跨模块契约（InspectTaskPort 端口 + PayOrderPaidEvent 事件）放在 far-manu-common，payment 只依赖契约、business 只实现契约，并用部分唯一索引防同一任务重复下单；用户说「先建任务草稿再下单」「付款成功后才把任务推下去提交检测」「payment 和 business 不要互相依赖」「C 端支付改造」时使用。
whenToUse: 用户要求改 far-manu-server 的 C 端下单流程（草稿先行、带 taskId 下单、webhook 推进任务提交检测），或质疑现在「任务与订单联动」的实现方式、要求两个模块解耦时使用。仅讲渠道鉴权/token/createIntent 金额单位、webhook 验签与幂等事件表、币种白名单与按币种取价时不用本技能（分别属于 airwallex-sandbox-auth-spring-feign、airwallex-webhook-receive-local-test、airwallex-multi-currency-order-support）。
---

# 支付：任务草稿先行 + 支付事件驱动履约

目标形态是一条三步链路，两个模块互不认识：

```
① POST /inspect/task/draft              → taskId（任务 status=0 草稿，不调 Precheck）
② POST /pay/c/order {taskId, currency}  → clientSecret（服务端按任务里的服务+币种算价，订单 biz_id=taskId）
③ 用户支付 → webhook → 订单 PAID → 发布 PayOrderPaidEvent → business 监听 → 草稿→待检测，提交 Precheck
```

解耦点只有两个，都放在 `far-manu-common`：`com.sangerbox.common.port.InspectTaskPort`（business 实现）、`com.sangerbox.common.event.PayOrderPaidEvent`（payment 发布）。payment 不得 import `com.sangerbox.business.*`。

## Steps

1. `cd 'C:\Develop\project\far-manu-server'`。动手前先取证真实签名，禁止凭印象编方法名：
   ```
   Get-Content far-manu-business\src\main\java\com\sangerbox\business\adapter\TaskRepositoryPort.java
   Get-Content far-manu-business\src\main\java\com\sangerbox\business\app\TaskAsyncAppService.java
   ```
   记下真实方法名：后续端口里的履约方法直接复用这些入口，不自造。

2. 一次性摸清任务侧现状（类名 + DTO 路径 + 明细实体）：
   ```
   Get-ChildItem -Recurse -File far-manu-business\src\main\java -Include *CInspectTask*.java,*CInspectTaskItem*.java | Select-Object -ExpandProperty Name
   Get-ChildItem -Recurse -File far-manu-business\src\main\java -Filter TaskSubmitDTO.java | Select-Object -ExpandProperty FullName
   ```
   再用 `Get-Content <上一步给出的 FullName>` 读 DTO；读 `CInspectTaskItemEntity.java` 确认履约时 items 怎么落。TODO：TaskSubmitDTO 的包路径按上一步实际输出填，不要写死。

3. 新建跨模块契约 `far-manu-common\src\main\java\com\sangerbox\common\port\InspectTaskPort.java`：
   ```java
   package com.sangerbox.common.port;
   public interface InspectTaskPort {
       /** 查本人名下草稿；任务不存在或不属于该用户时返回 null */
       TaskBrief findOwnedDraft(Long taskId, Long userId);
       /** 支付成功后履约：把任务草稿推进并提交检测 —— 方法名 TODO 按第 1 步取证到的真实入口定 */
   }
   ```
   `TaskBrief` 只暴露下单算价要用的字段（服务标识、该任务金额/币种维度），实现在 business 侧。

4. 新建事件 `far-manu-common\src\main\java\com\sangerbox\common\event\PayOrderPaidEvent.java`，用 record，作为 payment 的唯一出口：
   ```java
   public record PayOrderPaidEvent(String orderNo, Long userId, String bizType, Long bizId,
                                   Long amount, String currency) {}
   ```

5. business 侧写实现 `far-manu-business\src\main\java\com\sangerbox\business\adapter\InspectTaskPortImpl.java`：用 MyBatis-Plus `Wrappers` 查 `CInspectTaskEntity`，归属条件（userId）直接进查询条件，查不到返回 null；履约复用 `TaskAsyncAppService`。

6. business 侧写监听 `far-manu-business\src\main\java\com\sangerbox\business\listener\PayPaidTaskListener.java`：`@EventListener` 接收 `PayOrderPaidEvent`，内部只调用 `InspectTaskPort`，不 import payment 的任何类。

7. `TaskAppService` 增 `createCDraft(TaskSubmitDTO request)`：返回 `Long`，标 `@Transactional(rollbackFor = Exception.class)`，只建草稿（status=0），**不调 Precheck**。改之前先 `read` 带 `offset/limit` 精读目标片段再 `edit`。

8. `CInspectTaskController` 暴露 `@PostMapping("/draft")`，签名 `R<Long> createDraft(@Valid @RequestBody TaskSubmitDTO request)`，`@Operation` 写清「只建任务不提交检测，付完款由支付事件驱动提交」。

9. `far-manu-payment\src\main\java\com\sangerbox\payment\dto\PayOrderCreateDTO.java` 改成只带 `taskId` + `currency` 两个必填字段，不含任何金额字段。

10. `far-manu-payment\src\main\java\com\sangerbox\payment\app\PayCOrderAppService.java`：注入 `InspectTaskPort`，用 `SpAuthUtil` 取当前用户 → `findOwnedDraft(taskId, userId)` 为 null 时 `AssertUtils`/`BizException` 报错；价格由服务端按任务里的服务 + `currency` 计算；订单 `biz_type`/`biz_id` 指向 taskId。

11. `far-manu-payment\src\main\java\com\sangerbox\payment\app\PayWebhookAppService.java`：补 import（`com.sangerbox.common.event.PayOrderPaidEvent`、`BizException`、`ResultCodes`），注入 `ApplicationEventPublisher`。

12. 回调里只在本次真的推进了订单时发事件：
    ```java
    log.info("支付回调处理完成：orderNo={}, 推进行数={}", orderNo, rows);
    if (rows == 1) {
        publisher.publishEvent(new PayOrderPaidEvent(orderNo, order.getUserId(), ...));
    }
    ```

13. `sql\pay_init.sql` 加生效订单的部分唯一索引（防同一任务重复下单/重复支付）：
    ```sql
    CREATE UNIQUE INDEX uk_c_pay_order_biz ON c_pay_order (biz_type, biz_id)
        WHERE deleted = 0 AND biz_id > 0 AND pay_status IN (0, 1, 2);
    ```

14. 先增量编译两个改动模块，用 Select-String 把输出收敛到关键行：
    ```
    mvn -pl far-manu-payment,far-manu-business -am compile 2>&1 | Select-String -Pattern "BUILD|SUCCESS \[|FAILURE|ERROR\]|\.java:\[" | Select-Object -Last 20
    ```

15. 全模块验收，命令与输出都要真实：
    ```
    mvn -DskipTests compile 2>&1 | Select-String -Pattern "BUILD|SUCCESS \[|FAILURE|ERROR\]" | Select-Object -Last 10
    ```

16. 交付：①draft ②下单 ③webhook→事件→履约 三步链路 + 契约归属表（谁定义/谁实现/谁发布）+ 改动清单，并点明「payment 只引用 common 契约，business 只实现契约」。

## Pitfalls

- 没读 `TaskRepositoryPort`、`TaskAsyncAppService` 就写实现，会自造方法名导致编译失败；本技能第 1 步是硬前提。
- 图省事让 payment 直接注入 `TaskAppService`，解耦就白做了；跨模块类型只能放 common，实现留在 business。
- `createCDraft` 里顺手调 Precheck：草稿态的含义是「未付费、未提交」，提交只能由事件链路触发，否则没付钱也跑检测。
- 端口契约返回 null 的语义必须守住：不要把「不属于该用户」改成抛异常或先查后比对再报 403，payment 侧按 null 判错即可。
- webhook 会重放：事件不能无条件 `publishEvent`，否则重复履约；只在 `rows == 1` 时发。
- 唯一索引漏写 `WHERE deleted = 0 AND biz_id > 0 AND pay_status IN (0, 1, 2)`，会把已取消/失败的历史订单一起算进唯一约束，导致正常二次下单被拒。
- 金额不要从 DTO 收：只收 `taskId + currency`，价格由服务端算；币种白名单、按币种取价、回调币种校验属于 airwallex-multi-currency-order-support，这里不重复实现。
- 大文件直接整段 `edit` 容易改错位置，先 `read`（带 `offset`/`limit`）再改。

## Verification

- `mvn -DskipTests compile` 输出里 7 个模块（含 far-manu-common、far-manu-integration、far-manu-payment、far-manu-business）全 SUCCESS，贴原始输出行，不接受「应该能编译」。
- `Get-ChildItem -Recurse far-manu-payment\src\main\java -Include *.java | Select-String -Pattern "com\.sangerbox\.business"` 无命中。
- `Select-String -Path sql\pay_init.sql -Pattern "uk_c_pay_order_biz"` 命中，且该语句含 `deleted = 0` 与 `pay_status IN (0, 1, 2)`。
- 逐项对表：draft 接口返回 taskId 且新建任务 status=0、无 Precheck 调用；下单 DTO 无金额字段、订单 biz_id==taskId；`PayWebhookAppService` 的 `publishEvent` 在 `if (rows == 1)` 内；`PayPaidTaskListener` 上确有 `@EventListener`。
- 任何无法从代码确认的点（例如端口履约方法名、TaskBrief 字段）当场 grep/read 补齐；补不到就标 TODO 并给出核对命令，不要编造接口或方法名。

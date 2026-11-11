---
name: airwallex-webhook-receive-local-test
description: 在 far-manu-server 接入并排查 Airwallex 沙盒 webhook 接收侧（HMAC-SHA256 原始字节验签、sys_pay_event event_id 唯一索引幂等、按 event_type 分派到 PayEventHandler、SecurityProperties 与 Nacos 双层放行），硬判据是「动手改前先连库统计 sys_pay_event 的 event_type×status 分布与真实 payload，改完用库里真实报文离线回放验收」；用户说「webhook 建好了怎么接」「沙盒回调怎么测」「未注册处理器，仅落库」「事件一直 status=0 / 钱到账订单没履约」「哪些事件在推、哪个 handler 是死判断」「要不要给这个事件加处理器」时使用。
whenToUse: 用户已拿到 Airwallex 沙盒 whsec_ 签名密钥要求把回调接进 far-manu-server、要求给出本地免真实付款的自测办法、或反馈回调被 Sa-Token 拦掉/重放重复履约/签名校验不通过/某事件类型未注册处理器/事件停在 status=0 订单没推进时使用；也用于「先查库看真实在推的事件再改分派」「核对已注册处理器全集」这类取证与审计请求。若需求是 API Key 鉴权、createIntent、金额单位换算与客户端收敛，走 airwallex-sandbox-auth-spring-feign；若只是删类/接口级死代码，走 java-dead-code-removal-verify，不用本 skill。
---

# Airwallex 沙盒 webhook 接收 + 本地自测

本仓库根：`C:\Develop\project\far-manu-server`（模块 `far-manu-common` / `far-manu-integration` / `far-manu-payment`）。主干是 **验签 → sys_pay_event 幂等 → 按 event_type 分派履约**，另有两句不能省的硬判据：

1. **先查库拿事实**：改代码前必须连生产库统计 `sys_pay_event` 的 `event_type × status` 分布并抽真实 payload，确认哪些事件真在推、哪些 handler 里的判断是死代码。代码假设会和库里的现实对不上（本仓实证：`payment_attempt.failed` 一条都没有，真实拒付是没接的 `payment_intent.payment_failed`）。
2. **再用真实报文回放**：收尾不用「编译通过」交差，而用库里真实 payload 离线回放新分派逻辑、逐个 `[PASS]`，并在回答里给「事实 / 证据」表。

## Steps

1. **先读后改**。要改的每个文件（尤其 `far-manu-common/src/main/java/com/sangerbox/common/config/SecurityProperties.java`、`far-manu-common/src/main/java/com/sangerbox/common/filter/TraceIdFilter.java`）先用 read 打开再 edit，否则写工具会以 `file has not been read` 拒绝。

2. **先连库统计事件分布（动笔前的硬门）**。本机没有 `psql`，用 `~/.m2` 里的 PostgreSQL 驱动 + 单文件 Java 直连（连接串沿用本环境既有值，不在回复里回显内网地址）：
    ```powershell
    $driver = "$env:USERPROFILE\.m2\repository\org\postgresql\postgresql\42.7.7\postgresql-42.7.7.jar"
    java -cp $driver "C:\Users\<user>\AppData\Local\Temp\pgq\PgQ.java" `
      "select event_type, status, count(*) from sys_pay_event group by event_type, status order by 3 desc"
    ```
    `PgQ.java` 只做三件事：`DriverManager.getConnection("jdbc:postgresql://<host>:5432/farmanu?currentSchema=public", ...)`、把 `args` 里每条 SQL 依次执行、按 `列名 | 值` 打印表头 + 行。
    一次拿到的分布直接决定改什么：本轮据此发现 `payment_intent.requires_payment_method`(15) / `payment_attempt.authentication_redirected`(14) 等大量在推、`payment_attempt.failed` 为 0、`payment_intent.payment_failed`(4) 与 `payment_attempt.expired`/`cancelled` 未注册，以及 11-03 的 `payment_intent.succeeded` 至今 `status=0`（钱到账但事件没标处理）。

3. **字段存在性必须实测，不能读代码猜**。对每类事件取一条最新 payload 逐字段验证：
    ```sql
    select position('merchant_order_id' in payload) as pos_moid, length(payload) as len
    from sys_pay_event where event_id = '<event_id>'
    ```
    `pos_moid = 0` 就是字段不存在。本轮两条实证：`data.object` 里**没有 `object` 字段**（22 种事件回放全空，因此 `"payment_attempt.failed".equals(object.getString("object"))` 是死判断）；`payment_attempt` 类事件有的带 `merchant_order_id`（expired/cancelled），有的不带（settled/paid），想接必须靠 `payment_intent_id` 反查订单。需要看全量报文时用 `select payload from sys_pay_event where event_type='<type>' order by id desc limit 1`。

4. **事件语义查官方，不靠命名推断**。`web_search` 在本环境常因缺 `DEEPSEEK_API_KEY` 直接失败，别卡在那里——改用 `web_fetch` 直接抓一手文档：`https://www.airwallex.com/docs/payments/accept-payments/_payment-intent-statuses`（PaymentIntent 状态语义）、`https://www.airwallex.com/docs/developer-tools/webhooks/listen-for-webhook-events`（会投递哪些事件、各代表什么）。结论必须能落到「这是终态还是中间态、是不是履约时机」。

5. **某事件类型「未注册处理器，仅落库」的定位链**（顺序不能颠倒）：
    1) 先 grep 事件类型字符串本身，例如 `grep(pattern="requires_customer_action|REQUIRES_CUSTOMER_ACTION", path="C:\\Develop\\project\\far-manu-server")`，**0 命中**才敢说「全仓没有该分支」；
    2) 再 read 取证分派服务 `far-manu-payment/src/main/java/com/sangerbox/payment/app/PayWebhookAppService.java`（真实逻辑：事件入库后按 event_type 查 handler，无 handler 只置 `status=1`）、接口 `far-manu-payment/src/main/java/com/sangerbox/payment/webhook/PayEventHandler.java`、`webhook/handler/` 下各实现、`constant/PayConstants.java` 的事件状态常量；
    3) 用官方文档定语义（见步骤 4）；
    4) 处置只允许二选一：**新增 handler 并注册 event_type**，或**显式「已知忽略」**（独立状态 / 明确日志 / 计数），不得继续静默标为已处理。

6. **审计已注册事件全集**。在 `far-manu-payment` 的 `*.java` 上 grep `eventTypes\(\)|Set\.of\("|payment_intent\.|payment_attempt\.|refund\.`（本轮 11 处命中）框出 `PayEventHandler` 接口与 handler 目录，再逐文件 read，输出「事件名 / 处理器 / 做的事 / 订单状态变化」表，事件名取 `eventTypes()` 原文（本轮 3 个处理器 / 6 个事件），并显式声明「没有别的 bean 注册同一接口」。同时指出哪些事件没有订单/履约副作用。

7. **抽常量类**：新建 `far-manu-payment/src/main/java/com/sangerbox/payment/constant/PayConstants.java`（`final` 类 + 私有构造），放 `DEFAULT_CURRENCY`、`ORDER_TTL_MINUTES`（注释标明与关单任务一致）、`PAY_STATUS_*`、`FULFILL_STATUS_*` 以及 **webhook 事件状态常量**（至少 收到 / 已处理 / `FAILED` 三态，取值以实际落地为准）；再把 `PayCOrderAppService.java` 里的魔法数字全部替换为 `PayConstants.*`，改完 grep 确认无残留字面量。

8. **建事件表**：在 `sql/pay_init.sql` 末尾追加 `sys_pay_event`：`id BIGSERIAL PRIMARY KEY`、`event_id VARCHAR(64) NOT NULL`（**唯一索引**，幂等靠它）、`event_type VARCHAR(64) NOT NULL DEFAULT ''`、`provider`、`payload`（原始报文，TEXT 级）、时间戳列、`status`。唯一索引必须落 SQL；`status` 必须能区分「收到 / 已处理 / 失败」，**不能**用「行存在」当已处理。实体与 Mapper：`entity/SysPayEventEntity.java`（继承本仓 `BaseEntity`，`@TableName("sys_pay_event")`）与 `mapper/SysPayEventMapper.java`。

9. **验签器**：`far-manu-integration/src/main/java/com/sangerbox/integration/pay/AirwallexWebhookVerifier.java`：待签名串 = `x-timestamp` **原始字符串** + **原始报文 body**（rawBody，不能反序列化再序列化）；`Mac.getInstance("HmacSHA256")` + `SecretKeySpec(secret.getBytes(UTF_8), "HmacSHA256")`，输出 hex（`HexFormat`）；比较用 `MessageDigest.isEqual`，禁止 `String.equals`；签名错误/时间戳缺失抛业务异常（4xx），不要冒泡成 500。

10. **密钥配置**：`far-manu-integration/src/main/java/com/sangerbox/integration/pay/AirwallexProperties.java` 增加 `webhookSecret`（注释：后台 Developer > Webhooks 里每个订阅独立的签名密钥，`whsec_` 开头）。

11. **一个事件一个处理器**：`webhook/PayEventHandler.java` 暴露 `Set<String> eventTypes()` 与 `void handle(String eventType, JSONObject object, String orderNo)`；`payment_intent.succeeded`、`payment_intent.payment_failed`、退款各一个独立 `@Component` 实现。**新事件只加 bean，不改分派逻辑**。成功处理器对本地 `pay_status` 0（已创建）/1/3（已关闭）放行，避免已到账款项被丢弃；金额单位与 createIntent 写入口径一致（本仓为分）。

12. **分派服务只留验签 / 幂等 / 路由 + 失败处置语义**：`PayWebhookAppService` 只做验签 → 取 `event_id`/`event_type` → `payload` 原文入库 → 按 `eventType` 路由到 handler → 成功才 `markProcessed`。
    - **throw 一律置 FAILED 并继续抛出**：`catch (RuntimeException e) { markFailed(eventId); throw e; }`，让控制器回非 200 交网关重试。**禁止** catch 后按「已处理」落库。
    - 幂等判定按**处理状态**（处理成功才跳过），不是「行存在即跳过」，否则失败事件无法被重投递再处理。

13. **控制器**：`far-manu-payment/src/main/java/com/sangerbox/payment/controller/PayWebhookController.java` 暴露 `POST /pay/webhook/airwallex/c`，取**原始报文**与 `x-timestamp` / 签名头，原样交给 app service，返回值用本仓 `R`。优先 `@RequestBody byte[]`（`String` 会按 Content-Type 字符集解码，带中文商品描述的非 ASCII 报文会被改写导致签名整串对不上）；等价做法是 `HttpServletRequest` 直接读流。临时诊断分支 TODO：后台「测试事件」若用报文自带的密钥签名（头部名与回落规则本轮未取证，确认后补进本节）。

14. **日志脱敏**：`TraceIdFilter` 打印请求体的分支里跳过 `request.getRequestURI().startsWith("/pay/webhook")`——回调报文含 `client_secret`，落日志即泄露。

15. **双层放行**（缺一不可）：
    - 代码侧：`SecurityProperties` 路径列表在 `/inspect/callback/**`、`/b/inspect/callback/**` 之后加 `"/pay/webhook/**"`。
    - Nacos 侧：`security.exclude-paths` 是**整表覆盖**。先取回整份配置原文，在原有条目基础上**补一行** `/pay/webhook/**`，再整体发布回去。
    - Nacos 走 OpenAPI（`/nacos/v1/auth/login` 取 token，`/nacos/v1/cs/configs?dataId=&group=&tenant=` 读写），dataId/group 用该服务实际配置名（TODO：确认后把名字补进本节）。

16. **本地签名脚本**：仓库根新建 `check-pay-webhook.ps1`，参数：
    ```powershell
    param(
        [string]$BaseUrl = 'http://127.0.0.1:11012',
        [Parameter(Mandatory = $true)][string]$OrderNo,
        [decimal]$Amount = 9.90,
        [string]$Currency = 'USD',
        [string]$Secret
    )
    ```
    拼一条 `payment_intent.succeeded` 报文 → `$ts = [DateTimeOffset]::UtcNow.ToUnixTimeSeconds()` → `HMACSHA256(secret, $ts + $body)` 转 hex → 带 `x-timestamp` 与签名头 POST 到 `$BaseUrl/pay/webhook/airwallex/c` → 断言 200。再依次跑失败分支：错误签名、金额与订单不一致、同一 `event_id` 重放（仍 200 但不重复履约）。端口按本服务实际监听端口确认后改默认值。

17. **编译**：
    ```powershell
    cd 'C:\Develop\project\far-manu-server'
    mvn -pl far-manu-payment -am compile 2>&1 | Select-String -Pattern "BUILD|SUCCESS \[|FAILURE|ERROR\]|\.java:\[" | Select-Object -Last 15
    ```
    必须 `BUILD SUCCESS`。这只算入场券，不是验收。

18. **离线回放库里真实 payload（真正的验收）**：写一个单文件 `TestWebhook.java`，用**真 handler + 真 PayWebhookAppService** 配**假 mapper**（内存里记录 `orderUpdates`、`eventStatuses`、`lastError`），对每个事件跑 `run(service, type, payloads)` 并逐条打印 `[PASS] <类型> <断言>`：
    - classpath 三件套：`$root\far-manu-payment\target\classes` + `$root\far-manu-common\target\classes` + `cp.txt`，**再显式拼上 PostgreSQL 驱动 jar**（漏了报 `No suitable driver found for jdbc:postgresql://...`）。
    - `cp.txt` 生成：`mvn -o -q org.apache.maven.plugins:maven-dependency-plugin:3.6.1:build-classpath "-Dmdep.outputFile=target/cp.txt" "-DincludeScope=runtime"` —— **每个 `-D` 参数整体加引号**，否则 pwsh 拆参报 `Unknown lifecycle phase ".outputFile=..."`。
    - 无 Spring 时 MyBatis-Plus 的 `LambdaUpdateWrapper` 取不到列名（`TableInfo` 未登记）：先 `new MybatisConfiguration()` 再手工 `TableInfoHelper.initTableInfo(...)` 登记实体（重载按本仓 mybatis-plus 版本确认）。
    - 编码：`java "-Dfile.encoding=UTF-8" -cp $cp "...\TestWebhook.java"`（`-D` 必须是独立带引号参数），否则 stdout 中文乱码，`[PASS]` 断言里的字段看不出来。
    - 断言面：订单 `pay_status` 推进到已支付(2)、事件 `status=已处理`；金额/币种与本地订单不一致时事件落 `FAILED` 且订单不变；handler 抛异常时异常**继续抛出**且事件 `FAILED`（对应 HTTP 非 200）；同一 `event_id` 重放不重复履约。

19. **收尾**：回答里先给「事实 / 证据」表（分布数据、死判断、被吞的 `status=0` 事件、字段缺失各一行，带 SQL 或回放输出），再给回放 `[PASS]` 清单；如果用户把 `whsec_...` 明文贴进对话，明确告知已泄露、联调后去后台重新生成，且任何回复里都不回显 secret 明文。

## Pitfalls

- **读代码猜字段 = 造死判断**：`data.object` 里没有 `object` 字段，`"payment_attempt.failed".equals(object.getString("object"))` 永远为 false。判定事件类别用事件名/资源类型这类**实测存在**的字段。
- **没有 handler 也置 `status=1`** 是静默吞：漏处理与「本来就无需处理」从此无法区分。必须三态（收到/已处理/失败）+ 未注册时打明确日志或计数。
- **catch 后 `markProcessed`** 会把已收款事件永久卡在「已处理」：置 `FAILED` 并 rethrow，让网关重投递。
- **「行存在即已处理」的幂等**同样是吞：按处理状态判，失败事件要能被再次处理。
- **只信代码不信库**：本轮 `payment_attempt.failed` 在库里 0 条，真实拒付是 `payment_intent.payment_failed`。先跑分布统计再改分派面。
- **先查后插不是幂等**：并发下双写，唯一索引 + 捕获冲突才是真幂等；冲突后返回成功避免渠道反复重试。
- **签名只能对原始字节算**：反序列化再 `JSON.toJSONString` 重拼、或用 `@RequestBody String` 按字符集解码非 ASCII 报文，签名都永远不过。
- **只用 `String.equals` 比较 HMAC** 有时序侧信道问题，必须 `MessageDigest.isEqual`。
- **验签失败返回 500** 会让渠道持续重试并触发告警；应返回 4xx。
- **pwsh 下的 `-D` 参数**：`mvn ... -Dmdep.outputFile=...` 会被拆参，必须 `"-Dmdep.outputFile=..."`；`java -Dfile.encoding=UTF-8` 同理。
- **回放脚本的 classpath**：忘拼驱动 jar → `No suitable driver found`；忘 `TableInfo` 登记 → `LambdaUpdateWrapper` 取不到列名。
- **Nacos 整表覆盖**：只把 `/pay/webhook/**` 发上去会冲掉 `security.exclude-paths` 其余条目；补回条目缩进必须 4 空格（与兄弟项同层），6 空格会变嵌套导致整块解析失败。发布后逐行打印缩进回读。
- **`BizException(ResultCodes.BAD_REQUEST, msg)` 编译不过**：本仓收 `int` code，写 `ResultCodes.BAD_REQUEST.getCode()`。
- **直接 edit 未读过的文件**会被拒，批量改文件统一先 read 再 edit。
- **`TraceIdFilter` 会把回调报文打进日志**，隐藏 `client_secret` 泄露；放行路径与脱敏一起做。
- **`web_search` 缺 key 不是死路**：直接 `web_fetch` 官方文档页，比搜索引擎二手结论更可靠。
- **secret 明文入对话**：视为已泄露，要求重置；回复里不回显。

## Verification

1. 回答开头有 `sys_pay_event` 按 `event_type × status` 的统计（含各类型条数），并能指出「库里 0 条」的死分支与未注册的真实事件类型。
2. 离线回放全部 `[PASS]`：列出每条用例、断言与结果，至少覆盖 成功推进(0/1/3→2)、金额不一致落 FAILED 且订单不变、handler 异常置 FAILED 并继续抛出、同 `event_id` 重放不重复履约。
3. 失败处置可验证：`markFailed` 后事件 `status=FAILED`，异常仍向上抛（控制器回非 200），日志里能看到该 `event_id`。
4. `mvn -pl far-manu-payment -am compile` 输出 `BUILD SUCCESS`，Reactor 里 `far-manu-common` / `far-manu-integration` / `far-manu-payment` 全 SUCCESS。
5. `./check-pay-webhook.ps1 -OrderNo <真实测试单号> -Amount 9.90` 返回 200、订单推进、履约落库；错误签名 4xx 且不写事件表；同 `event_id` 再跑仍 200 且 `select count(1) from sys_pay_event where event_id='<id>'` 恒为 1、`payload` 非空。
6. Nacos 回读 `security.exclude-paths`：逐行打印缩进，`/pay/webhook/**` 与 `/health`、`/auth/**` 同为 4 空格，原有条目一条没少。
7. 不带 token POST 回调地址，拿到的是业务 4xx 而不是 Sa-Token 未登录拦截。
8. 关键结论都带证据：`文件:行号` 链接或官方文档链接，「事实 / 证据」表在回答里；已提醒重置明文泄露的 `whsec_`。

---
name: airwallex-webhook-receive-local-test
description: 在 far-manu-server 接入 Airwallex 沙盒 webhook 的接收侧（魔法数字抽 PayConstants、sys_pay_event 事件表做幂等、HMAC-SHA256 验签、原始报文回调控制器、SecurityProperties 与 Nacos 双层放行）并用 check-pay-webhook.ps1 在本地免真实付款自测；用户说「webhook 建好了怎么接」「沙盒回调怎么测」「不想每次真实付款」「验签/重放/幂等问题」时使用。
whenToUse: 用户已经拿到 Airwallex 沙盒的 whsec_ 签名密钥，要求把回调接进 far-manu-server、或要求给出本地自测办法、或反馈回调被 Sa-Token 拦掉/重放重复履约/签名校验不通过时使用。若需求是 API Key 鉴权、createIntent、金额单位换算与客户端收敛，走 airwallex-sandbox-auth-spring-feign，不用本 skill。
---

# Airwallex 沙盒 webhook 接收 + 本地自测

本仓库根：`C:\Develop\project\far-manu-server`（模块 `far-manu-common` / `far-manu-integration` / `far-manu-payment`）。本 skill 只覆盖**回调接收侧**：验签、幂等事件表、原始报文留存、放行路径、本地模拟自测。鉴权与 createIntent 在另一条链路，不要混进来。

## Steps

1. **先读后改**。要改的每个文件（尤其 `far-manu-common/src/main/java/com/sangerbox/common/config/SecurityProperties.java`、`far-manu-common/src/main/java/com/sangerbox/common/filter/TraceIdFilter.java`）先用 read 打开再 edit，否则写工具会以 `file has not been read` 拒绝。

2. **抽常量类**：新建 `far-manu-payment/src/main/java/com/sangerbox/payment/constant/PayConstants.java`（`final` 类 + 私有构造），放 `DEFAULT_CURRENCY = "USD"`、`ORDER_TTL_MINUTES`（注释标明与关单任务一致）、`PAY_STATUS_*`、`FULFILL_STATUS_*` 以及 webhook 事件状态常量。

3. **改引用**：`far-manu-payment/src/main/java/com/sangerbox/payment/app/PayCOrderAppService.java` 里原有的魔法数字全部替换为 `PayConstants.*`，付款/履约状态判断同步替换，改完 grep 一遍确认没有残留字面量。

4. **建事件表**：在 `sql/pay_init.sql` 末尾追加 `sys_pay_event`：`id BIGSERIAL PRIMARY KEY`、`event_id VARCHAR(64) NOT NULL`（**唯一索引**，幂等靠它）、`event_type VARCHAR(64) NOT NULL DEFAULT ''`、`provider`、`payload`（原始报文字段，TEXT 级）、时间戳列。唯一索引必须在 SQL 里落，不靠应用层先查后插。

5. **实体与 Mapper**：`far-manu-payment/src/main/java/com/sangerbox/payment/entity/SysPayEventEntity.java`（继承本仓 `BaseEntity`，`@TableName("sys_pay_event")`）与 `far-manu-payment/src/main/java/com/sangerbox/payment/mapper/SysPayEventMapper.java`（`extends BaseMapper<SysPayEventEntity>`）。

6. **验签器**：`far-manu-integration/src/main/java/com/sangerbox/integration/pay/AirwallexWebhookVerifier.java`：
   - 待签名串 = `x-timestamp` **原始字符串** + **原始报文 body**（必须是 rawBody，不能是反序列化后再序列化的 JSON）。
   - `Mac.getInstance("HmacSHA256")` + `SecretKeySpec(secret.getBytes(UTF_8), "HmacSHA256")`，输出 hex（`HexFormat`）。
   - 比较用常量时间：`MessageDigest.isEqual(expected.getBytes(UTF_8), actual.getBytes(UTF_8))`，禁止 `String.equals`。
   - 签名错误/时间戳缺失都抛业务异常（4xx），不要冒泡成 500。

7. **密钥配置**：`far-manu-integration/src/main/java/com/sangerbox/integration/pay/AirwallexProperties.java` 增加 `webhookSecret` 字段（注释写明「后台 Developer > Webhooks 里每个订阅独立的签名密钥，whsec_ 开头」）。

8. **回调服务**：`far-manu-payment/src/main/java/com/sangerbox/payment/app/PayWebhookAppService.java`：验签 → 取 `event_id`/`event_type` → `payload` 原文入库 → 按 `event_type` 分派（`payment_intent.succeeded` 走履约）→ 事件表插入命中唯一索引即视为已处理，直接返回成功且不重复履约。金额不一致、订单不存在等属于业务失败，**不能**当成「已处理」吞掉。

9. **控制器**：`far-manu-payment/src/main/java/com/sangerbox/payment/controller/PayWebhookController.java` 暴露 `POST /pay/webhook/airwallex/c`，取**原始报文**（`HttpServletRequest` 读流或 `@RequestBody String`）与 `x-timestamp` / 签名头，原样交给 app service，返回值用本仓 `R`。

10. **日志脱敏**：`TraceIdFilter` 打印请求体的分支里跳过 `request.getRequestURI().startsWith("/pay/webhook")`——回调报文含 `client_secret`，落日志即泄露。

11. **双层放行**（缺一不可）：
    - 代码侧：`SecurityProperties` 的路径列表在 `/inspect/callback/**`、`/b/inspect/callback/**` 之后加 `"/pay/webhook/**"`。
    - Nacos 侧：`security.exclude-paths` 是**整表覆盖**语义。先取回整份配置原文，在原有条目基础上**补一行** `/pay/webhook/**`，再整体发布回去；只发新条目会把 `/health`、`/auth/**`、`/swagger-ui/**` 等全部冲掉。
    - Nacos 取值/发布走 OpenAPI（`/nacos/v1/auth/login` 取 token，`/nacos/v1/cs/configs?dataId=&group=&tenant=` 读，POST 同路径发布），地址与 tenant 沿用本环境既有值；dataId/group 用该服务在 Nacos 中的实际配置名（TODO：确认后把名字补进本节）。

12. **本地签名脚本**：仓库根新建 `check-pay-webhook.ps1`，参数与已实现一致：
    ```powershell
    param(
        [string]$BaseUrl = 'http://127.0.0.1:11012',
        [Parameter(Mandatory = $true)][string]$OrderNo,
        [decimal]$Amount = 9.90,
        [string]$Currency = 'USD',
        [string]$Secret
    )
    ```
    脚本内：拼一条 `payment_intent.succeeded` 报文 → `$ts = [DateTimeOffset]::UtcNow.ToUnixTimeSeconds()` → `HMACSHA256(secret, $ts + $body)` 转 hex → 带 `x-timestamp` 与签名头 POST 到 `$BaseUrl/pay/webhook/airwallex/c` → 断言 200。再依次跑失败分支：错误签名、金额与订单不一致、同一 `event_id` 重放（仍应 200 但不重复履约）。`$BaseUrl` 端口按本服务实际监听端口确认后改默认值。

13. **编译验收**：在仓库根执行
    ```powershell
    cd 'C:\Develop\project\far-manu-server'
    mvn -pl far-manu-payment -am compile 2>&1 | Select-String -Pattern "BUILD|SUCCESS \[|FAILURE|ERROR\]|\.java:\[" | Select-Object -Last 15
    ```
    必须看到 `BUILD SUCCESS`。

14. **收尾提醒**：如果用户把 `whsec_...` 明文贴进了对话，明确告知这条 secret 已算泄露，联调完成后去渠道后台重新生成一次；不要在任何回复里回显 secret 明文。

## Pitfalls

- **先查后插不是幂等**：并发回调下应用层「查 event_id 存在吗」会双写，唯一索引 + 捕获冲突才是真幂等；捕获冲突后返回成功，避免 Airwallex 反复重试。
- **签名用的 body 必须是原始报文**：用 fastjson 反序列化再 `JSON.toJSONString` 重拼，字段顺序/空格/转义一变签名就永远不过。
- **只用 `String.equals` 比较 HMAC** 会被逐字节时序侧信道攻击，必须 `MessageDigest.isEqual`。
- **验签失败返回 500** 会让渠道持续重试并触发告警；应返回 4xx。
- **Nacos 整表覆盖**：本轮真实踩坑——只把 `/pay/webhook/**` 发上去，`security.exclude-paths` 其余条目会被冲掉；而且补回的条目缩进必须是 4 空格（与兄弟项同层），写成 6 空格会变成嵌套导致整块解析失败。发布后逐行打印缩进回读确认。
- **`BizException(ResultCodes.BAD_REQUEST, msg)` 编译不过**：本仓 `BizException` 收的是 `int` code，要写 `ResultCodes.BAD_REQUEST.getCode()`。
- **直接 edit 未读过的文件**会被拒（`file has not been read`），批量改文件统一先 read 再 edit。
- **TraceIdFilter 会把回调报文打进日志**，隐藏 `client_secret` 泄露；放行路径和脱敏要一起做。
- **金额单位**：比对金额时的最小单位必须与 createIntent 写入的口径一致（本仓为分），拿元去比分会导致所有回调被判「金额不一致」。

## Verification

1. `mvn -pl far-manu-payment -am compile` 输出 `BUILD SUCCESS`，Reactor Summary 里 `far-manu-common` / `far-manu-integration` / `far-manu-payment` 全 SUCCESS。
2. `./check-pay-webhook.ps1 -OrderNo <真实测试单号> -Amount 9.90` 返回 200，且订单状态从待支付变为已支付、履约落库；错误签名分支返回 4xx 且不写事件表；金额不一致分支返回业务失败且订单状态不变；同 `event_id` 再跑一次仍 200 但履约记录不增加（`select count(1) from sys_pay_event where event_id = '<id>'` 恒为 1，`payload` 非空）。
3. Nacos 回读 `security.exclude-paths`：逐行打印缩进，`/pay/webhook/**` 与 `/health`、`/auth/**` 等兄弟项同为 4 空格，且原有条目一条没少。
4. 不带 token 直接 POST 回调地址，返回的是业务响应/4xx 业务错误而不是 Sa-Token 的未登录拦截，证明放行生效。
5. 回复里确认：已提醒用户重置明文泄露的 `whsec_`。

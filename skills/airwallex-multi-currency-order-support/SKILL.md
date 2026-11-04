---
name: airwallex-multi-currency-order-support
description: 把 far-manu-server 的 payment 模块从写死 USD 改造成 CNY/USD/HKD 三币种同时可付（币种常量与白名单收进 PayConstants、PayOrderCreateDTO 增加 currency 入参校验、按「服务+币种」查 sys_pay_price 且缺价整单拒绝、订单与订单行存原币金额不做汇率折算、webhook 增加币种一致性校验、pay_init.sql 补齐三套价目种子），并以 mvn 编译和三币种真实建单验收；当用户说「支付要同时支持人民币/美元/港币」「代码里好像只跑美元」「每个币种单独标价」「不要做汇率换算」「多币种下回调校验够不够」时使用。
whenToUse: 用户要求 far-manu-server / far-manu-payment 支持 CNY、USD、HKD 三币种下单，或要求核对/补多币种场景下的回调校验（只比金额不够，还要比币种）时使用。仅涉及 Airwallex 鉴权/token 与 createIntent 金额单位（分/元）时不要用本技能，那是 airwallex-sandbox-auth-spring-feign 的范围；只涉及回调接收侧验签/幂等事件表时用 airwallex-webhook-receive-local-test。
---

# Airwallex 多币种下单改造（CNY / USD / HKD）

本技能把支付模块的「币种维度」一次改到位：入参白名单、按币种取价、原币存储不折算、回调币种校验、价目种子三套。

**范围边界（重要）**：

- 鉴权、token 生命周期、`createIntent` 的金额单位（分/元）与客户端收敛 → `airwallex-sandbox-auth-spring-feign`。
- 回调接收侧（验签、幂等事件表、放行路径）→ `airwallex-webhook-receive-local-test`。
- 本技能只补其中**币种**这一层判据，不重做上面两块。

## Steps

1. **先取证再动手，不要直接改。** 在仓库根目录（证据环境为 `C:\Develop\project\far-manu-server`）grep 币种硬编码与价目表结构，先给出结论：
   - `Select-String -Path .\far-manu-payment\src\main\java\**\*.java -Pattern 'USD|currency|Currency'`
   - 确认 `sys_pay_price` / 订单 / 订单行表结构是否已有 currency 列（证据结论是「表结构支持，但代码写死 USD」）。
   - 把结论先讲给用户，再进入改造。
2. **币种常量收口到 `far-manu-payment\src\main\java\com\sangerbox\payment\constant\PayConstants.java`**，不散落硬编码：
   ```java
   import java.util.Set;

   /** 本期默认币种 */
   public static final String DEFAULT_CURRENCY = "USD";
   /** 本期支持币种：人民币、美元、港币，是否真能收还要看渠道是否开通对应币种 */
   public static final Set<String> SUPPORTED_CURRENCIES = Set.of("CNY", "USD", "HKD");
   ```
   注意同时补 `import java.util.Set;`（证据里这一行是单独一次 edit 补上的）。
3. **下单入参加币种**：`far-manu-payment\src\main\java\com\sangerbox\payment\dto\PayOrderCreateDTO.java` 新增 `currency` 字段。
   - 不传 → 取 `PayConstants.DEFAULT_CURRENCY`（USD），不强制前端必填。
   - 传了 → 必须落在 `SUPPORTED_CURRENCIES` 白名单内，否则校验报错。校验写法沿用该 DTO/服务层现有风格（`@NotEmpty` 等 Bean Validation 或 `AssertUtils`，以取证到的实际代码为准）。
4. **下单按「服务 + 币种」取价**：在 `far-manu-payment\src\main\java\com\sangerbox\payment\app\PayCOrderAppService.java` 下单主流程里，用 `currency` 与勾选的 `services` 逐项查 `sys_pay_price`（列：`service_id`、`currency`、`unit_amount`）。
   - 某个服务在该币种下缺价 → **整单拒绝**，报错信息必须指明是哪个服务缺价。
   - 不要拿 USD 价目或默认币种价格顶替，也不要做任何汇率折算。
5. **订单与订单行存原币金额**：订单、订单行都落「下单币种 + 该币种金额」，价格来自该币种价目行。全程不出现汇率、基准币种、折算系数这类概念。
6. **渠道意图金额转精度**：建 Airwallex 意图时币种跟着订单走，金额按该币种精度转换，用 `Currency.getDefaultFractionDigits()`（而不是写死 ×100 或写死两位）。
7. **回调补币种一致性校验**：在 `far-manu-payment\src\main\java\com\sangerbox\payment\app\PayWebhookAppService.java` 里，除原有金额校验外新增币种校验，取回调 payload 的 `currency` 与本地订单币种比较（忽略大小写），不一致就 `log.error` 并 `return`，**不推进订单状态**：
   ```java
   String paidCurrency = object.getString("currency");
   if (!order.getCurrency().equalsIgnoreCase(paidCurrency)) {
       log.error("回调币种与本地订单不一致，拒绝推进：orderNo={}, 本地={}, 渠道={}",
               order.getOrderNo(), order.getCurrency(), paidCurrency);
       return;
   }
   ```
8. **价目种子补三套**：`sql\pay_init.sql` 的 `sys_pay_price` 种子按「服务 + 币种」各自独立标价，CNY / USD / HKD 各一套（证据里每个币种各 5 条）。最小单位：USD/CNY/HKD 均为 2 位小数。
   ```sql
   -- 5. 种子价目：三币种各自独立标价（最小单位：USD/CNY/HKD 均为 2 位小数）
   INSERT INTO sys_pay_price (service_id, currency, unit_amount) VALUES
       ('sg_scope',      'USD',   990),
       ('third',         'USD',  1990),
       ('ethics_review', 'USD',  1490),
       -- ... 其余 USD 服务，以及 CNY / HKD 各自的整套价目
   ```
9. **编译验收**：
   ```powershell
   cd 'C:\Develop\project\far-manu-server'
   mvn -pl far-manu-payment -am compile 2>&1 | Select-String -Pattern "BUILD|SUCCESS \[|FAILURE|ERROR\]" | Select-Object -Last 6
   ```
   必须看到 `BUILD SUCCESS`（证据里 Reactor 为 4 个模块全部 SUCCESS）。
10. **三币种真实建单实测**：按该服务在 Nacos 里的配置（`nacos/v1/auth/login` 取 token 后读配置）拿运行参数，对 CNY / USD / HKD 各发起一次真实建单，逐条确认返回 `int_*` 意图号与 `REQUIRES_PAYMENT_METHOD`，且金额是该币种价目换算后的值（证据基线：`CNY amount=68`、`HKD amount=78`、`USD amount=9.9`）。
    - TODO：证据只保留了脚本片断，实际建单接口路径与鉴权头以本仓当前 controller / 脚本为准，不要凭猜测拼 URL。
11. **对外口径**：把「每个币种独立标价、不依赖基准币种，产品定不定基准都不影响这套表」讲清楚，用户对汇率的顾虑由此消解；后续若真的接汇率，是另一件事。

## 只需补回调币种校验时（缩小范围）

若用户只问「多币种下回调校验够不够」：跳到步骤 1 取证 `PayWebhookAppService` 现有校验，明确指出**只校验金额不足**，多币种下必须补币种一致性校验；按步骤 7 改，按步骤 9 编译收口。不要顺手重构鉴权或回调接收链路。

## Pitfalls

- **只改默认币种常量不够**：只留 `DEFAULT_CURRENCY = "USD"` 而不做 `SUPPORTED_CURRENCIES` 白名单，等于入参仍可传任意币种并落库。白名单和默认值要一起加。
- **`Set` 的 import 漏补**：`PayConstants` 原本没有 `java.util.Set`，改完必须补 import，否则编译直接挂。
- **回调只比金额**：多币种下不同币种可能有相同数值金额，只比金额会把币种不符的回调当成成功。必须 `equalsIgnoreCase` 比币种，且 `log.error` 后 `return`、不改状态。
- **缺价时静默回退**：某服务在某币种没配价目时回退到 USD 价或跳过该服务，会让用户用错价格下单。必须整单拒绝并指名服务。
- **引入汇率折算**：本改造刻意不做汇率与基准币种；任何「乘个系数」的写法都违背口径，也把产品和运营绑死。
- **金额精度写死两位**：用 `Currency.getDefaultFractionDigits()`，别假设所有币种都是 ×100。
- **种子只补一半**：`pay_init.sql` 只加 CNY 或只加 HKD，会导致实测时某币种建单被整单拒绝。三套一起补。
- **越界改别的链路**：鉴权/token/createIntent 金额单位属于 `airwallex-sandbox-auth-spring-feign`，回调接收侧验签/幂等属于 `airwallex-webhook-receive-local-test`，本轮不要动。

## Verification

1. **编译**：步骤 9 的命令输出 `BUILD SUCCESS`，Reactor Summary 里 `far-manu-payment` 为 `SUCCESS`。
2. **价目种子**：库中 `sys_pay_price` 对每个受支持服务都有 CNY / USD / HKD 三行，无缺币种服务。
3. **实时建单**：三币种各建一单，均返回 `int_*` 且状态 `REQUIRES_PAYMENT_METHOD`，金额分别对应各自币种价目——基线 `CNY 68 / HKD 78 / USD 9.9`。
4. **回调校验**：构造一个币种与订单不一致的 webhook payload（如订单 USD、payload CNY），确认日志出现「回调币种与本地订单不一致，拒绝推进」且订单状态未推进；再用币种一致的 payload 确认正常推进。
5. **入参校验**：传 `currency=EUR` 或空串应被拒绝；不传 `currency` 应按 USD 建单。

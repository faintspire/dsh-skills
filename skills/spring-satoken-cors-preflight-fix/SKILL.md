---
name: spring-satoken-cors-preflight-fix
description: Diagnoses and fixes Spring Boot + Sa-Token applications where CORS OPTIONS preflight requests are blocked by SaInterceptor and surface as NotLoginException 500s with a second stack trace from a forward to /error; use when the user reports OPTIONS 预检 报 NotLoginException / 未能读取到有效 token、Sa-Token 拦截了跨域预检、预检请求 500 转发 /error，或改了 SaTokenConfig 后错误依旧.
whenToUse: 浏览器预检 OPTIONS 请求在 Spring Boot + Sa-Token 项目里被登录拦截器拦下（日志出现 doOptions → StpUtil.checkLogin → NotLoginException，随后 500 forward 到 /error 再抛一次）；带 token 的正常请求不受影响；或用户已经改过 SaTokenConfig 但线上行为没变，需要确认 class 是否真的重新编译。
---

# Spring Boot + Sa-Token 预检（OPTIONS）被拦截的排查与修复

典型症状：控制台先打印一串 `请求方法: OPTIONS, 路径: /xxx`，紧接着
`Servlet.service() ... threw exception [... NotLoginException: 未能读取到有效 token]`，
再跟一段 `StandardHostValve.throwable` → `forward(/error)` 的第二段堆栈。带 token 的正常请求完全正常。

根因链（三段叠加，缺一不可）：

1. Spring MVC 处理预检时会给 `AbstractHandlerMapping` 的链套上 `PreFlightHandler`，但 `getCorsHandlerExecutionChain` 会把已有拦截器**保留**在链上，所以 `SaInterceptor.preHandle` 照样执行 → 没 token → `NotLoginException`。
2. 此时 handler 是 `PreFlightHandler` 而不是 `HandlerMethod`，`ExceptionHandlerExceptionResolver` 直接返回 null，`@RestControllerAdvice` 接不住，异常穿透成 500。
3. 500 转发 `/error` 又被同一个拦截器拦一次，第二段堆栈盖掉原始异常，误导排查方向。

修复形态固定：在 Sa-Token 路由匹配里 `notMatch(SaHttpMethod.OPTIONS)`，并同时 `notMatch("/error")`；
不要靠在配置类里注入 `HttpServletRequest` 做 preHandle 特判。

## Steps

1. **确认症状与调用链**。在日志里找到 `doOptions` → `StpUtil.checkLogin` → `NotLoginException`，并确认后面那段 `forward(/error)` 堆栈是从同一次请求转发出来的（同一 traceId），不是两个独立 bug。

2. **定位 Sa-Token 路由配置**。按文件名与注解找拦截器注册点：

   ```bash
   grep -rn "SaInterceptor\|SaRouter\|addInterceptors" --include=*.java .
   ```

   常见落点：`**/security/SaTokenConfig.java`（`WebMvcConfigurer.addInterceptors` 里
   `registry.addInterceptor(new SaInterceptor(...)).addPathPatterns(...)`）。
   同时确认异常处理器（如 `GlobalException.java` / `@RestControllerAdvice`）的实际形态，以及
   `application.yml` 里的 `sa-token.token-name`（本项目为 `Authorization`）——预检请求不带任何 token，配置怎么改都不会放行。

3. **先判定“改动到底有没有编译进产物”**（最容易误判的一步）。对比源码与 `target/classes` 里 class 的修改时间：

   ```bash
   ls -l --time-style=full-iso \
     src/main/java/<pkg>/security/SaTokenConfig.java \
     target/classes/<pkg>/security/SaTokenConfig.class
   ```

   如果源码时间明显晚于 class（例如源码 13:29:59、class 仍是 11:50:36），那么用户贴的报错来自**旧构建**；
   必须先重新编译再判断修复是否有效，否则会把“没生效”误诊成“方案不对”。

4. **（可选，老版本才需要）确认当前 Sa-Token 是否支持该重载**。`notMatch(SaHttpMethod...)` 在 1.39.0 已存在：

   ```bash
   javap -p -c -classpath ~/.m2/repository/cn/dev33/sa-token-core/<version>/sa-token-core-<version>.jar \
     cn.dev33.satoken.router.SaRouterStaff \
     | sed -n '/notMatch(cn.dev33.satoken.router.SaHttpMethod/,/areturn/p'
   ```

   同时可确认 `check(SaFunction)` 受 `isHit` 保护（未命中的路由不会执行登录校验），所以 `notMatch` 放行是安全的。

5. **改配置：保留原有 match 规则，只叠加两个 notMatch**。

   ```java
   @Configuration
   public class SaTokenConfig implements WebMvcConfigurer {
       @Override
       public void addInterceptors(InterceptorRegistry registry) {
           registry.addInterceptor(new SaInterceptor(handle -> SaRouter
                   .match("/**")                       // 保留项目原有的 match 列表
                   .notMatch(SaHttpMethod.OPTIONS)     // 放行 CORS 预检
                   .notMatch("/error")                 // 避免 500 forward 被二次拦截
                   .check(r -> StpUtil.checkLogin())))
               .addPathPatterns("/**");
       }
   }
   ```

   需要额外 import `cn.dev33.satoken.router.SaHttpMethod`。

6. **编译改动的模块**（多模块项目带上 `-am`）：

   ```bash
   mvn -B -q -pl biovisart-common -am -DskipTests compile
   ```

   `EXIT=0` 后再继续；编译错误先修完，不要跳过去做验证。

7. **反编译已编译的 class，确认改动真的进产物**：

   ```bash
   javap -p -c biovisart-common/target/classes/com/sangerbox/biovisart/security/SaTokenConfig.class \
     | grep -E "lambda|SaHttpMethod|OPTIONS|HttpServletRequest" | head -12
   ```

   期望看到 `String OPTIONS`、`class cn/dev33/satoken/router/SaHttpMethod`，以及
   `lambda$addInterceptors$1` 这类由 lambda 编译出的私有方法；**不应**再出现 `HttpServletRequest`。

8. **全反应堆构建，确认没有破坏其他模块**：

   ```bash
   mvn -B -DskipTests package 2>&1 | tail -12
   ```

   每个模块都要 `SUCCESS`。

9. **（可选）运行时复核**。重启应用后发一次预检，期望返回 CORS 响应头而不是 500，且日志不再出现
   `NotLoginException`（端口按项目实际替换）：

   ```bash
   curl -i -X OPTIONS 'http://localhost:<port>/bvaArtworks/page' \
     -H 'Origin: http://localhost:5173' \
     -H 'Access-Control-Request-Method: GET'
   ```

## Pitfalls

- **改了源码仍旧报错，先怀疑没编译**：源码 mtime 与 `target/classes` mtime 一对就露馅（见步骤 3）。用户贴的日志时间早于源码修改时间时，那段堆栈根本不反映新代码。
- **不要用注入 `HttpServletRequest` / 手写 `request.getMethod().equals("OPTIONS")` 的方式特判**：初始方案曾往配置类里注入 request，最后收敛为 Sa-Token 自己的 `SaRouter` 匹配；路由层表达放行规则更短、无额外依赖，也不用担心 config 被非请求线程调用。
- **只放行 OPTIONS 不够**：不排除 `/error`，任何 500 转发仍会被拦第二遍并盖掉原始异常，看起来像“改了没用”。
- **别指望在 `@RestControllerAdvice` 里 catch `NotLoginException` 兜住预检**：此时 handler 是 `PreFlightHandler` 而非 `HandlerMethod`，`ExceptionHandlerExceptionResolver` 返回 null，advice 不会触发，这也正是“500 没有 JSON 响应体”的原因。
- **日志里两段堆栈不要当成两个问题**：第二段是 `/error` forward 引起的重入，修好第 1、3 环后两段会一起消失。
- **用当前环境真实存在的 shell 工具**：直接调用不存在的 `pwsh` 工具会报 `unknown tool`，Windows 下改用会话里实际可用的 `win_bash`/`bash` 执行同样的命令。

## Verification

- `mvn -pl <module> -am -DskipTests compile` 退出码 0；`mvn -B -DskipTests package` 所有模块 `SUCCESS`。
- `javap -p -c target/classes/.../SaTokenConfig.class | grep -E "OPTIONS|SaHttpMethod"` 能命中 `String OPTIONS` 与 `SaHttpMethod` 常量，且不再出现 `HttpServletRequest`。
- 运行中应用对同一路径的 OPTIONS 请求返回 200 + `Access-Control-Allow-*` 响应头，日志里既没有 `NotLoginException`，也没有 `forward(/error)` 的第二段堆栈；带 token 的 GET/POST 仍然正常鉴权（未登录访问受保护接口依旧被拒）。

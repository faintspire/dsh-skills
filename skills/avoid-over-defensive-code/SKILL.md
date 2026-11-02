---
name: avoid-over-defensive-code
description: 约束 Java 代码不要过度防御：判定什么算过度防御、该在哪一层兜底、哪些检查是冗余的、已有工具类不要重造（现成工具类只有文件版 API、而内部已先拿到 byte[] 时，抽共享取字节核心并加内存版方法，而不是跟着落盘或复制一份压缩逻辑）、不要为假设的扩展性预建 properties 配置类/record/独立 service/缓存层，给出存量脏数据兼容这一合法防御的写法，并给出删掉冗余判空后的收尾（grep 零引用 → 删死工具类与失效 import → 编译验收）。当用户要求写或改 Java 业务代码、新增 service/接口/异步任务、做 code review，或提到"过度防御""不要多写一套""不要多余的分层""抽象层""可扩展""兜底""简化""脏数据兼容""临时文件""内存版""落盘""这个不要再判空了""零引用""死代码"时使用。
whenToUse: 当需求本身很小（判一个邮箱后缀、加一个开关、算一份配额）但方案开始膨胀成配置类 + record + 独立 service + 缓存层时；或用户说"直接做一个工具类就行""先做简单点""不要那么多层"时；也用于判断哪些兜底该删、哪些必须留；也用于发现"照着现成工具类的文件版方法写，结果多了一次落盘、一次二次解码和一段 finally 清理"时（见第二节的文件版/内存版判定）；也用于用户明确说"这些字段不是必传、不要再判空了"之后做收尾时（grep 零引用、删死工具类、清失效 import、跑 Maven 编译，见第六节）。
---

# 不过度防御

写代码前先问四件事，前三条任何一件答"是"就停下来改：

1. **这个兜底，上层是不是已经有了？** 有就删掉这层。
2. **这个工具/组件，项目里是不是已经有？** 有就直接用，不要手写。若它只暴露文件导向 API、而你要的是字节，别照着落盘 —— 抽共享核心 + 加内存版（见第二节）。
3. **这个判断，调用链上游是不是已经保证过了？** 保证过就不要再判。删掉之后还会连带产生死工具类与死 import，收尾见第六节。
4. **这个类/接口/配置/缓存，是为眼下这一处调用建的，还是为"以后可能扩展"建的？** 是后者就先只写一个静态工具类（见第三节）。

---

## 一、兜底只能放在边界

| 层 | 谁兜底 |
|---|---|
| HTTP | `GlobalException`（`@RestControllerAdvice`）统一处理异常 |
| WebSocket | 策略类（`MessageStrategy` 实现）负责推帧、进度、失败 |
| Service / Provider / Util | **只 throw，不 catch 转默认值**（唯一例外：存量脏数据兼容，见第五节） |

据此可以直接判断：

- service 里 `catch (Exception e) { throw new RuntimeException("xxx失败：" + e.getMessage(), e); }` —— 两层包装，边界已经会兜，删掉。
- service 里 `catch (Exception e) { return null; }` —— 把"出错"和"没数据"混成一个值，调用方无法区分；只有缓存/降级场景才有理由，且要注释说明。
- WebSocket 策略里已经 `catch` 推了失败帧，外层 `CompletableFuture.exceptionally` 就**只打日志**，不要再推一次。

## 二、不要重复造已有的东西

动手前先 grep 有没有现成实现：

- 推 WebSocket 帧 → `WebSocketResponseUtil`（`sendError` / `sendProgress` / `sendDrawingResult` / `sendDrawingFailure` / `sendDrawingRejected`），不要在业务里 `new JSONObject()` 拼帧再 `NettyChannelManager.sendMessageToUser`。
- TraceId → `TraceIdUtil`；分页 → `PageUtils`；文件路径 → `PathUtil`；图片下载/缩略图 → `ImageDownloadUtil`。
- 请求构造/解析 → 对应 Provider（`DoubaoImageProvider` / `QwenImageProvider`）与 `DashScopeRequestBuilder`。

> WebSocket 策略里取消息字段：`WebSocketRequestUtil`（`requireText`）已按第六节删除，**不要重建也不要重新引入**；`userId`、`prompt` 等一律 `requestJson.getString(...)` 直接取，缺值由 `MessageStrategyManager.processTextMessage` 的 catch 兜住推错误帧。

判据：**同样的协议/格式在仓库里出现第二遍，就是错的。**

### 现成工具类只有文件版 API 时：抽共享核心 + 加内存版，别跟着落盘

要分清"工具类的签名"和"这次的需求"。签名收的是 `String targetPath` / `File targetFile`，不代表需求就是要文件。

1. **看签名**：现成方法收路径/文件，而调用方真正要的是 `byte[]`（推对象存储、算 md5、继续交给下一个 API）→ 这次落盘是签名带出来的，不是设计意图。
2. **看内部**：进方法体确认它是不是本来就先得到 `byte[]` 再写文件（先压缩成字节、再写到文件；缩略图再从**那个文件** `ImageIO.read(...)` 读回来）。是 → 字节流一直在手上，调用方先落盘再读回来，纯属绕路。
3. **抽核心**：把"取字节"那段抽成一个方法（如 `compressToBytes`），文件版改成"调 core + 写文件"，原签名与原行为保持不变。
4. **加内存版**：新增一个直接返回 `byte[]` 的公开方法，内部同样调 core。落盘路径与内存路径共用同一份压缩/解码逻辑。
5. **调用方切过去**：删掉临时文件变量与 `finally` 里的删除，改成内存直推；md5、两级目录命名这类原有逻辑照旧保留，不要顺手改。
6. **不做的事**：不在调用方复制一份压缩逻辑；也不因为"文件版看起来多余"就删掉它 —— 还有调用方就保留成薄封装。

改工具类签名或新增公开方法前，先扫全仓调用点确认影响面与兼容性，例如：

```bash
rg -n 'downloadAndGenerateBoth|generateThumbnail\(|downloadAndCompressImage\(' --glob '*.java'
rg -n 'downloadAndGenerateBoth|generateThumbnail\(|downloadAndCompressImage\(' <file>
```

什么时候不适用：真的只有一次调用、且结果本来就要落盘给别处读（例如必须返回文件路径给前端下载），直接用文件版就好，不必为单次调用硬造内存版。

## 三、不要为假设的扩展性预建抽象层

判据：**这条判定只有一个调用方、一份常量、没有 IO 时，就只写一个静态工具类。**

本项目真实教训（公司邮箱生图配额）：首版方案为了"以后可能扩展"，同时新增 `DrawingQuotaProperties`（配置类）+ `DrawingQuota`（record）+ biz 层 `QuotaService` + Redis 缓存层，共 6 个文件改动；真实需求只是"邮箱后缀命中即不限次"。最终这些全部删掉，净新增 1 个文件。

只写静态工具类就够的信号：

- **规则是纯函数**：输入（email / userId）→ 输出（`unlimited(COMPANY_EMAIL)` / `limited(50, ADMIN)` / `limited(5, NORMAL)`），无 IO、无事务、无状态。
- **数据是常量**：域名集合、阈值、白名单。"域名会不断增加"= 往 `Set` 里加一行字符串，不需要配置类，更不需要为三态结果专门定义一个 record。
- **数据已经在登录态里**：Sa-Token 会话里已有当前用户邮箱，就不要再新增 Mapper / 查询 service / 缓存层去查库。项目已有的登录态入口（如 common 里的 `SecurityUtil`）直接复用，写法和它保持一致。

什么时候才升级（必须有一条真实需求，不是"以后可能"）：

- 出现第二个调用方且需要不同实现 → 才抽接口（不要先抽接口等调用方）。
- 需要不发版就改（域名白名单由运营在 Nacos/DB 里配）→ 才上 `@ConfigurationProperties`。
- 实测有性能问题、且跨请求重复计算 → 才加缓存层。

放哪里：`biovisart-common/src/main/java/com/sangerbox/biovisart/util/`，与 `SecurityUtil` 同模块同风格 —— 纯静态类 + `@Slf4j` + 类头 Javadoc，**不依赖 Spring 容器**，biz 和 netty 模块都能直接调。

两件必须放进取值逻辑里的事：

1. **优先级顺序写在工具类里**，不要散到调用方：公司邮箱 > 管理员角色。顺序反了，"公司员工恰好是 admin"就会掉回 50 次。
2. **域名取 `@` 之后整段做 `equals`**，不要用 `endsWith(domain)` —— 后者会放行 `a@sangerbox.com.evil.com`。

> TODO: 项目里从 Sa-Token 取邮箱的确切入口（`SecurityUtil` 的哪个方法）请在落地时按现有用法确认，不要照抄示例方法名。

## 四、异步线程里不要再开线程

业务已经在异步线程（`@Async`、`CompletableFuture.runAsync`、虚拟线程）里执行时：

- 不要再 `Executors.newSingleThreadScheduledExecutor` 做进度心跳或超时兜底；
- 不要再 `CompletableFuture.supplyAsync(...)` 然后自己 `.get()` 阻塞等它（等于白占一个线程池 worker）；
- 需要定时推送/硬超时，用**全局共享**的 `ScheduledExecutorService` Bean，任务结束 cancel。

超时优先交给 HTTP 客户端（Feign 的 `readTimeout`）和上游自身，不要在业务里再套一层。

---

## 五、存量脏数据兼容（唯一允许 service catch 转默认值的形态）

历史版本把数据写坏了（例如早期新建作品时把前端占位串 `"content"` 原样写进对象存储，只有 7 字节），按正常路径读取必然失败。这类"对象根本不是本工具产出的内容"，允许在读取侧降级为空白，但必须同时满足 4 条：

1. **类型上可区分**：加解密/解析工具（如 `DrawioCompressorUtil`）对"不是本工具产出的内容"单独抛一个子类，例如 `InvalidStoredContentException extends IllegalStateException`。触发条件要具体可判定（长度低于最小密文长度、头部不是预期格式），不是 catch 之后靠猜；解密 padding 错、解压失败、IO 错仍抛原异常。
2. **catch 收窄到这个子类**：`catch (InvalidStoredContentException e)`，不写 `Exception` / `RuntimeException` / 父类；catch 块里注释写清脏数据来源和降级后的用户可见行为（"作品还能打开，显示空白画布"）。说不清"哪些数据会进这个 catch"，就说明不该 catch。
3. **降级只给一个默认值**：返回项目已有的空白画板常量（如 `DEFAULT_DIAGRAM_CONTENT`），不要顺手返回 `null`；真正的解密/解压失败继续向上抛，由边界报错，不能被吞成空白。
4. **同一次提交堵住写入口**：写入口校验内容确实是 drawio/SVG XML，不是就复用同一常量（如 `resolveInitialContent`），不把占位串落库；格式判断复用 util 已有扫描逻辑，不新写一套。只修读取侧不修写入侧，脏数据会继续产生，不算修完。

---

## 六、去掉冗余判空后的收尾：死工具类、死 import、编译证据

命中第 3 条（"上游已保证就不要再判"）时，典型改法是把整段取参 + 判空删掉：

```java
// ❌ 上游 catch 已把缺值兜成错误帧，这两段都是多余的
String userId = WebSocketRequestUtil.requireText(ctx, requestJson, "userId");
if (userId == null) {
    return;
}

// ✅ 直接取值
String userId = requestJson.getString("userId");
String prompt = requestJson.getString("prompt");
```

删掉的不只是这几行 —— 取参工具可能因此变成零引用。收尾按下面顺序做，一步都不能省：

1. **先声明范围，再动手**：同一个写法往往不止一处（本次三个策略类都是 `requireText` + `if (x == null) return`）。改前先 grep 该写法在全仓的分布，把所有同类点一起清理，别只改被点名的那一处。
2. **改完立刻 grep 引用面**：

   ```bash
   rg -n 'WebSocketRequestUtil' --glob '*.java' .
   ```

   剩下的命中只应出现在工具类自身文件里（类声明、私有构造器）；别的文件还有命中，就是有调用点没改，回去补。
3. **确认零引用才删文件**（Windows PowerShell）：

   ```powershell
   $p = "<仓库>\biovisart-netty\src\main\java\com\sangerbox\biovisart\netty\util\WebSocketRequestUtil.java"
   if (Test-Path -LiteralPath $p) { Remove-Item -LiteralPath $p -Force; "deleted: $p" } else { "not found: $p" }
   ```

   还有一处引用就不删，留着。别用通配符/递归删除，用 `-LiteralPath` 指到具体文件。
4. **清掉失效符号**：删掉每个被改文件里的 `import ...WebSocketRequestUtil;`，再逐个文件扫一遍因删除调用而变成未使用的 import（本次是 `java.util.List`、`com.sangerbox.biovisart.vo.AiGalleryVo`）和局部变量。注意区别：未使用的 import 不影响编译但纯属噪音；**引用了已删类的 import 会直接编译失败**。若方法参数 `ctx` 只剩传参用途，先确认策略接口是否强制该签名 —— 是则保留并在汇报里说明，不是才删。
5. **编译验收**：

   ```bash
   mvn -pl biovisart-biz,biovisart-netty -am compile 2>&1 | Select-String -Pattern "BUILD|SUCCESS \[|FAILURE|ERROR\]" | Select-Object -Last 15
   ```

   末态必须是 `BUILD SUCCESS`，且 Reactor Summary 里每个改动模块都是 SUCCESS；用 `Select-String` 抽摘要而不是看 exit code。**别加 `clean`** —— 本仓 Windows 环境下 `clean` 会报 `Failed to clean project: Failed to delete ...\target\...`，与本次代码改动无关，绕它没有意义（Maven 细节见 `maven-multimodule-compile-verify-windows`）。
6. **汇报形态**：按「文件:行号 + 改动要点 + 编译结果」逐文件列成表格，最后一行给出实际执行的编译命令与 `BUILD SUCCESS`。

分工边界：本节只管"因删冗余判空而**新产生**的死工具类/死 import"；"整类死代码（接口 + 唯一实现 + 注入点）的删除与零引用验收"走 `java-dead-code-removal-verify`。

---

## 写完后过这 19 条

- [ ] 每个 `== null` 判断，都能说出"谁可能传进来 null"
- [ ] 每个 `catch` 都有上层没有的处置动作（打点/降级/转换），否则删掉
- [ ] 没有 `catch` 后抛新 `RuntimeException` 的纯包装
- [ ] 没有 `catch { return null; }`（缓存降级除外，且注释说明）
- [ ] `x != null && x.getY() != null` 的双重判空，确认不是上游已保证
- [ ] 校验只做一次：DTO 校验 / 层内重复校验只保留最外层一处
- [ ] 空结果用"推失败帧 + return"，而不是 `throw` 后被自己的 catch 接住
- [ ] 同一份换算/拼装逻辑只存在一处（尺寸、帧、路径）
- [ ] 没有为一次调用新建线程池/调度器
- [ ] 新增的工具方法，仓库里没有等价实现
- [ ] 现成工具类只有文件版 API 时，先确认它内部是否已拿到 `byte[]`：是则抽共享核心 + 新增内存版方法，没在调用方复制一份压缩逻辑
- [ ] 调用链上没有为拿字节新建的临时文件与 `finally` 删除（内存版能覆盖就不落盘），也没有从刚落盘的文件再 `ImageIO.read` 读回来
- [ ] 本次改动里没有一个文件是为"以后可能要扩展"建的（properties 配置类 / record / 独立 service / 缓存层）
- [ ] 域名、阈值、开关这类纯判定常量写在工具类里，没有为它单开配置类或缓存层
- [ ] 存量脏数据兼容只 catch 专用异常子类（如 `InvalidStoredContentException`），不是 `Exception` / 父类，且注释写明脏数据来源
- [ ] 写入口不落占位串/非目标格式内容，复用已有常量（如 `DEFAULT_DIAGRAM_CONTENT`），没新写一份
- [ ] 删掉判空后，原来的取参辅助方法/工具类已 grep 确认零引用，且文件已删除（不是留着"以后可能用"）
- [ ] 因删除调用而失效的 import、局部变量、未使用参数都清掉了；引用了已删类的 import 一处不剩
- [ ] 末态证据是 `mvn -pl <改动模块列表> -am compile`（**不带 clean**）的 `BUILD SUCCESS`，每个改动模块在 Reactor Summary 里都是 SUCCESS

## 该保留的防御（别删过头）

- **外部输入 + 历史脏数据**：HTTP body、WebSocket 消息、文件上传、URL 参数 —— 必须校验；对象存储/DB 中历史版本写坏的脏对象（不是本工具产出），读取侧按第五节用专用异常子类窄 catch 降级，别当成过度防御删掉。
- **第三方返回结构**：上游字段可能缺、可能是空数组，取之前判一次是合理的。
- **IO / 文件 / 网络**：确实需要打开流或落盘时必须 `try/finally` 关资源、清理临时文件；但若现成工具类内部本来就拿到了字节，优先按第二节抽内存版 API 直接拿 `byte[]`，从源头不产生临时文件。
- **并发**：共享可变状态的可见性、幂等，不是防御，是正确性。
- **安全**：越权、注入、密钥 —— 不属于本文范围，不能以"过度防御"为名删掉。

---

## 反例 → 正例（都来自本项目真实代码）

### 1. 抛异常又被自己接住

```java
// ❌ 打一条 error 堆栈，再走 catch 推失败，等于自己套自己
if (result == null || result.getImageUrls() == null || result.getImageUrls().isEmpty()) {
    throw new IllegalStateException("上游未返回图片");
}
// ✅ 空结果就是业务失败，直接推帧返回
List<String> urls = result == null ? null : result.getImageUrls();
if (urls == null || urls.isEmpty()) {
    log.warn("上游未返回图片, userId={}", userId);
    WebSocketResponseUtil.sendDrawingFailure(channelKey, "failed", "AI 绘图失败，请重新再试");
    return;
}
```

### 2. 死掉的 catch

```java
// ❌ AiService.text 内部已吞掉异常返回 null，这个 catch 永远不会进
try {
    raw = aiService.text(...);
} catch (Exception e) {
    return true;
}
// ✅ 只处理"没有结果"这一种真实情况
String raw = aiService.text(...);
if (raw == null || raw.isBlank()) { return true; }
```

判断依据：**先看被调用方法是否本来就吞异常/返回哨兵值**，再决定要不要包 try。

### 3. 同一个换算做两遍

```java
// ❌ service 算一次尺寸塞回 DTO，provider 里又算一次
request.setSize(QwenImageSizeUtil.of(request.getSize(), request.getRatio(), request.getTier()));
AiImageCallResult call = qwenImageProvider.generate(imageRequest);

// ✅ 换算只在 provider 一处，service 用返回值
AiImageCallResult call = qwenImageProvider.generate(imageRequest);
boolean is2K = QwenImageSizeUtil.is2K(call.getSize());
```

### 4. 手写一套已有的推送

```java
// ❌ 业务里自己拼帧
JSONObject push = new JSONObject();
push.put("type", "ai_drawing_result");
push.put("status", "completed");
NettyChannelManager.sendMessageToUser(channelKey, push.toJSONString());

// ✅ 用工具类
WebSocketResponseUtil.sendDrawingResult(channelKey, data);
```

### 5. 存量脏数据：专用子类窄 catch，而不是一把 catch

```java
// ❌ 读取侧 catch(Exception) 一把梭：真正的解密失败也被吞成空白画布，坏数据永远发现不了
try {
    return DrawioCompressorUtil.decryptAndDecompress(rawBytes);
} catch (Exception e) {
    log.warn("读取作品内容失败", e);
    return DEFAULT_DIAGRAM_CONTENT;
}

// ✅ 工具类只对"不是本工具产出"抛专用子类
// DrawioCompressorUtil
if (stored.length < 24) { // 历史 Base64 至少 16 B 密文 → 24 B
    throw new InvalidStoredContentException("存储对象不是本工具产出的内容, length=" + stored.length);
}

// BvaArtworksServiceImpl：只接这一种，注释写清原因
try {
    decryptedContent = DrawioCompressorUtil.decryptAndDecompress(rawBytes);
} catch (InvalidStoredContentException e) {
    // 存量脏对象：早期新建作品时把前端占位串（如 "content"）原样写进了对象存储，
    // 解不出来。按空白画布返回，作品还能打开；真正的解密/解压失败不在此类型，仍向上抛。
    log.warn("作品内容不是本工具产出，按空白画布返回, artworkId={}, size={}", artworkId, rawBytes.length);
    decryptedContent = DEFAULT_DIAGRAM_CONTENT;
}

// ✅ 写入口：不是 drawio/SVG XML 的占位串不落库，复用同一个常量
String content = resolveInitialContent(dto.getContent());
```

### 6. 为"以后要扩展"预建 6 个文件

```java
// ❌ 只是为了判定一个邮箱后缀，却建了一套分层
// DrawingQuotaProperties(配置类) + DrawingQuota(record) + biz 层 QuotaService + Redis 缓存层
//   共 6 个文件改动，唯一判定逻辑被拆到三处

// ✅ 一个纯静态工具类：common 的 util 包，和 SecurityUtil 同风格，biz/netty 都能直接调
@Slf4j
public class CompanyEmailUtil {
    private static final Set<String> DEFAULT_DOMAINS = Set.of("sangerbox.com", "shengxin.ren");

    /** 公司邮箱判定，优先级高于管理员角色 */
    public static boolean isCompanyEmail(String email) {
        int at = email == null ? -1 : email.lastIndexOf('@');
        return at >= 0 && DEFAULT_DOMAINS.contains(email.substring(at + 1));  // 整段 equals，不用 endsWith
    }
}
```

加一个公司域名 = 往 `DEFAULT_DOMAINS` 加一行字符串；邮箱从 Sa-Token 登录态取，不查库。等真的出现"不发版就要改白名单"的需求，再加配置类也不迟。

### 7. 现成工具类只有文件版 API，就跟着落了两次盘

```java
// ❌ 工具类 API 全是文件导向，照着写：写原图 → 再从该文件读回来生成缩略图 → 再读成 byte[] 推对象存储 → finally 删临时文件
String tmpPath = ...;
ImageDownloadUtil.downloadAndGenerateBoth(imageUrl, tmpPath); // 缩略图由 generateThumbnail(File, ...) 内部 ImageIO.read(sourceFile) 生成
byte[] full = Files.readAllBytes(Path.of(tmpPath));
// ... 推 RustFS
// finally { Files.deleteIfExists(Path.of(tmpPath)); }
```

```java
// ✅ 压缩逻辑内部本来就先得到 byte[]：抽共享核心，文件版与内存版都调它
// ImageDownloadUtil
/** 取字节核心：压缩逻辑只此一份 */
private static byte[] compressToBytes(BufferedImage image, /* TODO: 参数按实际落地的 writeCompressedImage 签名确认 */ ...) {
    // 原 writeCompressedImage 里"先压缩再写文件"的前半段
}

// 文件版：保持原签名，改成"调 core + 写文件"
public static long writeCompressedImage(BufferedImage image, File targetFile, ...) {
    byte[] bytes = compressToBytes(image, ...);
    // 只负责把 bytes 写进 targetFile
}

// 内存版：新增，调用方直接拿字节
// TODO: 公开方法名与参数按落地时确认（如 downloadAndCompressImageToBytes / generateThumbnailBytes）

// AiDrawingHistoryServiceImpl.recordFromUrl：内存里生成原图与缩略图，直推 RustFS
byte[] full  = ImageDownloadUtil.downloadAndCompressImageToBytes(imageUrl);
byte[] thumb = ImageDownloadUtil.generateThumbnailBytes(full, ...);
// 没有临时文件、没有二次解码、没有 finally 清理；md5 与两级目录命名逻辑照旧
```

### 8. 判空删了，取参工具类却留成死代码

```java
// ❌ 只删了调用处的判空，requireText 从此零引用，WebSocketRequestUtil 变成没人用的类
String userId = requestJson.getString("userId");
// WebSocketRequestUtil.java 还在，import 也还挂着
```

```java
// ✅ 三步收尾（第六节）：grep 零引用 → 删类 + 清 import → 编译验收
// 1) rg -n 'WebSocketRequestUtil' --glob '*.java' .   → 只剩工具类自身
// 2) Remove-Item -LiteralPath ...\netty\util\WebSocketRequestUtil.java -Force
//    三个策略类同时删掉 import ...WebSocketRequestUtil;
//    以及因此变成未使用的 java.util.List / com.sangerbox.biovisart.vo.AiGalleryVo
// 3) mvn -pl biovisart-biz,biovisart-netty -am compile  → BUILD SUCCESS
```

---

## 快速扫描

改完代码，扫一下本次改动文件：

```bash
rg -n 'catch \(Exception' <file>          # 每个都要能说出处置动作
rg -n 'catch \(' <file>                   # 逐个确认 catch 类型：存量脏数据只允许专用子类
rg -n 'return null;' <file>               # 是否把错误和空混成一个值
rg -n '!= null &&' <file>                 # 双重判空是否必需
rg -n 'newSingleThreadScheduledExecutor|supplyAsync.*\.get\(' <file>   # 异步里再开线程
rg -n 'createTempFile|deleteIfExists|Files\.readAllBytes' <file>       # 为拿字节新建的临时文件：先看有没有内存版 API
rg -n 'ImageIO\.read\(' <file>                                        # 是否从刚落盘的文件再读回来解码（二次解码）
rg -n 'WebSocketRequestUtil|requireText' --glob '*.java' .             # 判空删完后是否还有引用（应只剩 0 处）
git diff --stat                         # 净新增多个文件却只服务一条判定 → 回到第三节
```

## 提交前的自述

在回复用户时用一句话说明：这次的防御放在哪一层、删掉了哪些重复兜底、这次一共新增了几个文件（多于一个就说明为什么必须多于一个）。说不清楚，就是还没想清楚。如果做了存量脏数据兼容，再补一句：脏数据是哪一版写入口产生的、这次怎么堵住。如果为了"省一次落盘"改了工具类，再补一句：抽出的共享核心叫什么、文件版是否仍是薄封装、原调用方是否都还能编译。如果删掉了因判空移除而变成零引用的工具类，也补一句：grep 结果如何、删了哪个文件、`mvn -pl <改动模块列表> -am compile` 是否 `BUILD SUCCESS`。

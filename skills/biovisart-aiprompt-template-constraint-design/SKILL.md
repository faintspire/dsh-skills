---
name: biovisart-aiprompt-template-constraint-design
description: "改造 biovisart-server 的 AI 绘画提示词模板与场景约束组装：先取证合规审核的执行顺序与不拼约束的旁路，再把约束清单换成单一 %s 槽位模板并同步全部调用点为 buildPrompt；用户说「提示词模板要改」「用户文字要嵌进素材边界」「ARTWORK 那套也定下来一起改」「这条约束还要不要加进模板」时使用。"
whenToUse: "当用户要新增、修改或定稿 AI 绘画提示词模板与场景约束（MATERIAL / ARTWORK / PUBLIC），询问某条约束是否该写进模板，要求把用户原文嵌进模板，或要求把约束拼装方法统一下来时使用。不覆盖单纯诊断上游请求参数与只改提示词文本（那是 biovisart-doubao-prompt-diagnose 的范围），也不替代纯粹的 Maven 编译验收流程（见 maven-multimodule-compile-verify-windows）。"
---

# AI 绘画提示词模板与约束组装

范围：`biovisart-common` 的模板常量（`AiPromptConstants`）与 `biovisart-biz` 的拼装调用点。
核心原则：先取证现有合规审核与旁路，再定稿模板；默认只出方案，确认后才改代码。

## Steps

1. **定位模板常量与全部调用点。** 在仓库根目录 grep：
   - `PROMPT_REVIEW_TEMPLATE|promptReview|checkPrompt|reviewPrompt` → 定位审核模板常量与调用点（实测 `AiPromptConstants.java:74`、`AiArtworkDrawingStrategy` 等 4 处）。
   - `MATERIAL|ARTWORK|constraintsOf|setScene|getScene` → 列出所有场景取约束的位置。
   - `BIOVISART_SERVER` → 场景判定所依赖的常量。
   没读到具体行号之前不要下结论。

2. **确认合规审核在生图之前、且已覆盖哪些约束。** read `biovisart-biz/src/main/java/com/sangerbox/biovisart/netty/strategy/impl/AiArtworkDrawingStrategy.java` offset=220 limit=45（必要时再 offset=195 limit=40）。记录证据：`:90` 调用 `reviewPrompt`（生图之前），`:226-244` 解析 `pass` 判定，模板 `AiPromptConstants.java:74-86` 已明确覆盖「血腥、暴力、残害虐待……真实人物肖像、真实品牌 logo 或水印」。同时说明其覆盖范围可能只包含部分生图链路。

3. **核对旁路，定稿前把这条证据摆出来。** `NettyConstants.java:14` 定义 `BIOVISART_SERVER = "biovisart"`；`AiInteractionStrategy.java:177-179` 在请求 `server` 为 `null` 时判为 `PUBLIC`；`AiPromptConstants.java:98` 的 `PUBLIC` 分支返回空串。结论：日志里 `server: null` 的那次请求**根本没拼任何约束**，模板只作用于判定到 MATERIAL / ARTWORK 的链路，必须先向用户说明再谈模板内容。

4. **用单一 `%s` 槽位定稿模板。** 组装契约固定为 `最终 prompt = 模板.replace("%s", 用户原文)`，`%s` 全模板只出现一次；禁止把「约束块 + 用户文字」前后硬拼（用户已明确否掉这种拼法）。模板正文里声明优先级：

   | 项 | 谁说了算 |
   |---|---|
   | 题材、风格、配色、构图细节 | 用户原文（模板只声明以画面描述为准，不做额外风格限定） |
   | 背景纯白 / 单一主体居中 / 留白充足 / 可抠图 | 模板强制（仅 MATERIAL 素材场景） |
   | 无文字与色值、无水印商标、单一完整画面、不臆造元素、避免 AI 感 | 模板强制（两个场景都有，只保留这些边界） |

   ARTWORK 定稿模板（可直接沿用，未指定风格时不加兜底句）：

   ```java
   public static final String ARTWORK_PROMPT_TEMPLATE = """
           生成一张完整的成图作品。
           画面描述：%s
           以上画面描述中的题材、风格、配色与构图均以描述为准，不做额外风格限定，仅需满足：
           - 画面自洽，主体与背景层次完整，视觉重心稳定
           - 画面中不出现任何文字、标签、编号、标注或色值；描述里出现的颜色代码仅作配色理解
           - 不出现水印、logo、品牌名、商标、二维码
           - 单一完整画面，不是多格拼图、样机展示或带说明框的示意图
           - 主体结构与描述一致，各部分比例与连接关系合理，不臆造描述中未提及的元素
           - 避免过度光滑的 AI 感与蜡像感，避免肢体与结构畸变
           """;
   ```

   MATERIAL 模板沿用同样的槽位与优先级，额外带上白底均匀无渐变无纹理无场景无投影无装饰、单一主体居中完整不裁切、四周留白充足不画边框与底板、主体与白底边界清晰便于抠图。

5. **去重与泛化。** Step 2 已由 `reviewPrompt` 覆盖的条目（血腥画面、伤口特写、写实可识别人体、真实品牌 logo / 水印）不要再写进模板，重复写入只是重复占权重。场景专属措辞要泛化成对所有用户都成立的说法：把「符合医学与生物学常识」改为「主体结构与描述一致，各部分比例与连接关系合理，不臆造描述中未提及的元素」（站点会有非医学用户，原措辞过窄）。

6. **默认只出方案，零代码改动。** 交付组装契约、完整模板文本、逐条归属表（每行说明性质与保留/删除理由）和旁路说明，每条结论落 `文件:行号` 链接；声明「这轮仍然零代码改动，你确认后再开工」，等用户点名要动手。

7. **一次收敛剩余决策点。** 用 `ask_user_question` 在同一轮问完：未指定风格时是否兜底（不限定 / 补一句默认风格 / 前端给默认值）、是否保留「单一完整画面」限制。拿到答案后才修改模板与代码，不要逐个来回问。

8. **同步落地模板与调用点。** 三处一起改：
   - `AiPromptConstants.java`：两段「规范清单」常量换成带 `%s` 槽位的引导模板；`constraintsOf(PromptScene scene)` 改为 `buildPrompt(PromptScene scene, String userPrompt)`，内部把用户原文填进唯一槽位，`PUBLIC` 分支保持返回用户原文。
   - `AiDrawingServiceImpl.java:63`：`AiPromptConstants.constraintsOf(request.getScene()) + request.getPrompt()` → `AiPromptConstants.buildPrompt(request.getScene(), request.getPrompt())`。
   - `DoubaoDrawingServiceImpl.java:48`：同样改调 `buildPrompt`。

9. **验残留引用零命中。** grep `constraintsOf|ARTWORK_CONSTRAINTS|MATERIAL_CONSTRAINTS`，期望 `No matches found`，把结果原样贴给用户。

10. **clean compile 验收。** 跑带 `clean` 的 Maven 编译（细节按 maven-multimodule-compile-verify-windows 的参数与后台 job 约定），以 `BUILD SUCCESS` + Reactor Summary 各模块 `SUCCESS`、且模块确实重编了源文件（实测 common 56 个、biz 191 个）为验收依据，不看 exit code；最后 `git --no-pager diff -- <三个文件>` 展示改动，并按 `文件:行号` 汇报改动清单。

## Pitfalls

- **重复约束**：没先读 `reviewPrompt` 往模板里加「不出现血腥画面」，与 `AiPromptConstants.java:74-86` 已有条目重复，白占权重。
- **措辞过窄**：「符合医学与生物学常识」按场景写死，非医学用户场景不成立，要泛化成与描述一致性、比例关系、不臆造元素。
- **漏看旁路**：只按代码路径推断「所有请求都拼了约束」，忽略 `server: null` → PUBLIC → 空串这条链路，会把定稿结论建立在错误前提上。
- **前后拼接**：把边界块贴在用户文字前后，模型读到两段不相干的话；必须用单一 `%s` 槽位内嵌。
- **只改常量不改调用点**（或反之）：`constraintsOf` 与新 `buildPrompt` 并存，编译失败或旧路径继续生效；改完必须 grep 零命中。
- **增量编译误判**：不做 `clean` 会出现 `Nothing to compile` 的假通过；在 workspace-write 文件策略下 Maven 可能写不进各模块 `target` 目录报「拒绝访问」导致 BUILD FAILURE，这属于环境限制，换到可写 target 的环境重跑，不要误判成代码错误，也如实说明。
- **未收敛决策点就动代码**：风格兜底、单图限制这类取舍必须先问，否则改完还要回滚。

## Verification

- `grep constraintsOf|ARTWORK_CONSTRAINTS|MATERIAL_CONSTRAINTS` 返回 `No matches found`，并贴出原始结果。
- read 回 `AiPromptConstants.java` 确认模板中 `%s` 只出现一次、优先级声明在文、去重项已删除。
- `ask_user_question` 的两个决策点答案已落到模板文本（未指定风格是否兜底、是否保留单一完整画面）。
- Maven 输出含 `BUILD SUCCESS` 与各模块 `SUCCESS` 的 Reactor Summary，且源文件实际重编而非增量跳过。
- `git diff` 可见「1 个常量类 + 2 个调用点」三处改动，汇报时带到 `文件:行号`。

---
name: web-search-fetch-fallback
description: '在 web_search 因缺少 DEEPSEEK_API_KEY 直接失败、web_fetch 报 cross-origin redirect 或抓取超时的情况下，用固定降级链取证——直接 web_fetch 官方文档、按报错提示的目标 URL 原样重试、需要搜索时抓 https://cn.bing.com/search?q=<urlencoded> 结果页，并让每条结论都附官方文档 URL 与原文句子；当用户说「用官方文档确认」「查一下官方文档」「给出官方文档链接和原文依据」，或检索工具报 no API key / cross-origin redirect / 超时时使用。'
---

# web_search / web_fetch 检索降级链

本会话检索工具不可靠时的固定处置顺序：**先判断是真失败还是可重试**，可重试的只重试一次且必须换 URL，不可重试的立刻换通道。不要在同一 URL、同一工具上循环。

## Steps

1. 先正常用 `web_search(queries=[...])` 检索。若返回类似 `Error: DeepSeek search has no API key for "DEEPSEEK_API_KEY"; ...`，判定为**本会话环境缺凭据**：不要重试，不要换查询词再试，不要连发多条。直接进入第 2 步。给用户配 key 属于凭据服务的另一次操作，不是本技能的降级路径。

2. 立刻改用 `web_fetch(url="<官方文档 URL>")` 直取官方文档站，一次只抓一个 URL。按主题选站：
   - 阿里云百炼 / 通义千问：`https://help.aliyun.com/zh/model-studio/...`
   - 火山方舟 / 豆包：`https://www.volcengine.com/docs/...` 或 `https://docs.volcengine.com/docs/...`
   - Java 框架：`https://docs.langchain4j.dev/...`、`https://docs.spring.io/...`

3. 若 `web_fetch` 返回 `Error: cross-origin redirect to <URL> is not followed automatically; retry against that URL directly`，把报错文本里的那个 `<URL>` **原样**（不改 query、不改参数、不换域名、不加 /）作为下一次 `web_fetch(url="<该 URL>")` 立即重试一次。证据中的两次复现：
   - `https://www.volcengine.com/docs/82379/1541523` → 重试 `https://docs.volcengine.com/docs/82379/1541523`，该次又落到 `https://docs.volcengine.com/docs/ark/image-generation-api?lang=zh`，按最终 URL 再抓即成功。
   - `https://www.bing.com/search?q=...` → 重试 `https://cn.bing.com/search?q=...`，HTTP 200 拿到结果页。
   已知会跳转的域名可直接对跳转目标发起第一次请求（如 bing 直接用 `cn.bing.com`）。

4. 超时或其它网络错误（非 redirect）：同一 URL **最多重试一次**；仍失败就放弃该 URL，改抓同一主题的官方文档站页面。典型：`raw.githubusercontent.com` 的 README 超时后，改抓 `docs.langchain4j.dev` 或 `docs.spring.io` 对应章节，不要对同一 raw URL 反复重试。

5. 确实需要「搜索」语义（不知道文档入口、需要找官方页面）时，直接构造并 `web_fetch` 结果页：`https://cn.bing.com/search?q=<URL 编码后的查询词>`（英文词与 `images/generations`、`compatible-mode` 这类路径片段一并编码，用 `+` 连接词）。从结果页里挑官网链接，再回到第 2 步抓正文。

6. 取证时只认官方文档：厂商域名（help.aliyun.com、docs.volcengine.com、docs.langchain4j.dev、docs.spring.io 等）或官方 GitHub README。博客、CSDN、聚合站只能当线索，不能当结论依据。

7. 判定「某厂商图像接口是否兼容 OpenAI 的 `images/generations`」时，必须**同时读两页**再下结论：
   - 「兼容模式」页：OpenAI 兼容到底覆盖哪些端点/模型。如阿里云 `https://help.aliyun.com/zh/model-studio/compatibility-of-openai-with-dashscope`。
   - 「原生 API」页：该图像模型的真实端点与请求字段。如阿里云 `https://help.aliyun.com/zh/model-studio/qwen-image-api`、火山方舟 `https://docs.volcengine.com/docs/ark/image-generation-api?lang=zh`。
   结论要显式区分：**文本/对话类模型有 OpenAI 兼容模式，图像生成走各自原生接口**（千问-文生图不在 compatible-mode 覆盖内，火山方舟走 ark 图像接口）；不要拿兼容模式页里的文本模型条目推断图像模型。

8. 写结论：每条事实后面跟官方文档 URL + 原文句子（引号内直引关键句），请求字段逐项列（如 `model`、`prompt`、`size`、`response_format`、鉴权头）并各自标来源 URL。抓不到的项留 `TODO: 文档未覆盖`，不要凭印象补。

## Pitfalls

- `no API key for "DEEPSEEK_API_KEY"` 后反复重试 `web_search`（换词、换条数）是纯浪费：必失败，且拖慢取证。一次都没有重试才是正确行为。
- `cross-origin redirect` 不是失败终点：一次「对报错里目标 URL 的原样重试」通常就成功。放弃改换搜索、或手工猜域名（把 `www.volcengine.com` 猜成别的路径）都会把一次成功变成多次失败。
- 同一 URL 连续重试超过一次是反模式（对 raw.githubusercontent 超时尤其明显）：换站而不是重试。
- 只读「兼容模式」页就断言图像接口兼容 —— 兼容页列的多是文本模型，`qwen-image` 之类根本不在 `images/generations` 覆盖内，会被误导。两页对照才成立。
- 结论里只给 URL 不给原文句子，等于没取证：用户明确要求「原文依据」时会被判不合格。
- 把 `www.bing.com/search` 当第一次请求（会先吃一次跨域 redirect），以及不 URL 编码中文/斜杠查询词，都会白掉一次调用。

## Verification

- 输出里每条结论都能点开对应官方文档 URL，并附至少一句原文引用；引用的句子能在该 URL 正文中找到。
- 回看调用序列：`web_search` 因缺 key 失败后没有第二次 `web_search`；每次 `cross-origin redirect` 后紧跟着一次目标 URL 的 `web_fetch`，没有对原 URL 的重复请求。
- 图像接口类结论同时引用了「兼容模式」页与「原生 API」页，并明确写出「文本模型兼容 / 图像模型走原生接口」的区分。
- 字段表每行都有来源 URL；未取到的字段以 `TODO` 标出而非臆造。

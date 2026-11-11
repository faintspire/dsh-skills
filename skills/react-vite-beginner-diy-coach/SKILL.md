---
name: react-vite-beginner-diy-coach
description: 以「学员自己动手」模式陪初学者用 Vite 搭 React 项目并排查 JSX 组件错误：先一条 pwsh 探测 Node/npm/执行策略给结论表，再给带「预期输出」的编号步骤，卡住时先递归列 src 再读真实文件诊断，只给位置与骨架、不代写代码。用户说「一步步教我怎么用 Vite 建 React 项目」「拆组件传 props 页面什么都没显示也不报错」「代码我自己敲你不要帮我写」「现在可以不」时使用。
whenToUse: 用户是 React 初学者、正在照教程用 npm create vite 建项目或拆组件传 props 时卡住，且明确要求自己动手（不要替他写文件）；也适用于他贴不出报错、需要你直接看文件定位白屏/整页红字的原因时。
---

# React + Vite 初学者「自己动手」教练

## 硬边界（先立住，再动手）

- **一行代码都不写**：不创建项目、不建/改文件、不粘贴可整文件运行的源码、不代替执行 `npm create` / `npm install` / `npm run dev`。
- 允许由你执行的只有**只读探测**：`node -v`、`npm -v`、`Get-ChildItem`、`Get-ExecutionPolicy`、`read`、以及用 `node -e` 现场复现错误原文。
- 你产出的是**位置 + 结构骨架**：组件名、变量名、文字内容全部留空给用户自己定。
- 用户若改口「你直接帮我写完」，先复述这条边界并确认，再按新指令执行。

## Steps

1. **先确认练手目录，再探测环境。** 用一条 pwsh 命令同时拿齐版本、包管理器、目录和策略，不要分多次问：

   ```powershell
   node -v; npm -v; "--- pnpm ---"; (Get-Command pnpm -ErrorAction SilentlyContinue).Source; "--- 目录 ---"; Get-ChildItem -Force . | Select-Object Mode,Length,Name | Format-Table -AutoSize; "--- 执行策略 ---"; Get-ExecutionPolicy -Scope CurrentUser
   ```

2. **把探测结果压成结论表**，只写「项 / 实测 / 结论」，不铺陈：

   | 项 | 实测 | 结论 |
   |---|---|---|
   | Node | `v24.15.0` | 够用（Vite 7 要求 20.19+） |
   | npm | `11.12.1` | 够用 |
   | pnpm | `...\npm\pnpm.ps1` | 可选，有就可用 |
   | PowerShell 执行策略 | `RemoteSigned` | `npm` 能直接跑，不用改 |
   | 现有目录 | 已有 `xxx.html` / `.vscode` | 保持不动，新项目放子目录 |

   Node < 20.19 或命令找不到 → 结论写「不够用」并给升级动作，仍由用户自己执行。

3. **输出编号步骤链，每步三件套：命令 / 为什么 / 预期输出。** 链条按「定目录与项目名 → 脚手架 → 安装依赖 → 启动 → 改标题 → 拆组件传 props」推进；脚手架命令形如：

   ```powershell
   npm create vite@latest <项目名> --
   ```
   之后**照抄终端打印的 Next steps**（通常是 install → run dev），不要凭记忆替它补命令。
   每步都写「**预期输出**：…」，并交代「卡住就把报错原文整段贴我，不要只说『不行』」。

4. **项目名规则要连原因一起说**：全小写、无空格、无中文、用 `-` 连接。因为 npm 包名不允许大写和中文，`React练习` 这类名字会直接报 `Invalid name`。

5. **用户说「我卡在第 N 步 / 7、8 不太明白」时，先取证再诊断**（禁止套模板）：

   ```powershell
   Get-ChildItem <项目>\src -Force -Recurse | Where-Object { $_.FullName -notmatch 'node_modules' } | Select-Object FullName
   ```
   然后逐个 `read` 真实文件（如 `file_path=<项目>\src\App.jsx`、`file_path=<项目>\src\components\<组件>.jsx`，长文件可加 `limit=30`）。诊断必须落在**用户真实那一行**上。

6. **拆组件传 props 的四类经典错，固定按这个顺序逐条给「文件:行号 + 后果」**，不要跳序、不要合并成一句：
   1. **组件名必须大写开头** —— 小写会被当成 HTML 标签，**静默不渲染也不报错**（用户「写了没反应」的第一原因）。
   2. **文件末尾必须 `export default <组件名>;`** —— 没有导出，外面 `import` 拿到空值，渲染中断。
   3. **使用处必须 `import`** —— 没 import 就用标签、或 import 行还是注释状态，等价于第 1 类。
   4. **不许 `props = {…}` 覆盖传入值** —— 函数体第一行赋值会把父组件传的 props 冲掉，页面永远只显示写死的内容，且不报错。改法是删掉该行，直接读 `props.xxx` 或解构参数。

7. **`export default` 后面是空的 → 判成语法错误，并现场复现错误原文当证据：**

   ```powershell
   node --input-type=module -e "export default" 2>&1 | Select-Object -First 6
   ```
   预期最后一行 `SyntaxError: Unexpected end of input`。pwsh 会把 stderr 包成 `NativeCommandError` 前缀，`2>&1` 必加，取头部几行即可。
   结论要写清：这是**语法解析失败导致整页渲染中断**，不是 props 传值的问题；`App.jsx` 那行 `import` 实际拿不到组件。

8. **用户问「现在可以不 / 好了吗」时，一律重读文件核对后再答**，绝不凭上一轮记忆断言。重读 `Greeting.jsx` 与 `App.jsx`，逐条核对第 6 步的四项。

9. **回复固定三段式**，顺序不许换：
   1. **你已经做对的部分** —— 逐条列真实做对的（文件位置、组件名大写、import 路径、传 prop 写法）。
   2. **问题表** —— 每条写「`文件:行号` 发生了什么 / 错误原文或静默后果 / 最小改法」，并用「问题 1 / 问题 2」编号，能叠加就说明是叠加导致。
   3. **操作清单** —— 按「文件 → 操作 1/2/3」给编号动作，每步只写该在第几行、写什么结构，代码留骨架：
      ```
      export default 你的组件名;
      import 你的组件名 from "./components/你的文件名";
      <你的组件名 属性名="值" />
      ```
      并明确写「组件名、变量名、显示文字由你自己定」。

## Pitfalls

- **小写组件名静默不渲染**：初学者最容易被「没报错」误导，把原因归到 props；先查大小写。
- **把空 `export default` 说成 props 问题**：白屏 + 整页红字的真因是 `SyntaxError: Unexpected end of input`，先修语法再谈传值。
- **`props = {…}` 覆盖**：写死了值页面看着「能显示」，其实父组件的传值全被丢弃，不报错、极难自查。
- **凭记忆回答「现在可以不」**：用户两轮之间大概率只改了一半，必须重读文件。
- **套模板给整份源码**：直接贴完整 `App.jsx` 会剥夺练习目的，也违反本窗口硬边界；只给位置与骨架。
- **目录列表忘了排除 `node_modules`**：递归列 `src` 时用 `Where-Object { $_.FullName -notmatch 'node_modules' }`。
- **pwsh 下直接跑 `node -e` 取错误**：不加 `2>&1` 拿不到 stderr 原文，只剩一个 exit code。
- **替用户执行脚手架/启动**：`npm create`、`npm install`、`npm run dev` 都由用户敲，你只给命令和预期输出。

## Verification

回答前自检（缺一条就补齐再发）：

- [ ] 环境结论表四项齐全：Node / npm / 包管理器 / 执行策略 + 现有目录结论（够用 / 不够用）。
- [ ] 每个步骤都带「预期输出」，并已要求用户卡住时贴报错原文。
- [ ] 所有判断都基于本轮 `Get-ChildItem` + `read` 到的真实行，没有一条来自模板猜测。
- [ ] 每条问题都有 `文件:行号`，且顺序为 大小写 → export default → import → props 覆盖。
- [ ] 语法类问题附了现场复现的错误原文（`2>&1` 拿到），不是凭印象写的错误信息。
- [ ] 操作清单里没有任何整文件可粘贴源码，只出现「第 N 行 + 骨架 + 由你自己命名」。
- [ ] 用户问「现在可以不」的那一轮，回复前确实重读过 `App.jsx` 与组件文件。

# dsh-gateway-provider

[![gitleaks](https://github.com/Luck9Star/dsh-gateway-provider/actions/workflows/gitleaks.yml/badge.svg)](https://github.com/Luck9Star/dsh-gateway-provider/actions/workflows/gitleaks.yml)

> English: [README.md](../README.md)

把 **LLM 网关背后的全部模型** —— newapi、LiteLLM、Higress，或任何 OpenAI 兼容端点 —— 直接搬进 DeepSeek Harness 用。

装好插件、贴上 API key，网关里的每个模型就会出现在 dsh 的模型选择器里，并且带着从 [models.dev](https://models.dev) 拉来的**真实**参数（上下文窗口、输出上限、推理支持）。请求按每个模型各自的原生协议出去 —— OpenAI、Anthropic 或 Gemini —— 工具调用和流式输出都按该模型官方的方式工作。

## 为什么需要它

dsh 自带的适配器一个 provider 一个。如果你的模型都在网关后面，手工做法是维护一份静态模型清单，上下文窗口和输出上限全靠猜。本插件直接把网关本身挂进来：

- **不用手工维护** —— 模型列表从网关读取（`GET /v1/models`，newapi 附加管理 API 兜底）；网关侧加一个模型，dsh 里就多一个，无需重新部署。
- **真实数字，不是猜测** —— models.dev 数据补齐上下文窗口、输出上限、推理档位、发布日期；配置默认值只做兜底。
- **一个插件，所有协议** —— 每个模型走自己的协议（OpenAI chat completions / OpenAI responses / Anthropic messages / Gemini），由官方 dsh 适配器同款的 [pi-ai](https://www.npmjs.com/package/@earendil-works/pi-ai) SDK 处理。
- **多网关并存** —— 默认 `newapi` 路由，外加每个额外网关一条 `gateway:<id>` 路由，各自的缓存与密钥。
- **设置页面代替 YAML** —— **Settings → Gateway Models**：从模板添加网关（NewAPI / LiteLLM / Higress / OpenAI 兼容 / 完全自定义）、测连接、同步模型、隐藏或覆盖任意模型、手工添加自定义模型。

## 环境要求

- DeepSeek Harness (dsh)，带 `web` profile（设置页面是 web UI 扩展；provider 本身任何 profile 都能用）。
- 一个网关 API key（比如 newapi 令牌）。

## 从 GitHub 源码安装（写给执行者/AI 的步骤）

装本仓库检出本身，不装 npm 发布版：npm 版本可能滞后（发布构建早于未发布的修复）。以下按"谁执行谁读"写——路径、命令、检查点齐全。
前置：pnpm（≥ 10）；Windows 上还要 Git Bash（`bash` 在 `PATH` 里——仓库 `prepare` 钩子要跑 `scripts/link.sh`）。

1. **克隆并装好仓库依赖** —— 仓库必须自带锁定的 pi-ai；harness 同名包软链到宿主 profile，保证进程内单实例：

   ```sh
   git clone <仓库地址> dsh-gateway-provider
   cd dsh-gateway-provider
   DSH_HOME=<harness-home> pnpm install
   ```

   `<harness-home>`：命令行版 dsh 为 `~/.dsh`；DSH Desktop 为 `%APPDATA%\dsh-desktop\harness`。`DSH_HOME` 指向它后，`prepare` 钩子会把 `node_modules/@deepseek-ai/*` 软链进该 harness 的共享树（单实例）。
   装完检查：`node_modules/@earendil-works/pi-ai/package.json` 存在；`node_modules/@deepseek-ai/cordis` 是链接（Windows 上是目录 junction）。没有软链权限时，把失败的每条链接改成 junction：`cmd /c mklink /J <链接> <目标>`。

2. **接线目标 profile**（先备份它的 `package.json`）：

   - profile 目录：CLI 为 `$DSH_HOME/profiles/<name>`（通常是 `web`）；Desktop 为 `%APPDATA%\dsh-desktop\harness\profiles\web`；
   - 在 profile 的 `package.json` `dependencies` 加：`"dsh-gateway-provider": "link:<仓库绝对路径>"`；
   - 注册插件行，只加一处——往 profile 的 `cordis.patch.yml` 追加：

     ```yaml
     - insert:
         - id: llm-newapi
           name: 'dsh-gateway-provider'
     ```

     （等价做法：把包名加进 profile 的 `dsh.profile.bundles`，行由插件自带 bundle patch 提供。两种只能二选一：`llm-newapi` 重复注册 = 加载器报错。）
   - 在 profile 目录里跑 pnpm：`pnpm install`。DSH Desktop 必须走它自己的 runner（store 固定 + EPERM 恢复）：
     `node "<desktop>\resources\app\node_modules\node\bin\node.exe" "<harness>\.desktop-bin\pnpm-runner.mjs" "<desktop>\resources\app\node_modules\pnpm\bin\pnpm.cjs" install`

3. **重启 harness** —— Desktop：完全退出再打开应用（harness 是壳的子进程）；CLI：重启 `dsh`。

4. **验证** —— `Settings → Gateway Models` 出现管理 UI；模型选择器多出 "NewAPI" 路由。失败时看 harness 日志（Desktop：`%APPDATA%\dsh-desktop\logs\harness.log`；插件行 `llm-newapi`，包名 `dsh-gateway-provider`）。

5. **提供 API key** —— 把凭据写入 harness 凭据文件（默认变量名 `NEWAPI_API_KEY`；CLI `~/.dsh/.credentials.yaml`；Desktop `%APPDATA%\dsh-desktop\harness\.credentials.yaml`），或让设置页代写。网关卡片点 **Test**，预期 `✓ Connected — N models`；不用 newapi 公有云就先在卡片上填 **Base URL**。

迭代 = 改克隆目录 → 重启 harness 即生效（`lib/client.js` 就是实际 UI 产物，无构建步骤）。卸载：移除补丁行和依赖，在 profile 里重跑 `pnpm install`。

## 日常使用

一切都在 **Settings → Gateway Models**：

- **加更多网关** —— "Add Gateway"，选模板（LiteLLM、Higress、OpenAI 兼容，或按协议分别填 URL 的完全自定义），填 base URL 和 key 的环境变量名，Test、Sync。每个网关在选择器里有自己的路由。
- **管好模型列表** —— 非对话模型（图像 / 语音 / 向量 / 重排……）默认被正则排除；任意模型可隐藏、可改名；网关藏起来的模型可手工添加；每个模型的协议、上下文窗口、输出上限、推理档位、**是否支持图片输入**都可改。目录里没有的视觉模型（或想强制关掉某模型的看图能力），把"图片输入"从"自动"改成"支持图片 / 仅文本"即可。
- **密钥放在 dsh 凭据存储里** —— 设置页有状态徽标（`✓ Key set · NEWAPI_API_KEY` / `⚠ No key set`），也可以直接帮你把 key 写进去。

## 配置参考

全部可选 —— 下表每项都有可用默认值。配置写在 `~/.dsh/settings.yaml` 的 `llm-newapi:` 段（设置页改的就是同一批键）。常用项：

| 键 | 默认值 | 含义 |
| --- | --- | --- |
| `baseURL` | `https://api.newapi.ai` | 网关 base URL。环境变量兜底：`NEWAPI_BASE_URL`、`NEWAPI_API_URL`。 |
| `apiKeyEnv` | `NEWAPI_API_KEY` | key 存在哪个环境/凭据变量里。 |
| `label` | `NewAPI` | 选择器里显示的路由名。 |
| `flavor` | `newapi` | 仅作模板标签（`newapi` / `litellm` / `higress` / `openai-compatible` / `custom`）。 |
| `gateways` | — | 额外网关数组：`{ id, baseURL, apiKeyEnv, label, … }`，每个成为一条 `gateway:<id>` 路由。 |
| `models` | — | 按模型覆盖：`{ id, name, disabled, protocol, contextWindow, maxTokens, reasoningLevels, inputModalities }`。 |
| `useModelsDev` / `modelsUrl` | `true` / models.dev | 参数增补来源（支持 `file:` URL 离线用）。 |
| `excludePatterns` | 图像/语音/…… | 要从选择器排除的模型 id 正则列表。 |
| `sortModelsByRelease` | `true` | 最新模型排前面。 |
| `catalogMode` | `auto` | `v1`（只用 `/v1/models`）/ `management`（newapi 用户 API）/ `auto`。 |
| `endpointPriority` | responses → anthropic → openai → gemini | 模型支持多种协议时的优先序。 |
| `openaiURL` / `responsesURL` / `anthropicURL` | — | 仅完全自定义网关：按协议分别填端点 URL；不填 = 该协议关闭。 |
| `maxTokens` / `defaultContextWindow` | `32768` / `128000` | models.dev 无数据时的兜底。 |
| `streamIdleTimeoutMs` | `600000` | 流式空闲超时。 |
| `headers` | — | 发往网关的额外 HTTP 头。 |

## 常见问题

| 症状 | 原因 → 处理 |
| --- | --- |
| 路由有了但模型为零 | 插件读不到模型列表。检查网关 base URL；newapi 网关若限制 `/v1/models`，试试 `catalogMode: "management"`。 |
| 每次请求都 `401` / 鉴权错误 | key 缺失或不对：看 Settings → Gateway Models 的徽标，或 `~/.dsh/.credentials.yaml` 里的 `NEWAPI_API_KEY`。 |
| 某模型的上下文窗口看着不对 | models.dev 没匹配上。在设置页编辑该模型（或写 `models:` 覆盖）。 |
| 单个模型工具调用不稳、格式怪 | 该模型被路由到了它处理不好的协议。在模型上钉死 `protocol`（`openai`、`openai-response`、`anthropic`、`gemini`）。 |
| 自定义网关各协议端点分开 | 用 `flavor: "custom"`，显式填 `openaiURL` / `responsesURL` / `anthropicURL`。 |

## 工作原理（一分钟版）

启动时，插件为每个网关注册一条 provider 路由，从网关拉取模型列表，再把每个模型 id 与 models.dev 模糊匹配补齐真实参数。选中模型后，dsh 的请求被翻译成 pi-ai SDK 的格式、按该模型的原生协议发出；流式回复再翻译回 dsh 的分块。模型目录按网关缓存（默认 30 分钟）。没有任何手写协议代码 —— 桥接层直接复用官方 `dsh-llm-pi-ai` 适配器。

## 开发

```sh
git clone https://github.com/Luck9Star/dsh-gateway-provider
cd dsh-gateway-provider
pnpm install               # 一条命令搞定:pi-ai 固定副本 + @deepseek-ai/* 软链(prepare 钩子自动执行)
pnpm run test:client       # 设置页渲染测试，双语言
pnpm run test:urls         # URL 派生单元测试
pnpm run smoke             # 真实网关往返（需要真 key）
```

### 为什么插件自带 pi-ai

`@earendil-works/pi-ai` 是精确锁定的直接依赖，与 harness 自带的 pi-ai 版本解耦：网关模型目录（思考档位、各家 compat，如智谱 GLM 的 `supportsDeveloperRole: false`）不再随 harness 升级被动漂移——harness 旧目录里缺的模型不会再退化请求编码。插件与 harness 的边界只传纯数据（`GenerateOptions` 进、dsh `StreamChunk` 出，`lib/pi-bridge.js` 从不把 pi-ai 对象泄漏到边界外），因此插件副本与 harness 自带副本可在同一进程内安全共存。

从本地检出开发 = 上面"源码安装"那套流程的迭代循环：改检出 → 重启 harness 即生效。接线只保留一条路径（要么 profile 补丁行、要么 `dsh.profile.bundles` 成员——两条都做会出现重复 `llm-newapi` 行 = 加载器报错）。离线测试：`pnpm run test:client`、`test:urls`、`test:schema`、`test:errors`；`smoke` 需要真实网关 key。

## 参考与致谢

- [pi-ai SDK](https://www.npmjs.com/package/@earendil-works/pi-ai) —— 四种协议全由它实现；桥接层复用官方 `dsh-llm-pi-ai` 适配器的翻译代码。
- [models.dev](https://models.dev) —— 参数目录（上下文窗口、输出上限、推理、发布日期）。
- [new-api](https://github.com/QuantumNous/new-api)、
  [LiteLLM](https://github.com/BerriAI/litellm)、
  [Higress](https://github.com/alibaba/higress) —— 本插件测试过的网关（任何 OpenAI 兼容端点都行）。

## 安全

密钥只存放在 dsh 凭据存储或启动环境里 —— 绝不写进 settings YAML。仓库在 CI 与 pre-commit 中运行 [gitleaks](https://github.com/gitleaks/gitleaks) 防泄漏。

## 许可证

[MIT](../LICENSE)

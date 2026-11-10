# 供在此代码仓库中工作的编码智能体使用

更改任何内容前请先阅读本文。以下每一行均已于 2026-11-09 根据实时 API、`brightdata-sdk` 2.5.2 或 Bright Data CLI 0.4.1 验证。如果 API 参考页面与实时 API 不一致，以此处记录的实时 API 行为为准。

## 身份验证

- `token()` 先读取 `BRIGHTDATA_API_TOKEN`，然后读取存储在 `brightdata-cli/credentials.json` 中的密钥 `bdata login`（Python SDK 也读取同一个文件）。设置该变量或运行一次登录，是唯一必须由人工完成的步骤；切勿将密钥粘贴到代码中。
- README 中的原始 HTTP 快速入门仅读取 `BRIGHTDATA_API_TOKEN`。它无法获取 CLI 登录信息。

## API 的行为方式

- 仅使用一个数据集 `gd_m7aof0k82r803d5bjm`，且仅支持 collect。每个输入都需要 `url: "https://chatgpt.com/"` 和一个 `prompt`。
- Python SDK 对其进行了封装，但不会发送 `require_sources`。此软件包使用标准库调用 `https://api.brightdata.com/datasets/v3`，且没有任何依赖项。请保持这种方式。
- 每次调用都是异步任务：`/trigger`、轮询 `/progress/<id>`，然后执行 `/snapshot/<id>?format=json`。获取回答需要 44 至 282 秒；一次使用 `web_search` 和 `require_sources` 的调用耗时 667 秒。`TIMEOUT` 为 900。
- 如果状态轮询因网络错误而失败，会持续重试直至截止时间。HTTP 错误仍会引发异常。
- 始终发送 `include_errors=true`。
- 2026-11-09 的实时行为：
  - 如果没有 `web_search`，`citations` 可能是 `null`，而不是 `[]`（一次运行），也可能是完整列表（另一次运行）。请针对 `null` 做防护。
  - `web_search_triggered` 不会与输入保持一致：在包含十条引用的回答中为 `false`，而在未使用 `web_search` 发送的提示词中为 `true`。
  - 每条记录中的 `model` 都是 `null`。
  - `geolocation`（"40.7484,-73.9857"，曼哈顿中城）得到的是关于休斯敦的回答，并显示“由于无法获取你的精确位置”。不要在文档中声称它可以正常工作。
  - `answer_text_markdown` 采用 html2text 风格：使用 Setext 标题、竖线表格，代码放在单独语言名称行下方并缩进四个空格，绝不使用围栏代码块。
  - `answer_html` 是完整页面，一行约 1.4 MB。因此，README 摘录会将每行截断为 160 个字符；测试也采用相同方式。
- 架构可能随时更改。绝不要硬编码字段列表。
- 完整文档索引，每个条目对应一个 `.md` 页面：https://docs.brightdata.com/llms.txt

## 在此代码库中工作

- `pytest` 可离线运行且不需要令牌。`ruff check .` 必须通过。
- CI 会在空白机器上执行 README 自身的命令，并通过精确的 H2 文本查找各个章节。如果重命名标题，也要在 ci.yml 中更新。
- 每周工作流会针对真实 API 执行 README 中的每个围栏代码块，每日工作流则会执行较小规模的实时检查。代码片段必须完整且可直接单独粘贴运行，展示的输出也必须真实。
- README 中的字段表位于 `<!-- fields:start -->` 与 `<!-- fields:end -->` 之间，每天由 `.github/scripts/field_tables.py` 根据 https://docs.brightdata.com/scrapers-full.json 重新生成。不要手动编辑。
- README 顶部的“上次验证”徽章行由每日运行自动重写。不要手动编辑。
- 标题按搜索需求编写，例如“Python 版 ChatGPT 爬虫工具”“如何抓取……”。添加章节时请保持这种模式。
- 保持精简。除状态轮询外，不要添加重试、去重、调度、数据库、异步示例或并发。

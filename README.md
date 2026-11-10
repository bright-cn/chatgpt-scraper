# Python 版 ChatGPT 爬虫工具

使用 Python 将 ChatGPT 回答、引用和来源抓取为 JSON。跟踪品牌在 ChatGPT 中的可见度。无需 OpenAI API 密钥。基于 [Bright Data ChatGPT 爬虫工具 API](https://www.bright.cn/products/web-scraper/chatgpt?utm_source=github&utm_medium=referral&utm_content=chatgpt-scraper) 构建。新账户每月可获得 5,000 免费额度。

[![实时检查](https://github.com/bright-cn/chatgpt-scraper/actions/workflows/live.yml/badge.svg)](https://github.com/bright-cn/chatgpt-scraper/actions/workflows/live.yml) [![上次验证](https://img.shields.io/badge/last%20verified-9%20Nov%202026-brightgreen)](https://github.com/bright-cn/chatgpt-scraper/actions/workflows/live.yml) <!-- verified: rewritten by the daily run -->

[快速入门](#如何使用-python-抓取-chatgpt-结果) · [无需 OpenAI 密钥](#无需-openai-api-密钥即可抓取-chatgpt) · [CLI](#批量-chatgpt-提示词抓取-cli) · [引用](#跟踪某个提示词的-chatgpt-引用和品牌可见度) · [端点](#抓取-chatgpt-回答引用和来源) · [数据](#chatgpt-回答数据字段json) · [错误](#chatgpt-爬虫工具错误) · [MCP 与 Claude Code](#chatgpt-mcp-服务器和-claude-code-技能) · [免费额度](#这个-chatgpt-爬虫工具免费吗) · [文档](https://docs.brightdata.com/products/scrapers/chatgpt/introduction) · [支持](#支持)

在 Python 中以 JSON 获取 ChatGPT 回答：发送提示词，即可获取 ChatGPT 在其自身界面中显示的回答，包括文本、Markdown 和 HTML 格式，以及它引用的网站；还可按国家/地区获取结果、使用网页搜索和后续提示词。专为 ChatGPT 可见度跟踪和 GEO 打造：了解 ChatGPT 提及和推荐哪些品牌，以及它引用哪些来源。无需 OpenAI API 密钥、浏览器或依赖项。基于 [Bright Data ChatGPT 爬虫工具 API](https://www.bright.cn/products/web-scraper/chatgpt?utm_source=github&utm_medium=referral&utm_content=chatgpt-scraper) 构建。

这里还提供：用于处理提示词列表的单命令 CLI，以及完全不需要 Python 的 [Bright Data CLI](#chatgpt-mcp-服务器和-claude-code-技能)。

## 如何使用 Python 抓取 ChatGPT 结果

需要 Python 3.10 或更高版本，无需安装任何内容：仅使用标准库。

```bash
export BRIGHTDATA_API_TOKEN=YOUR_API_KEY
```

从 [Bright Data 控制面板](https://www.bright.cn/cp/setting/users)获取令牌。还没有账户？[创建账户](https://www.bright.cn/cp/start)；新账户每月可获得 [5,000 免费额度](https://docs.brightdata.com/general/account/billing-and-pricing/free-tier)。

```python
import json
import os
import time
import urllib.request

API = "https://api.brightdata.com/datasets/v3"
HEADERS = {
    "Authorization": f"Bearer {os.environ['BRIGHTDATA_API_TOKEN']}",
    "Content-Type": "application/json",
}


def call(path, body=None):
    data = None if body is None else json.dumps(body).encode()
    request = urllib.request.Request(API + path, data=data, headers=HEADERS)
    with urllib.request.urlopen(request) as response:
        return json.load(response)


prompt = {
    "url": "https://chatgpt.com/",
    "prompt": "Best CRM software for small businesses",
    "country": "US",
    "web_search": True,
}
job = call("/trigger?dataset_id=gd_m7aof0k82r803d5bjm&include_errors=true", [prompt])
while call(f"/progress/{job['snapshot_id']}")["status"] not in ("ready", "failed"):
    time.sleep(10)
answer = call(f"/snapshot/{job['snapshot_id']}?format=json")[0]
if "error" in answer:
    raise SystemExit(answer["error"])
print(answer["answer_text"][:120])
for citation in (answer["citations"] or [])[:3]:
    print("cited:", citation["url"])
```

```
The best CRM software for a small business depends on whether you primarily need to manage customers, track sales, gener
cited: https://www.hubspot.com/products/crm?utm_source=chatgpt.com
cited: https://www.zoho.com/crm/?utm_source=chatgpt.com
cited: https://www.pipedrive.com/?utm_source=chatgpt.com
```

预计需要一到五分钟：API 会向 ChatGPT 提问，循环则等待回答。一次启用网页搜索和来源的运行耗时十一分钟。按[每个回答](https://www.bright.cn/pricing/web-scraper)计费。

该代码片段中有三项不是可选项。

`url` 必须为 `https://chatgpt.com/`。爬虫工具要求每个输入都包含它；提示词放在 `prompt` 中。

`web_search: true` 会要求 ChatGPT 搜索网页，这通常会带来引用。如果不使用它，`citations` 可能返回 `null`，而不是空列表，因此循环会对此进行防护。

`include_errors=true` 会让 API 将失败的提示词报告为包含 `error` 键的行。API 默认关闭此功能。

Bright Data [Python SDK](https://github.com/bright-cn/bright-data-sdk-python) 封装了这个抓取工具，但它不会发送 `require_sources`，因此无法要求 ChatGPT 提供引用。所以这里以及此软件包中都使用普通 HTTP。

## 无需 OpenAI API 密钥即可抓取 ChatGPT

ChatGPT 爬虫工具 API 不需要 OpenAI 账户、OpenAI API 密钥或 ChatGPT 登录信息。唯一凭据是设置为 `BRIGHTDATA_API_TOKEN` 的 Bright Data API 密钥。此软件包及其命令也会查找由 `bdata login` 存储的密钥。

它返回的是用户在 ChatGPT 本身看到的内容，而不是 OpenAI API 返回的内容：包括回答及其引用、链接、购物和地图区块，并按照 ChatGPT 为相应国家/地区呈现的方式返回。

## 批量 ChatGPT 提示词抓取 CLI

此代码库中的命令会同时对多个提示词运行快速入门流程，并写入一个 JSON 文件。

```bash
pip install git+https://github.com/bright-cn/chatgpt-scraper
chatgpt-scraper "Best CRM software for small businesses" "Best project management tool for startups" --web-search
```

```
Asking ChatGPT 2 prompt(s) from US, with web search
Usually one to five minutes each. Billed per answer, 5,000 free credits a month.
asking  'Best CRM software for small businesses'...
got     'Best CRM software for small businesses': 3,416 characters, 11 citations
asking  'Best project management tool for startups'...
got     'Best project management tool for startups': 2,903 characters, 0 citations

Saved 2 answers as JSON to chatgpt.json
```

在终端中，`asking` 行会被原地更新的旋转指示器和运行计时器替代，因此你可以看到它正在运行以及已经运行了多长时间。

```
--country CC   two-letter country ChatGPT answers from, default US
--web-search   ask ChatGPT to search the web before answering
--out PATH     output file, default chatgpt.json
```

`python -m chatgpt_scraper` 也可以使用。该命令会从 `BRIGHTDATA_API_TOKEN` 中查找令牌；如果你运行过一次 `npx -p @brightdata/cli bdata login`，它也会从已存储的登录信息中查找令牌，供你以及该终端中的任何编码智能体使用。

你也可以导入它而不是直接运行，从而为每个提示词获取 `ok` 和 `error`，而不是原始行。`scrape` 不会因某一个错误提示词而引发异常；读取 `answer` 前请检查 `ok`：

```python
from chatgpt_scraper import scrape

for outcome in scrape(["Best running shoes for beginners", "Best free photo editing app"]):
    if outcome.ok:
        print(f"{outcome.prompt}: {outcome.answer['answer_text'][:60]}")
    else:
        print(f"{outcome.prompt} failed: {outcome.error}")
```

```
Best running shoes for beginners: If you're just getting into running, I'd recommend shoes tha
Best free photo editing app: Best overall free app Snapseed Snapseed Free on iPhone and A
```

## 抓取 ChatGPT 回答、引用和来源

这里只有一个 ChatGPT 爬虫工具，其文档位于 [ChatGPT 爬虫工具 API 文档](https://docs.brightdata.com/products/scrapers/chatgpt/introduction)。下面的每个代码片段都是完整的：执行上面的 `pip install` 后可原样粘贴运行。它们每周一都会在 Actions 中运行，每隔一天还会执行一次较小规模的检查，顶部徽章显示的是最新结果。

| 你想要的内容 | 方法 |
| --- | --- |
| ChatGPT 对提示词的回答 | `ask(["prompt one", "prompt two"], "US")` |
| ChatGPT 引用的网站 | `ask(prompt, web_search=True, require_sources=True)` |
| 同一对话中的后续提问 | `ask(prompt, additional_prompt="...")` |
| 来自其他国家/地区的回答 | `ask(prompt, "DE")` |
| Markdown 格式的回答 | 任意回答上的 `answer_text_markdown` |

`ask` 接受一个提示词或提示词列表。列表是一个任务，而不是每个提示词一个任务。每个回答对应一条记录，API 无法回答的提示词会作为包含 `error` 键的行返回。API 接受的任何其他输入都按 API 中的命名方式传入，例如 `index`。

### 跟踪某个提示词的 ChatGPT 引用和品牌可见度

```python
from urllib.parse import urlparse

from chatgpt_scraper import ask

answer = ask("Best project management software for remote teams", web_search=True, require_sources=True)[0]
domains = [urlparse(c["url"]).netloc for c in answer["citations"]]
print(len(domains), "citations from", len(set(domains)), "sites:")
for domain in sorted(set(domains))[:5]:
    print(" ", domain, domains.count(domain))
```

```
10 citations from 9 sites:
  asana.com 2
  basecamp.com 1
  clickup.com 1
  monday.com 1
  resources.rework.com 1
```

`require_sources` 要求 ChatGPT 提供引用，`web_search` 则让它进行查找。`citations` 保存每个被引用的 `url` 及其 `title`；`links_attached` 保存回答文本中的链接。每天询问相同提示词，是 GEO 工具随时间跟踪引用份额的方式。

### 向 ChatGPT 提出后续问题

```python
from chatgpt_scraper import ask

answer = ask(
    "Name three popular sourdough bread recipes",
    additional_prompt="Which of them is easiest for a beginner? One sentence.",
)[0]
print("first: ", answer["answer_text"][:80])
print("second:", answer["additional_answer_text"][:80])
```

```
first:  Here are three popular sourdough bread recipes: 1. Classic Artisan Sourdough Bre
second: Sourdough sandwich bread is the easiest for beginners because it's soft, forgivi
```

后续问题在同一对话中运行，因此可以引用第一个回答。其回答会存入 `additional_answer_text`，并带有自己的 `citations_additional_prompt`。

### 按国家/地区比较 ChatGPT 回答

```python
from chatgpt_scraper import ask

for country in ("US", "DE"):
    answer = ask("Which supermarket chain has the cheapest groceries?", country, web_search=True)[0]
    print(country, "|", answer["answer_text"][:90])
```

```
US | ALDI is generally the cheapest traditional supermarket chain in the U.S. , according to 20
DE | In Germany, Aldi, Lidl, Netto and Penny are generally the cheapest supermarket chains. The
```

`country` 是 ChatGPT 回答所依据国家/地区的双字母代码。美国回答提到了 ALDI；德国回答则提到了 Aldi、Lidl、Netto 和 Penny。也接受 `geolocation`，但在 2026 年 11 月 9 日，曼哈顿中城坐标得到的是关于休斯敦的回答，并以“由于无法获取你的精确位置”开头，因此此软件包不将其记录为可用功能。

### 以 Markdown 获取 ChatGPT 回答

```python
from pathlib import Path

from chatgpt_scraper import ask

answer = ask("Explain what a REST API is, with one short code example")[0]
markdown = answer["answer_text_markdown"]
Path("answer.md").write_text(markdown, encoding="utf-8")
lines = markdown.splitlines()
headings = [line for line, under in zip(lines, lines[1:]) if under and set(under) == {"-"}]
print(len(markdown), "characters saved to answer.md; headings:", headings)
```

```
1854 characters saved to answer.md; headings: ['How a REST API works', 'Short code example', 'In simple terms']
```

`answer_text_markdown` 由渲染后的页面转换而来：标题使用短横线下划线，而不是写成 `#`；表格采用竖线表格；代码位于标明语言的行下方并缩进四个空格，而不是使用围栏。`answer_text` 是同一回答的纯文本版本，`answer_html` 则是完整页面，约 1.4 MB。

### 立即触发，稍后获取

对于较长的提示词列表，不要让进程阻塞一小时。触发任务并保存快照 ID，准备就绪后再获取。快照可供下载 30 天。

```python
import time

from chatgpt_scraper import CHATGPT, fetch, status, trigger

item = {"url": "https://chatgpt.com/", "prompt": "What is the capital of Australia?", "country": "US"}
snapshot = trigger(CHATGPT, [item])
print("snapshot:", snapshot)
while (state := status(snapshot)) not in ("ready", "failed"):
    time.sleep(10)
print("status:", state)
print("fetched:", fetch(snapshot)[0]["answer_text"][:60])
```

```
snapshot: sd_mw9bv3ophde91g5k7
status: ready
fetched: The capital of Australia is Canberra . 🇦🇺 It is located in t
```

实时 API 在 2026-11-09 接受的输入字段，适用于数据集 `gd_m7aof0k82r803d5bjm`：

| 输入 | 必填 | 作用 |
| --- | --- | --- |
| `url` | 是 | 始终为 `https://chatgpt.com/` |
| `prompt` | 是 | 问题 |
| `country` | 否 | ChatGPT 回答所依据国家/地区的双字母代码 |
| `web_search` | 否 | `true` 要求 ChatGPT 搜索网页 |
| `require_sources` | 否 | `true` 要求 ChatGPT 引用来源 |
| `additional_prompt` | 否 | 同一对话中的后续提问 |
| `index` | 否 | 你自己的编号，会随记录一起返回 |
| `geolocation` | 否 | `"lat,long"`；在我们的测试中未生效 |

`bdata pipelines info gd_m7aof0k82r803d5bjm collect_by_url` 会打印 API 验证的输入字段。

## ChatGPT 回答数据字段（JSON）

大多数人希望从回答中获取的字段：

```
prompt  answer_text  answer_text_markdown  citations  links_attached  country  web_search_triggered
```

`web_search_triggered` 不会遵循 `web_search` 输入：在 2026 年 11 月 9 日，包含十条引用的回答中其值为 `false`，而在未使用 `web_search` 发送的提示词中其值为 `true`。每条记录中的 `model` 都是 `null`。代码未硬编码任何字段列表。API 返回的所有内容都会进入相应记录以及命令生成的文件。

<!-- fields:start -->
每天根据已发布的架构 [scrapers-full.json](https://docs.brightdata.com/scrapers-full.json) 重新生成，因此不会过时。每条记录包含适用于它的字段，以及 `timestamp` 和 `input`；示例文件中的第一个回答有 33 个键。

<details>
<summary>全部 32 个 ChatGPT 回答字段，<code>gd_m7aof0k82r803d5bjm</code></summary>

| 字段 | 类型 | 描述 |
| --- | --- | --- |
| `url` | URL | ChatGPT 页面 URL |
| `prompt` | 文本 | 输入的提示词 |
| `answer_html` | 文本 | ChatGPT 搜索的回答 |
| `answer_text` | 文本 | 纯文本格式的回答文本 |
| `links_attached` | 数组 | 回答文本中包含的超链接，以及其位置和链接文字。 |
| `citations` | 数组 | 回答中直接引用的来源，通常链接到具体事实或主张。 |
| `country` | 国家/地区 | 执行搜索的国家/地区 |
| `is_map` | 布尔值 | 是否生成了地图包结果 |
| `references` | 数组 | 归入“参考资料”部分的链接 |
| `shopping` | 数组 | ChatGPT“购物”面板中的商品推荐卡片 |
| `shopping_visible` | 布尔值 | 此回答是否显示了“购物”面板 |
| `index` | 数字 | 索引列允许你为每个爬取请求分配唯一 ID，以便轻松跟踪。 |
| `answer_text_markdown` | html2markdown | Markdown 格式的回答文本。 |
| `web_search_triggered` | 布尔值 | ChatGPT 返回回答时是否触发了网页搜索 |
| `additional_prompt` | 文本 | additional_prompt 是收到第一个回答后发送的后续输入，旨在澄清、扩展或优化初始回答。系统会处理此提示词，并将结果作为 additional_answer_text 返回。 |
| `additional_answer_text` | 文本 | additional_answer_text 是系统对 additional_prompt 的回答，紧随初始交互之后。它会根据用户的后续输入提供更多信息、澄清内容或优化后的回答。 |
| `map` | 数组 | 地图包结果 |
| `search_sources` | 数组 | ChatGPT 用于收集信息的外部网页列表，通常显示在回答底部。 |
| `search_sources_more` | 数组 | ChatGPT 用于收集信息的外部网页列表，通常显示在回答底部（来自“更多”分区）。 |
| `response_raw` | 文本 | ChatGPT 回答的原始数据 |
| `response_raw_additional_prompt` | 文本 | ChatGPT 对后续提示词回答的原始数据。 |
| `answer_section_html` | 文本 | 回答区段的 HTML |
| `model` | 文本 | 回答所使用的模型 |
| `web_search_query` | 数组 | 模型生成的网页搜索查询（按执行顺序排列的精确字符串） |
| `prompt_sent_at` | 日期 | 提示词的精确时间戳 |
| `answer_text_raw` | 文本 | 提取的原始 answer_text，未进行截断或空白规范化（可能包含开头/结尾空格和换行符）。 |
| `business_locations` | 数组 | 请考虑添加新字段，并附上回答中提到的每个营业地点可获取的 JSON 示例。回答包含本地商家结果时返回的营业地点实体数组。每个条目包含商家 ID、坐标、名称、类别、地址、联系信息、评分、营业时间、图片和营业状态。 |
| `ads` | 数组 | 广告卡片数据数组。 |
| `additional_answer_text_markdown` | html2markdown | Markdown 格式的 additional_answer_text_markdown。 |
| `questionnaire` | 数组 | 问卷 |
| `shopping_additional_prompt` | 数组 | 后续提示词结果中来自 ChatGPT“购物”面板的商品推荐卡片 |
| `citations_additional_prompt` | 数组 | 后续提示词回答中直接引用的来源，通常链接到具体事实或主张。 |

</details>
<!-- fields:end -->

<details>
<summary>真实输出文件的开头，来自 <code>chatgpt-scraper "Best CRM software for small businesses"</code></summary>

此处每行均截断为 160 个字符；仅 `answer_html` 就是一行 1.4 MB 的内容。

```json
{
  "generated_at": "2026-11-09T14:00:41.695758+00:00",
  "country": "US",
  "prompts": [
    {
      "prompt": "Best CRM software for small businesses",
      "answer": {
        "url": "https://chatgpt.com/?q=Best%20CRM%20software%20for%20small%20businesses",
        "prompt": "Best CRM software for small businesses",
        "answer_html": "<html class=\"x108lcm5 xzx5cyk xojncy5 xy3g5xu x1nktip0 xwbwmuu x1it3fuk xh8yej3 xuu8av0 xeuugli xgs08xk x1javzw2 xfqjyog x1rob05q xvmah...
        "answer_text": "The best CRM software for a small business depends on whether you need to manage leads, track sales, automate follow-ups, manage custome...
        "links_attached": [
          {
            "url": "https://www.hubspot.com/products/crm?utm_source=chatgpt.com",
            "text": "HubSpot website",
            "position": 1
          },
          {
            "url": "https://www.zoho.com/crm/?utm_source=chatgpt.com",
            "text": "Zoho website",
  ...
```

包含一个回答及其所有字段的完整文件位于 [examples/sample_output.json](examples/sample_output.json)。

</details>

## ChatGPT 爬虫工具错误

| 你看到的内容 | 含义 |
| --- | --- |
| `API token required but not found.` | 在发出任何请求前以状态码 2 退出。请设置令牌，或运行 `bdata login`。 |
| `failed  'prompt': ...` | 以状态码 1 退出。API 为该提示词返回了错误行，而不是回答。消息来自 API。 |
| `failed  'prompt': timeout: snapshot ... not ready after 900s` | 以状态码 1 退出。请重新运行，或稍后使用 `fetch()` 获取快照。 |
| `failed  'prompt': HTTP 401: ...` | 令牌已设置，但不正确。 |

任何失败都会以状态码 1 退出，因此可安全地用运行结果控制脚本是否继续。

在函数中，相同情况如下所示：

| 你看到的内容 | 含义 |
| --- | --- |
| `TokenMissing` | 没有 `BRIGHTDATA_API_TOKEN`，也没有 `bdata login`。 |
| `ApiError: HTTP 400: ...validation_error...` | 输入包含 API 不接受的内容。消息会指出相应字段。 |
| 包含 `error` 键的行 | API 对某一个提示词的回答。其他行不受影响。 |
| `TimeoutError` | 任务超过了 900 秒等待时间。将 `timeout=` 传给 `run`，或使用 `trigger` 并稍后获取。 |

## ChatGPT MCP 服务器和 Claude Code 技能

无需 Python，也无需安装任何内容。粘贴这两行；第一行会打开一次浏览器，也可在 SSH 和 CI 中使用 `bdata login --device`：

```bash
npx -p @brightdata/cli bdata login
npx -p @brightdata/cli bdata pipelines run gd_m7aof0k82r803d5bjm collect_by_url --url "https://chatgpt.com/" --prompt "Best CRM software for small businesses" --country US --pretty
```

2026 年 11 月 9 日，1 条记录、19 个字段、耗时 75 秒。CLI 会省略值为 null 的字段。该记录约为 3 MB JSON，其中大部分是 `answer_html`。

`npx skills add brightdata/skills` 会让 Claude Code、Cursor 和 Codex 掌握这些命令和文档，之后即可使用自然语言操作。完整指南：[为你的编码智能体使用 Bright Data](https://docs.brightdata.com/quickstart-coding-agent)。

对于没有终端的托管助手，可以使用 [Bright Data MCP 服务器](https://github.com/bright-cn/brightdata-mcp#which-tool-to-use)。它通过其 `scrapers` 分组中的 `search_scrapers` 和 `run_scraper` 运行任意预构建爬虫工具，包括 ChatGPT：

    https://mcp.brightdata.com/mcp?token=YOUR_API_TOKEN&groups=scrapers

智能体也可以自行创建账户，无需填写注册表单：[智能体注册](https://brightdata.com/auth.md)。Bright Data 支持的其他所有连接，从 LangChain 到 Zapier 和 n8n，均可参见[集成](https://docs.brightdata.com/integrations/introduction)。

## 这个 ChatGPT 爬虫工具免费吗？

新的 Bright Data 账户每月可获得 [5,000 免费额度](https://docs.brightdata.com/general/account/billing-and-pricing/free-tier)。ChatGPT 爬虫按记录计费，每个回答对应一条记录。[定价](https://www.bright.cn/pricing/web-scraper)列出了费率。

## 支持

此代码库中的错误：[创建 issue](https://github.com/bright-cn/chatgpt-scraper/issues)，并参阅 [CONTRIBUTING.md](CONTRIBUTING.md) 了解需要提供哪些信息。有关 API、你的账户或额度的任何问题，请联系 [Bright Data 支持团队](https://brightdata.zendesk.com/hc/en-us/requests/new)。

## 许可证

[Apache 2.0](LICENSE)。

## Node.js 版 ChatGPT 爬虫工具

同一个 ChatGPT 抓取工具也提供 Node.js 版本：[bright-cn/chatgpt-scraper-node](https://github.com/bright-cn/chatgpt-scraper-node)。

## 更多 Bright Data 开源爬虫工具

其他以相同方式构建并包含实时验证示例的 Python 爬虫工具：

- [Python 版 URL 转 Markdown（Fetch API）](https://github.com/bright-cn/fetch-api-scraper)
- [Python 版 YouTube 爬虫工具](https://github.com/bright-cn/youtube-scraper)
- [Python 版 LinkedIn 爬虫工具](https://github.com/bright-cn/linkedin-scraper-python)

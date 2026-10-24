# web scraping python：构建你的第一个爬虫，以及何时切换到抓取 API 并计算积分成本

大多数关于 Python 网页抓取（web scraping python）的教程在相同的 15 行代码处结束：`requests.get()`，BeautifulSoup，完成。真正决定项目能否成功的部分发生在那之后——当目标网站开始限速、验证或直接封锁你的 IP 时。本指南涵盖了两个部分：核心的 Python 爬虫栈，以及从自行构建（DIY）转向抓取 API（以 ScraperAPI 为例）的合理时机——包括大多数人直到收到高额账单时才意识到的积分（credit）计算方式。

> **快速选择：** 从免费层级（free tier）开始——ScraperAPI 提供永久免费的 1,000 积分/月（5 个并发连接），外加 7 天的 5,000 积分试用，无需信用卡。 Hobby 计划（$49/月，10 万积分，20 个并发线程）覆盖了每月最多约 20 万页面的价格监控和 SERP 追踪需求。 👉 [免费试用 ScraperAPI — 5,000 积分，无需信用卡](https://www.scraperapi.com/?fp_ref=coupons)

## 为什么 Python 在网页抓取中占据主导地位

Python 占据主导地位是因为其技术栈几乎涵盖了所有抓取场景，而且每个层级都有详尽的文档记录。对于静态 HTML，`requests + BeautifulSoup` 是标准配置。对于大规模多页面爬取，可以使用 `Scrapy`。对于通过 JavaScript 渲染内容的网站，可以使用 `Playwright` 或 `Selenium`。

2026 年指南中流传着一个实用的升级规则，它是正确的：从 `requests + BeautifulSoup` 开始。当你需要爬取数千个页面，且需要内置的并发、重试和导出管道（export pipelines）时，再升级到 `Scrapy`。只有当目标数据不在初始 HTML 响应中时，才使用 `Playwright`——也就是说，网站使用 React 或 Vue，返回一个几乎为空的 HTML 外壳，并让 JavaScript 来填充内容。每一次升级都会以速度和复杂度为代价，所以不要提前升级。

有一种方法可以完全跳过浏览器阶段，但很少有初学者教程会提到：打开浏览器的开发者工具（DevTools）→ Network 标签页，找到页面本身调用的 JSON 端点，然后直接用 `requests` 请求它。这比驱动无头浏览器（headless browser）更快、更稳定。

检查数据是否是 JS 渲染的，方法很简单：用 `requests` 获取页面，然后在响应文本中搜索你的目标数据。如果浏览器中可见但在响应中缺失，说明你需要渲染。如果两者都有，说明你不需要。

## Python 中的基础爬虫（以及人们遇到的三个 Bug）

一个基础爬虫就是一个四步循环：抓取（fetch）、解析（parse）、提取（extract）、保存（save）。以下版本针对 `books.toscrape.com` 运行——这是一个专门为了练习而构建的沙箱，这样你在学习时就不会对真实网站进行高频请求：

python
import time
import requests
from bs4 import BeautifulSoup

headers = {"User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64)"}

for page in range(1, 3):
    url = f"https://books.toscrape.com/catalogue/page-{page}.html"
    res = requests.get(url, headers=headers, timeout=15)
    res.raise_for_status()

    soup = BeautifulSoup(res.text, "html.parser")
    for book in soup.select("article.product_pod"):
        title = book.select_one("h3 a")["title"]
        price = book.select_one("p.price_color").text
        print(title, price)

    time.sleep(1)


教程中通常会跳过三个细节，而它们正是真实爬虫与演示爬虫的区别：

1. **User-Agent 标头。** 默认值 `python-requests/2.x` 是一个报出“我是爬虫”的横幅，许多网站一看到就会封锁。发送浏览器 User-Agent 的重要性再怎么强调也不为过。
2. **`raise_for_status()`。** 如果没有它，403 或 404 错误页面会被默默地解析为正常页面，而你的提取器只会返回空数据，且不会报错——这种 bug 只会在你的数据集完成之后才会被发现。
3. **在请求之间添加延迟。** 每秒发送数百个请求是导致 IP 被限速或封锁的最快方式。在像上面这样的循环中，每一轮之间停留一秒左右，如果要将它指向一个你没有权限抓取的真实网站，请将延迟随机化。

对于分页，遵循 URL 模式（`page-1`，`page-2`……）或追踪“下一页”链接的 `href`，直到页面返回无结果或 404。不要硬编码总页数——网站会改变分页，导致你的爬虫失效。

另一个值得注意的限制：抓取违反网站服务条款（terms of use）的数据在法律上是一个灰色地带，并且如果违反了网站的可接受使用政策（acceptable use policy），这可能是非法的。在实际操作中，这意味着在针对任何真实网站启动爬虫之前，请检查该网站的可接受使用政策。

## 80% 的问题：爬虫被封锁

代码编写是简单的 20%。保持不被封锁是剩下的 80%，而且这部分一年比一年难。这是独立基准测试中非常明确的结论，数字证实了这一点：Scrapeway 2026 年 7 月的基准测试发现，即使是最好的商业反爬 API，在受保护的目标网站上的成功率也只能达到 99% 左右，而较便宜的工具则低得多——Proxyway 测量发现 Firecrawl 在受防御的网站上成功率仅约 33%。没有代理轮换（proxy rotation）的手写 Python 脚本表现要糟糕得多。

你可以添加一些权宜之计：轮换 User-Agent，使用 `session_number` 式的技巧保持会话，在请求之间留出足够的延迟。在某个时候，你需要一个真正的代理池、解决验证码的能力，以及重试逻辑——此时你要做出的决定是，是自己构建这些基础设施，还是为此付费。

## ScraperAPI 的具体功能

ScraperAPI 位于你的代码和目标网站之间。你发送一个带有 API key 和目标 URL 的 GET 请求，它会处理代理轮换、重试、验证码解决，以及（可选）JavaScript 渲染。从你的角度来看，这只是一个 HTTP 调用：

python
import requests

API_KEY = "your-api-key"
target = "https://example.com/products"

resp = requests.get(
    "https://api.scraperapi.com/",
    params={"api_key": API_KEY, "url": target},
    timeout=60,
)
resp.raise_for_status()
soup = BeautifulSoup(resp.text, "html.parser")
# same extraction logic as before


你从基础爬虫中删掉了解析器调用之上的所有内容——代理、UA、重试——因为 API 已经为你处理了这些。可用的参数包括用于国家定位的 `country_code`，用于无头浏览器渲染的 `render=true`，用于更难封锁的住宅/移动 IP 池的 `premium=true` 和 `ultra_premium=true`，以及用于在大序列请求中保持相同 IP 的 `session_number`。

还有一个功能节省了大量实际工程时间：针对 Amazon 产品和搜索页面、Google Search 和 Shopping 以及 Walmart 的结构化端点（structured endpoints）返回解析后的 JSON，而不是原始 HTML。当 Amazon 重新调整其标记（markup）时，你无需维护自己的解析器。

IP 池公开标称为超过 4000 万个 IP，涵盖住宅、数据中心和移动网络，覆盖 50 多个国家。

## 计划和定价（所有计划，已对照官方定价页面核实）

这是在撰写本文时，ScraperAPI 定价页面上当前显示的完整计划列表。所有付费计划都包含相同的功能集——JS 渲染、高级住宅和移动 IP、高级绕过（advanced bypassing）、结构化数据 API、DataPipeline、按量付费（pay-as-you-go）——差异在于积分量、并发和地理定位范围。

| 计划 | 月付价格 | 年付价格 (节省 10%) | 积分/月 | 并发线程 |
| --- | --- | --- | --- | --- |
| Free | $0 | — | 1,000 | 5 |
| Hobby | $49 | $44.10 | 100,000 | 20 |
| Startup | $149 | $134.10 | 1,000,000 | 50 |
| Business | $299 | $269.10 | 3,000,000 | 100 |
| Scaling | $475 | $427.50 | 5,000,000 | 200 |
| Professional | $975 | $877.50 | 10,500,000 | 300 |
| Advanced | $1,975 | $1,777.50 | 21,500,000 | 500 |
| Enterprise | Custom | Custom | 22,000,000+ | 500+ |

两种低成本计划在地理定位上有限制：Hobby 和 Startup 计划仅支持定位美国和欧盟。从 Business 计划开始解锁国家级定位。另一个级别差异存在于分析面板：在 Hobby 和 Startup 上，你的使用数据只能回溯 30 天；从 Business 计划开始，该限制取消。如果你在运行需要几个月趋势数据的持续性监控工作，这值得注意。

Free 计划是一个真正的常设层级——1,000 积分/月，永久有效，注册后可用——而 7 天试用会增加 5,000 积分，因此你无需提交卡号即可针对真实目标测试 API。 👉 [查看所有计划的当前定价](https://www.scraperapi.com/pricing/?fp_ref=coupons)

## 大多数文章跳过的部分：积分数学计算

每个抓取 API 都是以积分（credit）计费的，这就是 ScraperAPI 的标价与实际成本背离的地方。常规标准页面花费 1 积分。但是：

- **Amazon 页面成本为 5 积分。**
- **Google 和 Bing（包括其子域名）成本为 25 积分。**
- **LinkedIn 成本为 30 积分。**
- **受 Cloudflare、Datadome 或 PerimeterX 保护的网站在 ScraperAPI 绕过它们时，每次请求会增加 10 积分。**
- **开启 `render=true` 会使单次请求成本跳升至 10 积分左右。**

结合使用这些选项，实际成本会急剧上升。一份独立的评测报告将计算方式说得很清楚：对于一个需要在受保护目标上同时开启 `render=true` 和 `premium=true` 的任务，单次成功抓取的成本可能是每积分标价的 20–30 倍。如果开启 JS 渲染抓取 Amazon 搜索结果页面，每月 100 万的积分可能只能覆盖 4 万到 10 万个实际页面，而不是 100 万个。

有两点可以让这一切变得可控，而这两点都在仪表盘（dashboard）中：**Domain Cost Estimator** 可以在运行之前告诉你任何 URL 的确切积分成本，并且你可以在每次抓取时设置 `max_cost`，这样单次请求就永远不会消耗超过设定的积分预算。在承诺任何一个计划之前，请针对你的实际目标运行估算器，并根据真实的单页成本来制定预算，而不是根据定价页面上的积分数量。

## 条款细则：积分用完、超额和会话

这三个细节很少出现在评测文章中，但却会改变你使用该服务的方式：

- **当你的积分用完时，默认情况下抓取会暂停，直到积分更新。** Hobby 到 Professional 计划提供了一个可选开启的 **auto-upgrade**，当你的积分用完时会自动将你移动到下一个计划——这是自动选择加入的，所以如果你让一个长时间运行的任务处于无人值守状态，请务必检查该设置。Scaling 及更高版本则提供 **pay-as-you-go** 选项，以固定的每积分费率继续抓取，最高可达你设定的月度上限。
- **超过并发限制不会让你进入等待队列。** 你会收到一个 429 响应，并且请求会被直接拒绝。这里没有超支费用，但一个每天要处理数万个页面且只有 20 个线程的作业（Hobby 计划的上限）会丢弃请求，而不是将它们延迟处理。这就是 $49 标价经常不代表实际月度支出的原因之一。
- **粘性会话（Sticky sessions）持续时间约为 1 小时。** `session_number` 是为单次抓取运行设计的，而不是为了长期保持相同的 IP。对于需要持续身份识别的账号管理或广告验证类抓取，它并不是合适的工具。
- **你无法选择特定的 IP 或 ASN。** 定位是国家级的（在较高计划中有时是城市级的），因此如果特定的 IP 范围被目标网站封锁，你无法进行干预。

## ScraperAPI vs. ZenRows vs. 自行构建

鉴于人们对“哪种抓取 API 实际有效”的兴趣，以下是通过公开定价页面核实过的对比。所有数字均为每月定价。

|  | ScraperAPI | ZenRows | 自行构建 (requests + BS4) |
| --- | --- | --- | --- |
| 免费层级 | 1,000 积分/月 + 5,000 积分 7 天试用 | 每月 5,000 积分，永久有效 | 仅你的基础设施 |
| 入门付费档 | $49/月，10 万积分，20 线程 | $16/月（Build 计划），45K 积分，20 并发 | $0 + 你的代理费用 |
| 中端付费档 | $299/月，300 万积分，100 线程（Business） | $58/月（Launch 计划），250K 积分，50 并发 | 不适用 |
| JS 渲染成本 | ~10 积分/请求 | 5 积分/请求 | Playwright，运行时成本 |
| 高级代理成本 | 10–30 积分/请求 | 10 积分/请求（JS + 高级代理：25） | 不适用 |
| 成本透明度 | 用于每 URL 成本的 Domain Cost Estimator | 发布每目标成功率 | 完全透明，手动操作 |

ZenRows 的入局成本更低，并且在免费层级上每月赠送的积分更多，而 ScraperAPI 的结构化端点（Amazon、Google、Walmart JSON）以及 Domain Cost Estimator 是大多数直接竞争者所不具备的差异化优势。根据一份独立汇总，在重度保护的页面上，ScraperAPI 的独立测得成功率接近 62–64%，而排名最高的 API 则在 99% 左右——因此，对于处于严重 Cloudflare 或 Akamai 防御之下的目标，ScraperAPI 并不是最合适的选择。对于标准的电商和搜索目标，它的默认池无需昂贵的 premium 层级即可处理大多数请求，而这正是大多数项目实际所在的地方。

## 谁不需要此功能

一些坦诚的排除项，因为评测文章中很少提及这些：

- **低频、静态 HTML 的需求。** 如果你每月抓取少于 1 万个不设防的页面，`requests + BeautifulSoup` 加上 `time.sleep()` 将为你免费搞定这一切，而不会产生任何订阅费用。
- **重度反爬虫目标。** 如果你的目标是 Akamai 或企业级的 Cloudflare bot management，独立测试表明 ScraperAPI 的成功率处于中等水平。最先寻找具有更高已验证成功率的服务吧。
- **需要长期 IP 身份识别的作业。** 约 1 小时的粘性会话上限排除了账号运营和需要长期会话的广告验证类工作。
- **需要原始代理控制的人。** 没有仪表盘可以挑选特定的 IP 或子网；你信任的是轮换逻辑。如果你需要这种粒度，最好使用代理提供商，而不是抓取 API。

评测作者精准地指出了它的最佳定位：单人开发者或小团队运行价格监控、SERP 追踪器或针对标准电商和搜索目标的线索挖掘爬虫（lead-gen scraper），月页面抓取量大约在 2 万到 20 万之间，且不想自己构建和维护代理轮换及重试逻辑。 👉 [开启 Hobby 计划 ($49/月) 并立即获取你的 API key](https://www.scraperapi.com/?fp_ref=coupons)

## 常见问题

**Python 适合进行网页抓取吗？**
是的——简化的语法结合 `requests`、`BeautifulSoup`、`Scrapy` 和 `Selenium`/`Playwright`，使 Python 成为任何规模抓取任务中最实用的语言选择。

**ScraperAPI 有免费试用吗？**
有。有一个永久免费的计划，每月提供 1,000 积分，最大 5 个并发连接，外加一个 7 天试用，在此期间你可获得 5,000 积分，且无需信用卡。

**ScraperAPI 的费用是多少？**
Hobby 计划为 $49/月（年付为 $44.10/月），包含 10 万积分和 20 个并发线程。计划范围最高达 Advanced 计划的 $1,975/月（包含 2150 万积分），Enterprise 计划为定制报价。

**我的爬虫每月实际可以抓取多少页面？**
这完全取决于你的目标。标准页面花费 1 积分，Amazon 5 积分，Google/Bing 25 积分，LinkedIn 30 积分，被保护的网站每次绕过请求会增加 10 积分，而 `render=true` 大约增加 10 积分。在进行任何操作之前，请使用仪表盘中的 Domain Cost Estimator 来检查你的具体 URL。

**如果没有信用卡，我可以使用它吗？**
可以——免费计划和 7 天试用都不需要信用卡。

## 最后一步行动指南

按场景简要说明：如果只需要偶尔抓取不设防的页面，请保持使用 `requests + BeautifulSoup`，不要再扩展技术栈了。如果你的月抓取量在 2 万至 20 万个标准电商或 SERP 页面之间，Hobby 计划是合理的起点，但如果你的目标包括 Amazon 或 Google，在确定规模前，请根据 Domain Cost Estimator 的数据重新计算积分，而不是按照页数计算。如果你的目标位于重度反爬虫系统之下，请寻找具备更高独立测得成功率的服务商，而不是从最便宜的选项开始。如果夜间作业的中断会让你感到困扰，请开启 auto-upgrade，或者直接选择带有 pay-as-you-go 选项的计划。

在提交前值得检查的一点：针对你计划抓取的每个域名，运行一次 Domain Cost Estimator，并确认并发线程数符合你的计划吞吐量。完成这些之后， 👉 [注册 7 天免费试用，并在付费前测试你的真实目标](https://www.scraperapi.com/?fp_ref=coupons)

&emsp;

**概要：**

本文介绍了如何使用 Python 和 Playwright 库，分两步高效爬取 Seeking Alpha 网站上上市公司的财报电话会议（Earnings Call Transcripts）文本。第一步按公司代码（Ticker）收集所有相关会议的链接，第二步逐条访问这些链接并提取会议正文。文章详细解释了每个步骤的技术实现，包括动态滚动加载、验证码处理、异常管理和断点续爬等关键环节。通过这种方法，研究人员可以系统获取高质量的财报电话会议文本数据，用于金融与会计领域的实证分析。


- **Title**: Python 爬虫：爬取 Seeking Alpha 的财报电话会议文本
- **Keywords**: 爬虫, Conference Calls, Playwright, B861

&emsp;

**概要：**

本文介绍了如何使用 Python 和 Playwright 库，分两步高效爬取 Seeking Alpha 网站上上市公司的财报电话会议（Earnings Call Transcripts）文本。第一步按公司代码（Ticker）收集所有相关会议的链接，第二步逐条访问这些链接并提取会议正文。文章详细解释了每个步骤的技术实现，包括动态滚动加载、验证码处理、异常管理和断点续爬等关键环节。通过这种方法，研究人员可以系统获取高质量的财报电话会议文本数据，用于金融与会计领域的实证分析。


&emsp;

## 1. 财报电话会议文本是什么？

财报电话会是信息含量极高的“一手文本”，在金融与会计研究中用途广泛。在以往的研究中已经有如下的用法：

- **文本情绪/基调（tone）**：管理层措辞的积极/消极、确定性/不确定性，常用于解释公告后**超额收益**或**波动**的横截面差异。
- **前瞻性指引与软信息**：对未来收入、成本、产能、订单、库存、宏观风险的描述，为**盈利预测误差**、**分析师分歧**、**机构持仓调整**等研究提供文本特征。
- **行业/主题抽取**：基于 topic model 或 embedding，挖掘公司/行业受关注主题与结构性变化，支撑**因子构建**与**行业轮动**研究。
- **管理层风格与信誉**：语言复杂度、模糊度、夸张型修辞、回避式回答等指标，常用于解释**盈余管理、信息不对称、公司治理**等议题。
- **事件研究**：结合公告窗口的价格与量，评估信息含量与市场反应。

以下是一些使用 Earning Call Transcripts 数据进行的研究：

1. Chen, J., Demers, E., & Lev, B. (2018). Oh What a Beautiful Morning! Diurnal Influences on Executives and Analysts: Evidence from Conference Calls. Management Science, 64(12), 5899–5924. [Link](https://doi.org/10.1287/mnsc.2017.2888), [PDF](http://sci-hub.ren/10.1287/mnsc.2017.2888), [Google](<https://scholar.google.com/scholar?q=Oh What a Beautiful Morning! Diurnal Influences on Executives and Analysts: Evidence from Conference Calls>).

2. Kimbrough, M. D. (2005). The effect of conference calls on analyst and market underreaction to earnings announcements. _The Accounting Review_, 80(1), 189–219. [Link](https://doi.org/10.2308/accr.2005.80.1.189), [PDF](http://sci-hub.ren/10.2308/accr.2005.80.1.189), [Google](<https://scholar.google.com/scholar?q=The effect of conference calls on analyst and market underreaction>).

3. Cohen, L., Lou, D., & Malloy, C. J. (2020). Casting Conference Calls. Management Science, 66(11), 5015–5039. [Link](https://doi.org/10.1287/mnsc.2019.3423), [PDF](http://sci-hub.ren/10.1287/mnsc.2019.3423), [Google](<https://scholar.google.com/scholar?q=Casting Conference Calls>).

4. Chen, J. V., Nagar, V., & Schoenfeld, J. (2018). Manager-analyst conversations in earnings conference calls. Review of Accounting Studies, 23(4), 1315–1354. [Link](https://doi.org/10.1007/s11142-018-9453-3), [PDF](http://sci-hub.ren/10.1007/s11142-018-9453-3), [Google](<https://scholar.google.com/scholar?q=Manager-analyst conversations in earnings conference calls>).

5. Price, S. M., Doran, J. S., Peterson, D. R., & Bliss, B. A. (2012). Earnings conference calls and stock returns: The incremental informativeness of textual tone. _Journal of Banking & Finance_, 36(4), 992–1011. [Link](https://doi.org/10.1016/j.jbankfin.2011.10.013), [PDF](http://sci-hub.ren/10.1016/j.jbankfin.2011.10.013), [Google](<https://scholar.google.com/scholar?q=Earnings conference calls and stock returns textual tone>).

总之，Transcript 是“可计算的基本面文本数据源”，对于**资产定价、文本因子、ESG、公司金融、审计与信息披露**等方向都很有价值。我们在公司的投资者关系页面可以获得公司最新的 Earning Call Transcript，但在研究中我们往往需要用到公司以往的大量 Transcript 文本，而一些 Transcript 的聚合服务商就提供了这样一个途径，Seeking Alpha 就是平台之一。

Seeking Alpha Premium 的 Transcript 是其团队从上市公司财报电话会议的音频中，通过专业转录和严格校对后生成的**高质量、可搜索、可阅读的逐字文字记录**。不仅是一份原始的对话文本，更是一个集成了强大工具的分析平台，旨在帮助投资者从这些最重要的公司沟通中高效地提取关键信息，对于研究者而言，也提供了一份不可多得的高质量研究数据。下面，本文将给出详细的 Transcript 的爬取策略和代码。

> 本文所讨论的技术方法旨在用于**个人教育、研究及合理使用**范畴，旨在提升信息获取的效率。请您务必遵守以下原则：
>
> 1. **尊重知识产权**：Seeking Alpha 平台的所有内容，包括但不限于财报会议文字记录（Transcripts）、新闻文章及分析评论，均受版权法及其他相关知识产权法的保护。这些内容是 Seeking Alpha 及其供稿人的宝贵资产。
> 2. **合法获取内容**：**Seeking Alpha Premium 会员服务**是获取并使用其内容的**唯一合法与授权途径**。我们强烈建议并支持您通过订阅官方会员服务来支持平台的持续运营与高质量内容的产出。
> 3. **禁止商业性滥用**：本文介绍的自动化技术**严禁**用于任何形式的**商业性数据爬取、大规模复制、重新分发或数据贩卖**行为。此类行为不仅严重侵犯版权，也可能违反 Seeking Alpha 的用户协议，并可能导致法律责任。
> 4. **遵循 robots.txt 与访问频率**：即使为个人用途，任何自动化访问也应严格遵守 Seeking Alpha 网站的`robots.txt`协议，并应将请求频率限制在友好的范围内，**避免对 Seeking Alpha 的服务器造成任何不必要的负担或干扰**。恶意、高频的爬取行为是不道德的，也可能是非法的。

&emsp;

&emsp;

## 2. 爬取策略：两步走

爬取的总体思路如下：

- **步骤 A（按 Ticker 收集链接）**：按公司代码逐个访问其 Transcript 列表页，**滚动加载**抓取所有文章链接，保存为 `{ticker}.json`文件用于下一步爬取。
- **步骤 B（按链接下载正文）**：读取每个 `{ticker}.json`，逐条访问文章页，提取正文并保存为 `.txt` 文件。并记录失败链接用于第二次爬取。

为什么要采取两步爬取策略呢？主要有以下几点考虑：

- 如果想要拿到所有的 Transcript 文本，Seeking Alpha 中显示所有 Transcript 的页面最多只能支持爬取一年左右的数据，而按照 Tikcer 爬取则无此限制。
- 拿到所有文章链接后，可以对不同的 ticker 分批、并发或者不同设备执行文章的下载。
- 文章链接的清单与文本的爬取分离执行、保存，结构清晰，容易排查问题和去重复。

&emsp;

## 3. 爬取程序必要库

本文使用的主要 Python 库包括：

- `playwright.async_api`：自动化浏览器，抓取动态加载页面（如无限滚动、按钮点击）。
- `asyncio / nest_asyncio`：异步并发与事件循环；`nest_asyncio.apply()` 让 Jupyter 交互环境下也能运行异步程序。
- `random / time / asyncio.sleep`：人类化的随机滚动与等待，降低被风控概率。
- `json / os / glob / re`：读写 JSON、创建目录、批量读文件、清洗文件名。
- `connect_over_cdp`：通过 `Chrome DevTools Protocol` 连接已启动的本机 Chrome（端口如 9222/9333），便于在**有界面**的浏览器中手动过验证码。

在正式开始前，请确保已完成以下准备工作：

1. 安装 `Playwright`：依次执行如下命令
   ```python
   pip install playwright
   playwright install
   ```
2. 启动 Chrome 远程调试：

   ```bash
   macOS:
   /Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome --remote-debugging-port=9222
   ```

   ```bash
   Windows(示例路径请按本机修改)
   "C:\Program Files\Google\Chrome\Application\chrome.exe" --remote-debugging-port=9222
   ```

3. 确保爬虫代码中的端口号（如 9222 或 9333）与实际启动浏览器的端口一致。

&emsp;

## 4. 代码详解

### 4.1 步骤 A: 按照 Ticker 收集 Transcript URL 链接

#### 4.1.1 导入必要的库和基础设置

```python
import time
import json
import random

import asyncio
from playwright.async_api import async_playwright
import time
import json
import random
import os

stop_signal = False  # 用于中断滚动线程

import nest_asyncio
nest_asyncio.apply()

```

在选择技术方案时，我们采用异步编程模式，核心目标是构建一个高性能的、便于未来扩展的并发爬虫系统，以实现对海量文本数据的高效抓取。异步编程的核心在于事件循环机制，它如同一个高效的中央调度器，负责在单个线程内协调所有网络请求任务：当一个请求处于等待状态时，CPU 资源会立即被调配去处理其他已准备就绪的任务，从而极大提升了 I/O 操作的效率和系统并发能力。

在标准 Python 环境中，我们使用`asyncio.run()`来启动和管理这个事件循环的生命周期。然而，在 Jupyter 这类交互式环境中，其内核自身已预置了一个持续运行的事件循环来支持实时计算，这会导致直接调用`asyncio.run()`时产生冲突报错。为了解决这一环境冲突，我们通过调用`nest_asyncio.apply()`来修补标准库，使现有循环能够无缝接纳新的异步任务，从而确保我们的高性能并发代码在交互式开发中也能顺畅运行。

### 4.1.2 模拟滚动获取链接

由于 Seeking Alpha 的 Transcript 链接是动态加载的，所以这一步是模拟用户滚动浏览网页的行为，以动态加载更多内容。具体来说，通过执行 JavaScript 代码来触发页面的滚轮下拉操作，每次滚动的距离和滚动后的等待时间都经过随机化处理，以更逼真地模仿人类操作模式，避免被反爬虫机制识别（即是这样，在长时间爬取后依然会触发反爬虫机制，解决方式在后文）。

在每次滚动完成后，函数会统计当前页面中所有符合目标选择器 `h3 a[data-test-id="post-list-item-title"]`（即 Seeking Alpha 列表页的文章标题链接元素）的数量。函数会持续这一“滚动-统计”的循环过程，直至连续两次检测到的文章链接数量不再增加，此时可判定所有可加载内容均已成功渲染，随即自动停止滚动并退出。

以下是具体实现代码：

```python
async def scroll_page_until_count_stops(page):
    prev_article_count = 0
    same_count = 0
    max_same = 3

    while True:
        await page.mouse.wheel(0, random.randint(1500, 4000))
        await asyncio.sleep(random.uniform(0.8, 1.5))

        article_links = await page.query_selector_all('h3 a[data-test-id="post-list-item-title"]')
        curr_article_count = len(article_links)

        if curr_article_count > prev_article_count:
            print(f"已加载 {curr_article_count} 个文章链接...")
            prev_article_count = curr_article_count
            same_count = 0
        else:
            same_count += 1
            if same_count >= max_same:
                print("连续多次未加载新链接，停止滚动。")
                break

```

### 4.1.3 人工验证

这一步定义一个函数用于自动检测并处理验证码。具体来说，在 5 秒内尝试定位并等待“按住”（英文语言环境下为“Press on”）按钮元素出现，若成功检测到，则暂停程序并提示用户在浏览器中手动完成验证；完成验证后，用户需在终端按回车键确认，程序随后等待页面重新加载并继续执行。若在 5 秒内未检测到该元素或出现任何异常，则判定为无验证码，程序直接继续执行后续操作。

以下是具体实现代码：

```python
async def wait_for_captcha_and_resume(page):
    """
    检测验证码页面，如果存在，则暂停程序等待用户手动解决。
    """
    print("正在检查是否存在验证码...")
    try:
        captcha_button = page.get_by_text("按住").first
        await captcha_button.wait_for(state="visible", timeout=5000)

        print("\n" + "="*50)
        print("检测到验证码！请在浏览器中手动完成验证。")
        print("完成后，请回到此终端并按 Enter 键继续。")
        print("="*50 + "\n")

        input("按下 Enter 键以继续...")

        await page.wait_for_load_state('domcontentloaded')
        print("验证码已解决，程序继续。")

    except Exception:
        print("未检测到验证码，继续执行。")
        pass

```

> 如果您的浏览器语言是英文，则检测并处理的验证码内容需要进行相应的修改，如下：
>
> ```python
> captcha_button = page.get_by_text("Press on").first
> ```

### 4.1.4 进度与异常记录

定义三个函数`mark_as_completed`、`mark_as_excluded`和`mark_as_failed`创建三个文本文件（`progress.txt`、`exclude_transcript.txt`、`failed.txt`）分别记录已成功爬取、无 Transcript 内容及爬取失败的股票代码（ticker），以此实现简易的断点续爬与状态管理机制。函数 `get_completed_tickers` 用于读取已完成集合，避免重复抓取；其余函数则分别将不同状态的 ticker 追加或初始化写入对应文件，形成基于文件的状态机，保障任务可中断恢复和问题追踪。

以下是具体的实现代码：

```python
def get_completed_tickers(progress_file="progress.txt"):
    if os.path.exists(progress_file):
        with open(progress_file, "r") as f:
            return set(line.strip() for line in f)
    return set()

def mark_as_completed(ticker, progress_file="progress.txt"):
    with open(progress_file, "a") as f:
        f.write(f"{ticker}\n")

def mark_as_excluded(ticker, exclude_file="exclude_transcript.txt"):
    if os.path.exists(exclude_file):
        with open(exclude_file, "a") as f:
            f.write(f"{ticker}\n")
    else:
        with open(exclude_file, "w") as f:
            f.write(f"{ticker}\n")

def mark_as_failed(ticker, failed_file="failed.txt"):
    if os.path.exists(failed_file):
        with open(failed_file, "a") as f:
            f.write(f"{ticker}\n")
    else:
        with open(failed_file, "w") as f:
            f.write(f"{ticker}\n")

```

### 4.1.5 主流程：连接浏览器-按 Ticker 收集链接

**Step 1. 连接浏览器与初始化状态管理**

函数首先通过 Playwright 的 `connect_over_cdp` 方法连接至本地已启动的 Chrome 浏览器实例（需开启远程调试端口）。随后读取三个状态文件，分别获取已完成、已排除和失败的股票代码（ticker），并计算需处理的新 ticker 列表，实现断点续爬功能。

```python
async def connect_and_scrape(ticker_list):
    print("正在连接 Chrome...")
    async with async_playwright() as p:
        try:
            browser = await p.chromium.connect_over_cdp("http://localhost:9333")
            context = browser.contexts[0]
        except Exception as e:
            print(f"连接失败: {e}")
            return

        completed_tickers = get_completed_tickers()
        excluded_tickers = get_completed_tickers(progress_file="exclude_transcript.txt")
        failed_tickers = get_completed_tickers(progress_file="failed.txt")

        all_processed_tickers = completed_tickers.union(excluded_tickers).union(failed_tickers)
        tickers_to_process = [t for t in ticker_list if t not in all_processed_tickers]

```

**Step 2. 预处理和文件夹创建**

检查是否有需要处理的 ticker，若无则直接退出。同时确保输出文件夹和记录文件（排除文件、失败文件）存在，为后续数据存储做准备。

```python
        # 连接下一步“预处理和文件夹创建”代码
        if not tickers_to_process:
                  print("所有 tickers 任务已完成、已排除或已失败，无需再次运行。")
                  return

              output_folder = "transcripts"
              if not os.path.exists(output_folder):
                  os.makedirs(output_folder)

              exclude_file = "exclude_transcript_.txt"
              if not os.path.exists(exclude_file):
                  with open(exclude_file, "w") as f:
                      pass
```

**Step 3. 页面处理与验证码检测**

循环处理每个 ticker：尝试复用已有页面（如果域名匹配），导航至对应 Transcripts 页面。调用 `wait_for_captcha_and_resume` 函数检测并处理可能出现的验证码，确保程序能在需要时等待人工干预。

```python
        for ticker in tickers_to_process:
            url = f'https://seekingalpha.com/symbol/{ticker}/earnings/transcripts'
            try:
                page = None
                pages = context.pages
                for existing_page in pages:
                    if "seekingalpha.com" in existing_page.url:
                        page = existing_page
                        break
                if not page:
                    page = await context.new_page()

                await page.goto(url, timeout=60000)
                await wait_for_captcha_and_resume(page)
```

**Step 4. 内容有效性检查与空数据处理**

检查页面是否存在指定的“Transcripts & Insights”按钮（选择器：`a[data-test-id="earnings/transcripts"]`）。如果不存在，则生成一个空 JSON 文件并将该 ticker 记录到排除列表中，跳过后续抓取步骤。

```python
                transcripts_button = await page.query_selector('a[data-test-id="earnings/transcripts"]')
                if not transcripts_button:
                    print(f"{ticker} 页面不包含 'Transcripts & Insights' 按钮。")
                    results = []
                    file_path = os.path.join(output_folder, f"{ticker}.json")
                    with open(file_path, "w", encoding='utf-8') as f:
                        json.dump(results, f, indent=2, ensure_ascii=False)
                    mark_as_excluded(ticker)
                    continue
```

**Step 5. 动态加载与数据提取**

等待目标内容加载完成后，自动滚动页面以触发动态加载更多文章。随后提取所有符合选择器 `h3 a[data-test-id="post-list-item-title"]` 的文章标题和链接，整理后保存为结构化 JSON 数据。

```python
                await page.wait_for_selector('h3 a[data-test-id="post-list-item-title"]', timeout=30000)
                await scroll_page_until_count_stops(page)

                article_links = await page.query_selector_all('h3 a[data-test-id="post-list-item-title"]')
                results = []
                for link_elem in article_links:
                    href = await link_elem.get_attribute('href')
                    title = await link_elem.inner_text()
                    if href:
                        clean_href = href.split('#')[0]
                        results.append({
                            "title": title.strip(),
                            "link": f"https://seekingalpha.com{clean_href}"
                        })

                file_path = os.path.join(output_folder, f"{ticker}.json")
                with open(file_path, "w", encoding='utf-8') as f:
                    json.dump(results, f, indent=2, ensure_ascii=False)
                mark_as_completed(ticker)
```

**Step 6. 异常处理与资源清理**

单个 ticker 处理过程中发生任何异常，会被捕获并记录到失败列表，程序继续处理下一个 ticker。所有任务完成后，正确关闭浏览器连接。

```python
            except Exception as e:
                print(f"抓取 {ticker} 失败: {e}")
                mark_as_failed(ticker)

        await browser.close()
        print("所有任务完成，浏览器已关闭。")
```

**Step 7. 启动步骤 A**

假设你已经有一个需要爬取的公司股票代码列表`ticker_list`，则将其导入列表并启动步骤 A 进行爬取。

```python
# 假设你已有一个 ticker_list，例如：
# ticker_list = ["AAPL", "MSFT", "NVDA", ...]
# 实际上这里的ticker_list已经放在 github 仓库中，是SeekingAlpha截止2025年9月份所有可获取的企业代码的codes.json文件
# 需要将codes.json文件导入后运行
asyncio.run(connect_and_scrape(ticker_list))
```

**Step 8. 以上主流程的完整代码如下**

```python
async def connect_and_scrape(ticker_list):
    print("正在连接 Chrome...")
    async with async_playwright() as p:
        try:
            browser = await p.chromium.connect_over_cdp("http://localhost:9333")
            context = browser.contexts[0]
        except Exception as e:
            print(f"连接失败: {e}")
            return

        completed_tickers = get_completed_tickers()
        excluded_tickers = get_completed_tickers(progress_file="exclude_transcript.txt")
        failed_tickers = get_completed_tickers(progress_file="failed.txt")

        all_processed_tickers = completed_tickers.union(excluded_tickers).union(failed_tickers)
        tickers_to_process = [t for t in ticker_list if t not in all_processed_tickers]

        if not tickers_to_process:
            print("所有 tickers 任务已完成、已排除或已失败，无需再次运行。")
            return

        print(f"将从 {len(tickers_to_process)} 个未完成的 tickers 开始爬取...")

        output_folder = "transcripts"
        if not os.path.exists(output_folder):
            os.makedirs(output_folder)
            print(f"已创建输出文件夹: {output_folder}")

        exclude_file = "exclude_transcript.txt"
        if not os.path.exists(exclude_file):
            with open(exclude_file, "w") as f:
                pass
        failed_file = "failed.txt"
        if not os.path.exists(failed_file):
            with open(failed_file, "w") as f:
                pass

        for ticker in tickers_to_process:
            url = f'https://seekingalpha.com/symbol/{ticker}/earnings/transcripts'

            try:
                page = None
                pages = context.pages
                for existing_page in pages:
                    if "seekingalpha.com" in existing_page.url:
                        page = existing_page
                        break

                if not page:
                    page = await context.new_page()

                print(f"\n正在处理 {ticker}...")
                print(f"导航到 {url}")
                await page.goto(url, timeout=60000)

                await wait_for_captcha_and_resume(page)

                print("检查 'Transcripts & Insights' 按钮是否存在...")
                transcripts_button = await page.query_selector('a[data-test-id="earnings/transcripts"]')

                if not transcripts_button:
                    print(f"{ticker} 页面不包含 'Transcripts & Insights' 按钮。")
                    print(f"为 {ticker} 创建一个空的 JSON 文件并标记为已排除。")

                    results = []
                    file_path = os.path.join(output_folder, f"{ticker}.json")
                    with open(file_path, "w", encoding='utf-8') as f:
                        json.dump(results, f, indent=2, ensure_ascii=False)

                    mark_as_excluded(ticker)
                    print(f"{ticker} 数据已成功保存至 {file_path} (无内容)。")
                    continue

                print("按钮存在，开始爬取。")

                print("等待页面加载文章内容...")
                await page.wait_for_selector('h3 a[data-test-id="post-list-item-title"]', timeout=30000)
                print("页面加载完毕。")

                print("开始自动滚动页面加载更多内容...")
                await scroll_page_until_count_stops(page)

                article_links = await page.query_selector_all('h3 a[data-test-id="post-list-item-title"]')
                print(f"共找到 {len(article_links)} 个文章链接 for {ticker}")

                results = []
                for link_elem in article_links:
                    href = await link_elem.get_attribute('href')
                    title = await link_elem.inner_text()

                    if href:
                        clean_href = href.split('#')[0]
                        results.append({
                            "title": title.strip(),
                            "link": f"https://seekingalpha.com{clean_href}"
                        })

                print(f"最终处理后找到 {len(results)} 个链接")

                file_path = os.path.join(output_folder, f"{ticker}.json")
                with open(file_path, "w", encoding='utf-8') as f:
                    json.dump(results, f, indent=2, ensure_ascii=False)

                mark_as_completed(ticker)
                print(f"{ticker} 数据已成功保存至 {file_path}")

            except Exception as e:
                print(f"抓取 {ticker} 失败: {e}")
                print(f"错误信息: {e}")
                mark_as_failed(ticker)

        await browser.close()
        print("所有任务完成，浏览器已关闭。")

```

### 4.2 步骤 B：按链接下载 Transcript 正文

> [!note]
> 这一步读取第一步的 `{ticker}.json`，逐条访问，抽取正文写入 `.txt`。同时记录失败的链接以便复爬。

#### 4.2.1 导入、工具与文件名清洗

定义一个函数 `sanitize_filename` 负责对字符串进行标准化处理，将其转换为合法的文件名。具体来说，通过正则表达式 `re.sub(r'[<>:"/\\|?*]', '_', title)` 识别并替换所有在常见操作系统中不允许作为文件名使用的字符（包括 `<`, `>`, `:`, `"`, `/`, `\`, `|`, `?`, `*`），将它们统一替换为下划线 `_`，确保从网页抓取的文章标题能够安全地用作本地文件名，避免因包含非法字符而导致的文件读写失败或系统错误。

```python
import os
import time
import glob
import json
import random
import re
import asyncio
from playwright.async_api import async_playwright
from playwright.async_api._generated import Page

def sanitize_filename(title):
    """
    Sanitizes a string to be a valid filename by replacing invalid characters.
    """
    return re.sub(r'[<>:"/\\|?*]', '_', title)

```

#### 4.2.2 人工验证

与 4.1.3 中的人工验证逻辑一致。在爬取正文过程中也可能会触发反爬虫机制。

```python
async def wait_for_captcha_and_resume(page: Page):
    """
    Checks for a captcha page and pauses the script for manual resolution.
    """
    print("Checking for CAPTCHA...")
    try:
        captcha_button = page.get_by_text("按住").first
        await captcha_button.wait_for(state="visible", timeout=5000)

        print("\n" + "="*50)
        print("检测到验证码！请在浏览器中手动完成验证")
        print("验证完成，请回到控制台并按回车键继续。")
        print("="*50 + "\n")

        await asyncio.to_thread(input, "按下回车键继续...")

        await page.wait_for_load_state('domcontentloaded')
        print("验证码已解决，继续执行。")

    except Exception:
        print("未检测到验证码，继续执行。")
        pass

```

#### 4.2.3 单页文本下载

首先需要说明的是，Earnings Call Transcript 的下载功能仅对 Seeking Alpha 的付费会员用户开放。因此，在正式运行爬虫程序之前，必须确保你拥有一个具备下载权限的有效会员账号。

接下来，在终端中启动 Chrome 浏览器，要保证启动时的端口号与你后续爬取文本时的端口相同（详细见前文中“启动浏览器步骤”）。手动访问 Seeking Alpha 官方网站，并使用你的会员账号完成登录操作。登录成功后，浏览器会自动保存当前的登录状态（包括 Cookie 与 Session 信息）。

由于登录凭证会被保留在浏览器会话中，后续的爬取程序即可直接复用当前登录状态，从而无需在每次执行爬虫脚本时重复登录。

**Step 1. 页面导航与验证码处理**

函数首先使用 Playwright Page 对象导航到指定的文章 URL，并设置超时时间。随后调用辅助函数检测并处理可能出现的验证码，确保页面可正常访问。

```python
async def download_transcript(page: Page, url: str, output_dir: str, title: str):
    try:
        print(f"导航到页面: {url}") # 导航到当前爬取的文章链接
        await page.goto(url, timeout=60000)
        await wait_for_captcha_and_resume(page)
```

**Step 2. 错误页面检测**

页面加载完成后，函数会检查是否存在 Seeking Alpha 的错误提示页面"Whoops, we’ve hit a bottom."（特征为包含特定选择器的元素）。如果检测到错误信息，则主动抛出异常，中断当前下载流程。

```python
        error_page_check = page.locator('h1[data-test-id="yikes-page-title"]')
        if await error_page_check.is_visible():
            raise Exception("在Seeking Alpha没有找到相应页面.")
```

**Step 3. 正文内容定位与提取**

函数使用精确的选择器 `div.T2G6W[data-test-id="content-container"]` 来定位正文容器，这种特定选择器避免了 Playwright 严格模式下的多元素匹配冲突。等待目标元素出现后，提取其文本内容。

```python
        transcript_locator = 'div.T2G6W[data-test-id="content-container"]'
        await page.wait_for_selector(transcript_locator, timeout=10000)
        transcript_div = page.locator(transcript_locator)
        transcript_text = await transcript_div.inner_text()
```

**Step 4. 内容保存与成功返回**

确保输出目录存在后，函数将获取到的文本内容保存为 UTF-8 编码的文本文件。文件名使用经过 sanitize_filename 处理后的安全标题，避免非法字符导致的保存失败。

```python
        os.makedirs(output_dir, exist_ok=True)
        file_path = os.path.join(output_dir, f"{title}.txt")
        with open(file_path, "w", encoding="utf-8") as f:
            f.write(transcript_text)
        print(f"Transcript保存到路径 {file_path}")
        return True
```

**Step 5. 异常处理与错误返回**

整个流程中的任何异常都会被捕获。函数返回一个包含 False 和错误信息的元组，供上层调用者记录到失败日志中，实现高效的错误处理和流程控制。

```python
    except Exception as e:
        return False, str(e)
```

#### 4.2.4 主流程：读取 JSON 并逐条下载

**Step 1. 初始化设置与浏览器连接**

函数首先定义输入 JSON 文件目录和输出主目录路径。然后通过 Playwright 连接到运行在 9222 端口的 Chrome 浏览器实例，实现浏览器复用。

```python
async def main():
    json_dir = "/Users/mac/Documents/Seekingalpha/test_json"
    main_output_dir = "/Users/mac/Documents/Seekingalpha/transcript"

    async with async_playwright() as p:
        try:
            browser = await p.chromium.connect_over_cdp("http://localhost:9222")
            context = browser.contexts[0]
            page = context.pages[0]
```

**Step 2. 文件遍历与预处理**

查找所有 JSON 文件并逐个处理。跳过空文件，并从文件名中提取 ticker 代码作为标识。

```python
    json_files = glob.glob(os.path.join(json_dir, "*.json"))

    for file_path in json_files:
        try:
            if os.stat(file_path).st_size == 0:
                print(f"跳过空JSON文件: {os.path.basename(file_path)}")
                continue

            file_name = os.path.basename(file_path)
            ticker = os.path.splitext(file_name)[0].split("_")[-1]
```

**Step 3. 输出目录与状态管理**

为每个 ticker 创建独立的输出目录，并初始化失败链接记录文件。检查已下载的文件以避免重复工作。

```python
            ticker_output_dir = os.path.join(main_output_dir, ticker)
            os.makedirs(ticker_output_dir, exist_ok=True)

            failed_links_file = os.path.join(ticker_output_dir, "failed.json")
            failed_links = []

            downloaded_files = glob.glob(os.path.join(ticker_output_dir, "*.txt"))
            print(f"\n--- 正在处理 Ticker: {ticker} ---")
```

**Step 4. 数据加载与过滤**

读取 JSON 文件中的文章链接数据，并进行基本验证。此处可添加条件过滤，如仅下载标题包含'Earnings'的文章。

```python
            with open(file_path, 'r', encoding='utf-8') as f:
                data = json.load(f)

            results = [(item.get('title'), item.get('link')) for item in data]

            for title, link in results:
                if not title or not link:
                    continue

                # 可选过滤条件：仅下载包含'Earnings'的文章
                if 'Earnings' in title:
```

**Step 5. 下载执行与去重检查**

对每条链接，首先检查是否已下载过（通过文件名判断），避免重复下载。使用清洗后的安全文件名进行保存。

```python
                safe_title = sanitize_filename(title)

                # 已下载跳过
                if any(f"{safe_title}.txt" == os.path.basename(f) for f in downloaded_files):
                    print(f"'{title}'的Transcript已经存在，跳过")
                    continue

                download_result = await download_transcript(page, clean_url, ticker_output_dir, safe_title)
```

**Step 6. 错误处理与延时控制**

捕获下载过程中的错误并记录到失败列表。每次请求后添加随机延时，避免过于频繁的访问。

```python
                if isinstance(download_result, tuple):
                    _, error_message = download_result
                    failed_links.append({
                        "title": title,
                        "link": clean_url,
                        "error": error_message
                    })

                delay = random.uniform(2, 5)
                print(f"Pausing for {delay:.2f} seconds...")
```

**Step 7. 启动步骤 B**

```python
# 运行正文下载
asyncio.run(main())
```

**Step 8. 主流程的完整代码如下**

```python
async def main():
    json_dir = "/Users/mac/Documents/RA/Olivia_Gu/text_analysis/scraper/7/scraper2.0/测试爬取文章/test_json"
    main_output_dir = "/Users/mac/Documents/RA/Olivia_Gu/text_analysis/scraper/7/scraper2.0/测试爬取文章/transcript"

    json_files = glob.glob(os.path.join(json_dir, "*.json"))
    print(f"Found {len(json_files)} JSON files to process.")

    async with async_playwright() as p:
        try:
            browser = await p.chromium.connect_over_cdp("http://localhost:9222")
            context = browser.contexts[0]
            page = context.pages[0]
        except Exception as e:
            print(f"❗ Failed to connect to browser instance: {e}")
            print("Please ensure you have run '.../Google Chrome --remote-debugging-port=9222 ...' in the terminal.")
            return

        for file_path in json_files:
            try:
                if os.stat(file_path).st_size == 0:
                    print(f"❗ Skipping empty file: {os.path.basename(file_path)}")
                    continue

                file_name = os.path.basename(file_path)
                ticker = os.path.splitext(file_name)[0].split("_")[-1]

                ticker_output_dir = os.path.join(main_output_dir, ticker)
                os.makedirs(ticker_output_dir, exist_ok=True)

                failed_links_file = os.path.join(ticker_output_dir, "failed.json")
                failed_links = []

                downloaded_files = glob.glob(os.path.join(ticker_output_dir, "*.txt"))

                print(f"\n--- Processing Ticker: {ticker} ---")

                with open(file_path, 'r', encoding='utf-8') as f:
                    data = json.load(f)

                results = [(item.get('title'), item.get('link')) for item in data]

                for title, link in results:
                    if not title or not link:
                        continue

                    safe_title = sanitize_filename(title)

                    # 已下载跳过
                    if any(f"{safe_title}.txt" == os.path.basename(f) for f in downloaded_files):
                        print(f"Transcript for '{title}' already exists. Skipping.")
                        continue

                    clean_url = link.split('#')[0]
                    if 'Earnings' in title:
                        download_result = await download_transcript(page, clean_url, ticker_output_dir, safe_title)

                        if isinstance(download_result, tuple):
                            _, error_message = download_result
                            failed_links.append({
                                "title": title,
                                "link": clean_url,
                                "error": error_message
                            })

                        delay = random.uniform(2, 5)
                        print(f"Pausing for {delay:.2f} seconds...")
                        await asyncio.sleep(delay)

                if failed_links:
                    with open(failed_links_file, 'w', encoding='utf-8') as f:
                        json.dump(failed_links, f, indent=4, ensure_ascii=False)
                    print(f"{len(failed_links)} failed links saved to {failed_links_file}")

            except json.JSONDecodeError:
                print(f"Error: {os.path.basename(file_path)} is not a valid JSON file. Skipping.")
                continue
            except Exception as e:
                print(f"An unexpected error occurred while processing {file_path}: {e}")

    print("\n--- All done. ---")

# 运行正文下载
asyncio.run(main())

```

&emsp;

## 5. 目录结构与输出示例

以下是运行代码输出的内容目录结构：

```yaml
project_root/
├─ transcripts/                 # 步骤 A 产出（每个 ticker 一个链接清单）
│  ├─ AAPL.json
│  ├─ MSFT.json
│  └─ ...
├─ transcript/                         # 步骤 B 产出（每个 ticker 一个文件夹）
│  ├─ AAPL/
│  │  ├─ 2024 Q2 Apple Inc. Earnings Call Transcript.txt
│  │  ├─ ...
│  │  └─ failed.json                   # 下载失败的链接列表
│  ├─ MSFT/
│  └─ ...
├─ progress.txt                 # 已完成的 tickers
├─ exclude_transcript.txt       # 无 transcripts 的 tickers
└─ failed.txt                   # 收集链接阶段失败的 tickers

```

&emsp;

## 6. 常见问题

根据我的实际运行经验，以下是一些常见问题及建议：

- **端口统一**：`connect_over_cdp("http://localhost:9222")` 与你启动 Chrome 的端口一致；爬取链接和爬取正文文本的浏览器端口统一，不要 步骤 A 用 9333、步骤 B 用 9222 。
- **验证码**：该流程按“**人工验证**”设计；若验证码频繁，可以考虑减慢爬取频率、增加随机等待时间，或使用**多个随机代理**（需遵守网站条款）。
- **标题过滤**：`if 'Earnings' in title:` 控制只下载“Earnings”类文章。若要全部下载，删除这行条件。
- **文件名冲突**：`sanitize_filename` 已处理非法字符，但同名不同文可能会覆盖上一个文件，建议再加上**发布日期**进行区分。
- **存储格式**：目前正文是 以`.txt`的格式储存。若后续需要做 NLP 处理，推荐转为 **JSONL/Parquet**（含 `{ticker, title, url, date, text}` 等字段），便于后续 pandas/pyarrow/duckdb 读取与并行处理。



&emsp;

## 1. 财报电话会议文本是什么？

财报电话会是信息含量极高的“一手文本”，在金融与会计研究中用途广泛。在以往的研究中已经有如下的用法：

- **文本情绪/基调（tone）**：管理层措辞的积极/消极、确定性/不确定性，常用于解释公告后**超额收益**或**波动**的横截面差异。
- **前瞻性指引与软信息**：对未来收入、成本、产能、订单、库存、宏观风险的描述，为**盈利预测误差**、**分析师分歧**、**机构持仓调整**等研究提供文本特征。
- **行业/主题抽取**：基于 topic model 或 embedding，挖掘公司/行业受关注主题与结构性变化，支撑**因子构建**与**行业轮动**研究。
- **管理层风格与信誉**：语言复杂度、模糊度、夸张型修辞、回避式回答等指标，常用于解释**盈余管理、信息不对称、公司治理**等议题。
- **事件研究**：结合公告窗口的价格与量，评估信息含量与市场反应。

以下是一些使用 Earning Call Transcripts 数据进行的研究：

1. Chen, J., Demers, E., & Lev, B. (2018). Oh What a Beautiful Morning! Diurnal Influences on Executives and Analysts: Evidence from Conference Calls. Management Science, 64(12), 5899–5924. [Link](https://doi.org/10.1287/mnsc.2017.2888), [PDF](http://sci-hub.ren/10.1287/mnsc.2017.2888), [Google](<https://scholar.google.com/scholar?q=Oh What a Beautiful Morning! Diurnal Influences on Executives and Analysts: Evidence from Conference Calls>).

2. Kimbrough, M. D. (2005). The effect of conference calls on analyst and market underreaction to earnings announcements. _The Accounting Review_, 80(1), 189–219. [Link](https://doi.org/10.2308/accr.2005.80.1.189), [PDF](http://sci-hub.ren/10.2308/accr.2005.80.1.189), [Google](<https://scholar.google.com/scholar?q=The effect of conference calls on analyst and market underreaction>).

3. Cohen, L., Lou, D., & Malloy, C. J. (2020). Casting Conference Calls. Management Science, 66(11), 5015–5039. [Link](https://doi.org/10.1287/mnsc.2019.3423), [PDF](http://sci-hub.ren/10.1287/mnsc.2019.3423), [Google](<https://scholar.google.com/scholar?q=Casting Conference Calls>).

4. Chen, J. V., Nagar, V., & Schoenfeld, J. (2018). Manager-analyst conversations in earnings conference calls. Review of Accounting Studies, 23(4), 1315–1354. [Link](https://doi.org/10.1007/s11142-018-9453-3), [PDF](http://sci-hub.ren/10.1007/s11142-018-9453-3), [Google](<https://scholar.google.com/scholar?q=Manager-analyst conversations in earnings conference calls>).

5. Price, S. M., Doran, J. S., Peterson, D. R., & Bliss, B. A. (2012). Earnings conference calls and stock returns: The incremental informativeness of textual tone. _Journal of Banking & Finance_, 36(4), 992–1011. [Link](https://doi.org/10.1016/j.jbankfin.2011.10.013), [PDF](http://sci-hub.ren/10.1016/j.jbankfin.2011.10.013), [Google](<https://scholar.google.com/scholar?q=Earnings conference calls and stock returns textual tone>).

总之，Transcript 是“可计算的基本面文本数据源”，对于**资产定价、文本因子、ESG、公司金融、审计与信息披露**等方向都很有价值。我们在公司的投资者关系页面可以获得公司最新的 Earning Call Transcript，但在研究中我们往往需要用到公司以往的大量 Transcript 文本，而一些 Transcript 的聚合服务商就提供了这样一个途径，Seeking Alpha 就是平台之一。

Seeking Alpha Premium 的 Transcript 是其团队从上市公司财报电话会议的音频中，通过专业转录和严格校对后生成的**高质量、可搜索、可阅读的逐字文字记录**。不仅是一份原始的对话文本，更是一个集成了强大工具的分析平台，旨在帮助投资者从这些最重要的公司沟通中高效地提取关键信息，对于研究者而言，也提供了一份不可多得的高质量研究数据。下面，本文将给出详细的 Transcript 的爬取策略和代码。

> 本文所讨论的技术方法旨在用于**个人教育、研究及合理使用**范畴，旨在提升信息获取的效率。请您务必遵守以下原则：
>
> 1. **尊重知识产权**：Seeking Alpha 平台的所有内容，包括但不限于财报会议文字记录（Transcripts）、新闻文章及分析评论，均受版权法及其他相关知识产权法的保护。这些内容是 Seeking Alpha 及其供稿人的宝贵资产。
> 2. **合法获取内容**：**Seeking Alpha Premium 会员服务**是获取并使用其内容的**唯一合法与授权途径**。我们强烈建议并支持您通过订阅官方会员服务来支持平台的持续运营与高质量内容的产出。
> 3. **禁止商业性滥用**：本文介绍的自动化技术**严禁**用于任何形式的**商业性数据爬取、大规模复制、重新分发或数据贩卖**行为。此类行为不仅严重侵犯版权，也可能违反 Seeking Alpha 的用户协议，并可能导致法律责任。
> 4. **遵循 robots.txt 与访问频率**：即使为个人用途，任何自动化访问也应严格遵守 Seeking Alpha 网站的`robots.txt`协议，并应将请求频率限制在友好的范围内，**避免对 Seeking Alpha 的服务器造成任何不必要的负担或干扰**。恶意、高频的爬取行为是不道德的，也可能是非法的。

&emsp;

&emsp;

## 2. 爬取策略：两步走

爬取的总体思路如下：

- **步骤 A（按 Ticker 收集链接）**：按公司代码逐个访问其 Transcript 列表页，**滚动加载**抓取所有文章链接，保存为 `{ticker}.json`文件用于下一步爬取。
- **步骤 B（按链接下载正文）**：读取每个 `{ticker}.json`，逐条访问文章页，提取正文并保存为 `.txt` 文件。并记录失败链接用于第二次爬取。

为什么要采取两步爬取策略呢？主要有以下几点考虑：

- 如果想要拿到所有的 Transcript 文本，Seeking Alpha 中显示所有 Transcript 的页面最多只能支持爬取一年左右的数据，而按照 Tikcer 爬取则无此限制。
- 拿到所有文章链接后，可以对不同的 ticker 分批、并发或者不同设备执行文章的下载。
- 文章链接的清单与文本的爬取分离执行、保存，结构清晰，容易排查问题和去重复。

&emsp;

## 3. 爬取程序必要库

本文使用的主要 Python 库包括：

- `playwright.async_api`：自动化浏览器，抓取动态加载页面（如无限滚动、按钮点击）。
- `asyncio / nest_asyncio`：异步并发与事件循环；`nest_asyncio.apply()` 让 Jupyter 交互环境下也能运行异步程序。
- `random / time / asyncio.sleep`：人类化的随机滚动与等待，降低被风控概率。
- `json / os / glob / re`：读写 JSON、创建目录、批量读文件、清洗文件名。
- `connect_over_cdp`：通过 `Chrome DevTools Protocol` 连接已启动的本机 Chrome（端口如 9222/9333），便于在**有界面**的浏览器中手动过验证码。

在正式开始前，请确保已完成以下准备工作：

1. 安装 `Playwright`：依次执行如下命令
   ```python
   pip install playwright
   playwright install
   ```
2. 启动 Chrome 远程调试：

   ```bash
   macOS:
   /Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome --remote-debugging-port=9222
   ```

   ```bash
   Windows(示例路径请按本机修改)
   "C:\Program Files\Google\Chrome\Application\chrome.exe" --remote-debugging-port=9222
   ```

3. 确保爬虫代码中的端口号（如 9222 或 9333）与实际启动浏览器的端口一致。

&emsp;

## 4. 代码详解

> 你可以下载本推文中的爬虫程序，并在本地 Jupyter Notebook 中运行。请确保已安装上述必要库，并正确配置了 Chrome 远程调试。
>
> - [download_transcript.ipynb](https://github.com/lianxhcn/data/blob/main/B861-SeekingAlpha_conference_call_sample_data/download_transcript.ipynb)
> - [scrape_transcript_url_by_ticker.ipynb](https://github.com/lianxhcn/data/blob/main/B861-SeekingAlpha_conference_call_sample_data/scrape_transcript_url_by_ticker.ipynb)

### 4.1 步骤 A: 按照 Ticker 收集 Transcript URL 链接

#### 4.1.1 导入必要的库和基础设置

```python
import time
import json
import random

import asyncio
from playwright.async_api import async_playwright
import time
import json
import random
import os

stop_signal = False  # 用于中断滚动线程

import nest_asyncio
nest_asyncio.apply()

```

在选择技术方案时，我们采用异步编程模式，核心目标是构建一个高性能的、便于未来扩展的并发爬虫系统，以实现对海量文本数据的高效抓取。异步编程的核心在于事件循环机制，它如同一个高效的中央调度器，负责在单个线程内协调所有网络请求任务：当一个请求处于等待状态时，CPU 资源会立即被调配去处理其他已准备就绪的任务，从而极大提升了 I/O 操作的效率和系统并发能力。

在标准 Python 环境中，我们使用`asyncio.run()`来启动和管理这个事件循环的生命周期。然而，在 Jupyter 这类交互式环境中，其内核自身已预置了一个持续运行的事件循环来支持实时计算，这会导致直接调用`asyncio.run()`时产生冲突报错。为了解决这一环境冲突，我们通过调用`nest_asyncio.apply()`来修补标准库，使现有循环能够无缝接纳新的异步任务，从而确保我们的高性能并发代码在交互式开发中也能顺畅运行。

### 4.1.2 模拟滚动获取链接

由于 Seeking Alpha 的 Transcript 链接是动态加载的，所以这一步是模拟用户滚动浏览网页的行为，以动态加载更多内容。具体来说，通过执行 JavaScript 代码来触发页面的滚轮下拉操作，每次滚动的距离和滚动后的等待时间都经过随机化处理，以更逼真地模仿人类操作模式，避免被反爬虫机制识别（即是这样，在长时间爬取后依然会触发反爬虫机制，解决方式在后文）。

在每次滚动完成后，函数会统计当前页面中所有符合目标选择器 `h3 a[data-test-id="post-list-item-title"]`（即 Seeking Alpha 列表页的文章标题链接元素）的数量。函数会持续这一“滚动-统计”的循环过程，直至连续两次检测到的文章链接数量不再增加，此时可判定所有可加载内容均已成功渲染，随即自动停止滚动并退出。

以下是具体实现代码：

```python
async def scroll_page_until_count_stops(page):
    prev_article_count = 0
    same_count = 0
    max_same = 3

    while True:
        await page.mouse.wheel(0, random.randint(1500, 4000))
        await asyncio.sleep(random.uniform(0.8, 1.5))

        article_links = await page.query_selector_all('h3 a[data-test-id="post-list-item-title"]')
        curr_article_count = len(article_links)

        if curr_article_count > prev_article_count:
            print(f"已加载 {curr_article_count} 个文章链接...")
            prev_article_count = curr_article_count
            same_count = 0
        else:
            same_count += 1
            if same_count >= max_same:
                print("连续多次未加载新链接，停止滚动。")
                break

```

### 4.1.3 人工验证

这一步定义一个函数用于自动检测并处理验证码。具体来说，在 5 秒内尝试定位并等待“按住”（英文语言环境下为“Press on”）按钮元素出现，若成功检测到，则暂停程序并提示用户在浏览器中手动完成验证；完成验证后，用户需在终端按回车键确认，程序随后等待页面重新加载并继续执行。若在 5 秒内未检测到该元素或出现任何异常，则判定为无验证码，程序直接继续执行后续操作。

以下是具体实现代码：

```python
async def wait_for_captcha_and_resume(page):
    """
    检测验证码页面，如果存在，则暂停程序等待用户手动解决。
    """
    print("正在检查是否存在验证码...")
    try:
        captcha_button = page.get_by_text("按住").first
        await captcha_button.wait_for(state="visible", timeout=5000)

        print("\n" + "="*50)
        print("检测到验证码！请在浏览器中手动完成验证。")
        print("完成后，请回到此终端并按 Enter 键继续。")
        print("="*50 + "\n")

        input("按下 Enter 键以继续...")

        await page.wait_for_load_state('domcontentloaded')
        print("验证码已解决，程序继续。")

    except Exception:
        print("未检测到验证码，继续执行。")
        pass

```

> 如果您的浏览器语言是英文，则检测并处理的验证码内容需要进行相应的修改，如下：
>
> ```python
> captcha_button = page.get_by_text("Press on").first
> ```

### 4.1.4 进度与异常记录

定义三个函数`mark_as_completed`、`mark_as_excluded`和`mark_as_failed`创建三个文本文件（`progress.txt`、`exclude_transcript.txt`、`failed.txt`）分别记录已成功爬取、无 Transcript 内容及爬取失败的股票代码（ticker），以此实现简易的断点续爬与状态管理机制。函数 `get_completed_tickers` 用于读取已完成集合，避免重复抓取；其余函数则分别将不同状态的 ticker 追加或初始化写入对应文件，形成基于文件的状态机，保障任务可中断恢复和问题追踪。

以下是具体的实现代码：

```python
def get_completed_tickers(progress_file="progress.txt"):
    if os.path.exists(progress_file):
        with open(progress_file, "r") as f:
            return set(line.strip() for line in f)
    return set()

def mark_as_completed(ticker, progress_file="progress.txt"):
    with open(progress_file, "a") as f:
        f.write(f"{ticker}\n")

def mark_as_excluded(ticker, exclude_file="exclude_transcript.txt"):
    if os.path.exists(exclude_file):
        with open(exclude_file, "a") as f:
            f.write(f"{ticker}\n")
    else:
        with open(exclude_file, "w") as f:
            f.write(f"{ticker}\n")

def mark_as_failed(ticker, failed_file="failed.txt"):
    if os.path.exists(failed_file):
        with open(failed_file, "a") as f:
            f.write(f"{ticker}\n")
    else:
        with open(failed_file, "w") as f:
            f.write(f"{ticker}\n")

```

### 4.1.5 主流程：连接浏览器-按 Ticker 收集链接

**Step 1. 连接浏览器与初始化状态管理**

函数首先通过 Playwright 的 `connect_over_cdp` 方法连接至本地已启动的 Chrome 浏览器实例（需开启远程调试端口）。随后读取三个状态文件，分别获取已完成、已排除和失败的股票代码（ticker），并计算需处理的新 ticker 列表，实现断点续爬功能。

```python
async def connect_and_scrape(ticker_list):
    print("正在连接 Chrome...")
    async with async_playwright() as p:
        try:
            browser = await p.chromium.connect_over_cdp("http://localhost:9333")
            context = browser.contexts[0]
        except Exception as e:
            print(f"连接失败: {e}")
            return

        completed_tickers = get_completed_tickers()
        excluded_tickers = get_completed_tickers(progress_file="exclude_transcript.txt")
        failed_tickers = get_completed_tickers(progress_file="failed.txt")

        all_processed_tickers = completed_tickers.union(excluded_tickers).union(failed_tickers)
        tickers_to_process = [t for t in ticker_list if t not in all_processed_tickers]

```

**Step 2. 预处理和文件夹创建**

检查是否有需要处理的 ticker，若无则直接退出。同时确保输出文件夹和记录文件（排除文件、失败文件）存在，为后续数据存储做准备。

```python
        # 连接下一步“预处理和文件夹创建”代码
        if not tickers_to_process:
                  print("所有 tickers 任务已完成、已排除或已失败，无需再次运行。")
                  return

              output_folder = "transcripts"
              if not os.path.exists(output_folder):
                  os.makedirs(output_folder)

              exclude_file = "exclude_transcript_.txt"
              if not os.path.exists(exclude_file):
                  with open(exclude_file, "w") as f:
                      pass
```

**Step 3. 页面处理与验证码检测**

循环处理每个 ticker：尝试复用已有页面（如果域名匹配），导航至对应 Transcripts 页面。调用 `wait_for_captcha_and_resume` 函数检测并处理可能出现的验证码，确保程序能在需要时等待人工干预。

```python
        for ticker in tickers_to_process:
            url = f'https://seekingalpha.com/symbol/{ticker}/earnings/transcripts'
            try:
                page = None
                pages = context.pages
                for existing_page in pages:
                    if "seekingalpha.com" in existing_page.url:
                        page = existing_page
                        break
                if not page:
                    page = await context.new_page()

                await page.goto(url, timeout=60000)
                await wait_for_captcha_and_resume(page)
```

**Step 4. 内容有效性检查与空数据处理**

检查页面是否存在指定的“Transcripts & Insights”按钮（选择器：`a[data-test-id="earnings/transcripts"]`）。如果不存在，则生成一个空 JSON 文件并将该 ticker 记录到排除列表中，跳过后续抓取步骤。

```python
                transcripts_button = await page.query_selector('a[data-test-id="earnings/transcripts"]')
                if not transcripts_button:
                    print(f"{ticker} 页面不包含 'Transcripts & Insights' 按钮。")
                    results = []
                    file_path = os.path.join(output_folder, f"{ticker}.json")
                    with open(file_path, "w", encoding='utf-8') as f:
                        json.dump(results, f, indent=2, ensure_ascii=False)
                    mark_as_excluded(ticker)
                    continue
```

**Step 5. 动态加载与数据提取**

等待目标内容加载完成后，自动滚动页面以触发动态加载更多文章。随后提取所有符合选择器 `h3 a[data-test-id="post-list-item-title"]` 的文章标题和链接，整理后保存为结构化 JSON 数据。

```python
                await page.wait_for_selector('h3 a[data-test-id="post-list-item-title"]', timeout=30000)
                await scroll_page_until_count_stops(page)

                article_links = await page.query_selector_all('h3 a[data-test-id="post-list-item-title"]')
                results = []
                for link_elem in article_links:
                    href = await link_elem.get_attribute('href')
                    title = await link_elem.inner_text()
                    if href:
                        clean_href = href.split('#')[0]
                        results.append({
                            "title": title.strip(),
                            "link": f"https://seekingalpha.com{clean_href}"
                        })

                file_path = os.path.join(output_folder, f"{ticker}.json")
                with open(file_path, "w", encoding='utf-8') as f:
                    json.dump(results, f, indent=2, ensure_ascii=False)
                mark_as_completed(ticker)
```

**Step 6. 异常处理与资源清理**

单个 ticker 处理过程中发生任何异常，会被捕获并记录到失败列表，程序继续处理下一个 ticker。所有任务完成后，正确关闭浏览器连接。

```python
            except Exception as e:
                print(f"抓取 {ticker} 失败: {e}")
                mark_as_failed(ticker)

        await browser.close()
        print("所有任务完成，浏览器已关闭。")
```

**Step 7. 启动步骤 A**

假设你已经有一个需要爬取的公司股票代码列表`ticker_list`，则将其导入列表并启动步骤 A 进行爬取。

```python
# 假设你已有一个 ticker_list，例如：
# ticker_list = ["AAPL", "MSFT", "NVDA", ...]
# 实际上这里的ticker_list已经放在 github 仓库中，是SeekingAlpha截止2025年9月份所有可获取的企业代码的codes.json文件
# 需要将codes.json文件导入后运行
asyncio.run(connect_and_scrape(ticker_list))
```

**Step 8. 以上主流程的完整代码如下**

```python
async def connect_and_scrape(ticker_list):
    print("正在连接 Chrome...")
    async with async_playwright() as p:
        try:
            browser = await p.chromium.connect_over_cdp("http://localhost:9333")
            context = browser.contexts[0]
        except Exception as e:
            print(f"连接失败: {e}")
            return

        completed_tickers = get_completed_tickers()
        excluded_tickers = get_completed_tickers(progress_file="exclude_transcript.txt")
        failed_tickers = get_completed_tickers(progress_file="failed.txt")

        all_processed_tickers = completed_tickers.union(excluded_tickers).union(failed_tickers)
        tickers_to_process = [t for t in ticker_list if t not in all_processed_tickers]

        if not tickers_to_process:
            print("所有 tickers 任务已完成、已排除或已失败，无需再次运行。")
            return

        print(f"将从 {len(tickers_to_process)} 个未完成的 tickers 开始爬取...")

        output_folder = "transcripts"
        if not os.path.exists(output_folder):
            os.makedirs(output_folder)
            print(f"已创建输出文件夹: {output_folder}")

        exclude_file = "exclude_transcript.txt"
        if not os.path.exists(exclude_file):
            with open(exclude_file, "w") as f:
                pass
        failed_file = "failed.txt"
        if not os.path.exists(failed_file):
            with open(failed_file, "w") as f:
                pass

        for ticker in tickers_to_process:
            url = f'https://seekingalpha.com/symbol/{ticker}/earnings/transcripts'

            try:
                page = None
                pages = context.pages
                for existing_page in pages:
                    if "seekingalpha.com" in existing_page.url:
                        page = existing_page
                        break

                if not page:
                    page = await context.new_page()

                print(f"\n正在处理 {ticker}...")
                print(f"导航到 {url}")
                await page.goto(url, timeout=60000)

                await wait_for_captcha_and_resume(page)

                print("检查 'Transcripts & Insights' 按钮是否存在...")
                transcripts_button = await page.query_selector('a[data-test-id="earnings/transcripts"]')

                if not transcripts_button:
                    print(f"{ticker} 页面不包含 'Transcripts & Insights' 按钮。")
                    print(f"为 {ticker} 创建一个空的 JSON 文件并标记为已排除。")

                    results = []
                    file_path = os.path.join(output_folder, f"{ticker}.json")
                    with open(file_path, "w", encoding='utf-8') as f:
                        json.dump(results, f, indent=2, ensure_ascii=False)

                    mark_as_excluded(ticker)
                    print(f"{ticker} 数据已成功保存至 {file_path} (无内容)。")
                    continue

                print("按钮存在，开始爬取。")

                print("等待页面加载文章内容...")
                await page.wait_for_selector('h3 a[data-test-id="post-list-item-title"]', timeout=30000)
                print("页面加载完毕。")

                print("开始自动滚动页面加载更多内容...")
                await scroll_page_until_count_stops(page)

                article_links = await page.query_selector_all('h3 a[data-test-id="post-list-item-title"]')
                print(f"共找到 {len(article_links)} 个文章链接 for {ticker}")

                results = []
                for link_elem in article_links:
                    href = await link_elem.get_attribute('href')
                    title = await link_elem.inner_text()

                    if href:
                        clean_href = href.split('#')[0]
                        results.append({
                            "title": title.strip(),
                            "link": f"https://seekingalpha.com{clean_href}"
                        })

                print(f"最终处理后找到 {len(results)} 个链接")

                file_path = os.path.join(output_folder, f"{ticker}.json")
                with open(file_path, "w", encoding='utf-8') as f:
                    json.dump(results, f, indent=2, ensure_ascii=False)

                mark_as_completed(ticker)
                print(f"{ticker} 数据已成功保存至 {file_path}")

            except Exception as e:
                print(f"抓取 {ticker} 失败: {e}")
                print(f"错误信息: {e}")
                mark_as_failed(ticker)

        await browser.close()
        print("所有任务完成，浏览器已关闭。")

```

### 4.2 步骤 B：按链接下载 Transcript 正文

> [!note]
> 这一步读取第一步的 `{ticker}.json`，逐条访问，抽取正文写入 `.txt`。同时记录失败的链接以便复爬。

#### 4.2.1 导入、工具与文件名清洗

定义一个函数 `sanitize_filename` 负责对字符串进行标准化处理，将其转换为合法的文件名。具体来说，通过正则表达式 `re.sub(r'[<>:"/\\|?*]', '_', title)` 识别并替换所有在常见操作系统中不允许作为文件名使用的字符（包括 `<`, `>`, `:`, `"`, `/`, `\`, `|`, `?`, `*`），将它们统一替换为下划线 `_`，确保从网页抓取的文章标题能够安全地用作本地文件名，避免因包含非法字符而导致的文件读写失败或系统错误。

```python
import os
import time
import glob
import json
import random
import re
import asyncio
from playwright.async_api import async_playwright
from playwright.async_api._generated import Page

def sanitize_filename(title):
    """
    Sanitizes a string to be a valid filename by replacing invalid characters.
    """
    return re.sub(r'[<>:"/\\|?*]', '_', title)

```

#### 4.2.2 人工验证

与 4.1.3 中的人工验证逻辑一致。在爬取正文过程中也可能会触发反爬虫机制。

```python
async def wait_for_captcha_and_resume(page: Page):
    """
    Checks for a captcha page and pauses the script for manual resolution.
    """
    print("Checking for CAPTCHA...")
    try:
        captcha_button = page.get_by_text("按住").first
        await captcha_button.wait_for(state="visible", timeout=5000)

        print("\n" + "="*50)
        print("检测到验证码！请在浏览器中手动完成验证")
        print("验证完成，请回到控制台并按回车键继续。")
        print("="*50 + "\n")

        await asyncio.to_thread(input, "按下回车键继续...")

        await page.wait_for_load_state('domcontentloaded')
        print("验证码已解决，继续执行。")

    except Exception:
        print("未检测到验证码，继续执行。")
        pass

```

#### 4.2.3 单页文本下载

首先需要说明的是，Earnings Call Transcript 的下载功能仅对 Seeking Alpha 的付费会员用户开放。因此，在正式运行爬虫程序之前，必须确保你拥有一个具备下载权限的有效会员账号。

接下来，在终端中启动 Chrome 浏览器，要保证启动时的端口号与你后续爬取文本时的端口相同（详细见前文中“启动浏览器步骤”）。手动访问 Seeking Alpha 官方网站，并使用你的会员账号完成登录操作。登录成功后，浏览器会自动保存当前的登录状态（包括 Cookie 与 Session 信息）。

由于登录凭证会被保留在浏览器会话中，后续的爬取程序即可直接复用当前登录状态，从而无需在每次执行爬虫脚本时重复登录。

**Step 1. 页面导航与验证码处理**

函数首先使用 Playwright Page 对象导航到指定的文章 URL，并设置超时时间。随后调用辅助函数检测并处理可能出现的验证码，确保页面可正常访问。

```python
async def download_transcript(page: Page, url: str, output_dir: str, title: str):
    try:
        print(f"导航到页面: {url}") # 导航到当前爬取的文章链接
        await page.goto(url, timeout=60000)
        await wait_for_captcha_and_resume(page)
```

**Step 2. 错误页面检测**

页面加载完成后，函数会检查是否存在 Seeking Alpha 的错误提示页面"Whoops, we’ve hit a bottom."（特征为包含特定选择器的元素）。如果检测到错误信息，则主动抛出异常，中断当前下载流程。

```python
        error_page_check = page.locator('h1[data-test-id="yikes-page-title"]')
        if await error_page_check.is_visible():
            raise Exception("在Seeking Alpha没有找到相应页面.")
```

**Step 3. 正文内容定位与提取**

函数使用精确的选择器 `div.T2G6W[data-test-id="content-container"]` 来定位正文容器，这种特定选择器避免了 Playwright 严格模式下的多元素匹配冲突。等待目标元素出现后，提取其文本内容。

```python
        transcript_locator = 'div.T2G6W[data-test-id="content-container"]'
        await page.wait_for_selector(transcript_locator, timeout=10000)
        transcript_div = page.locator(transcript_locator)
        transcript_text = await transcript_div.inner_text()
```

**Step 4. 内容保存与成功返回**

确保输出目录存在后，函数将获取到的文本内容保存为 UTF-8 编码的文本文件。文件名使用经过 sanitize_filename 处理后的安全标题，避免非法字符导致的保存失败。

```python
        os.makedirs(output_dir, exist_ok=True)
        file_path = os.path.join(output_dir, f"{title}.txt")
        with open(file_path, "w", encoding="utf-8") as f:
            f.write(transcript_text)
        print(f"Transcript保存到路径 {file_path}")
        return True
```

**Step 5. 异常处理与错误返回**

整个流程中的任何异常都会被捕获。函数返回一个包含 False 和错误信息的元组，供上层调用者记录到失败日志中，实现高效的错误处理和流程控制。

```python
    except Exception as e:
        return False, str(e)
```

#### 4.2.4 主流程：读取 JSON 并逐条下载

**Step 1. 初始化设置与浏览器连接**

函数首先定义输入 JSON 文件目录和输出主目录路径。然后通过 Playwright 连接到运行在 9222 端口的 Chrome 浏览器实例，实现浏览器复用。

```python
async def main():
    json_dir = "/Users/mac/Documents/Seekingalpha/test_json"
    main_output_dir = "/Users/mac/Documents/Seekingalpha/transcript"

    async with async_playwright() as p:
        try:
            browser = await p.chromium.connect_over_cdp("http://localhost:9222")
            context = browser.contexts[0]
            page = context.pages[0]
```

**Step 2. 文件遍历与预处理**

查找所有 JSON 文件并逐个处理。跳过空文件，并从文件名中提取 ticker 代码作为标识。

```python
    json_files = glob.glob(os.path.join(json_dir, "*.json"))

    for file_path in json_files:
        try:
            if os.stat(file_path).st_size == 0:
                print(f"跳过空JSON文件: {os.path.basename(file_path)}")
                continue

            file_name = os.path.basename(file_path)
            ticker = os.path.splitext(file_name)[0].split("_")[-1]
```

**Step 3. 输出目录与状态管理**

为每个 ticker 创建独立的输出目录，并初始化失败链接记录文件。检查已下载的文件以避免重复工作。

```python
            ticker_output_dir = os.path.join(main_output_dir, ticker)
            os.makedirs(ticker_output_dir, exist_ok=True)

            failed_links_file = os.path.join(ticker_output_dir, "failed.json")
            failed_links = []

            downloaded_files = glob.glob(os.path.join(ticker_output_dir, "*.txt"))
            print(f"\n--- 正在处理 Ticker: {ticker} ---")
```

**Step 4. 数据加载与过滤**

读取 JSON 文件中的文章链接数据，并进行基本验证。此处可添加条件过滤，如仅下载标题包含'Earnings'的文章。

```python
            with open(file_path, 'r', encoding='utf-8') as f:
                data = json.load(f)

            results = [(item.get('title'), item.get('link')) for item in data]

            for title, link in results:
                if not title or not link:
                    continue

                # 可选过滤条件：仅下载包含'Earnings'的文章
                if 'Earnings' in title:
```

**Step 5. 下载执行与去重检查**

对每条链接，首先检查是否已下载过（通过文件名判断），避免重复下载。使用清洗后的安全文件名进行保存。

```python
                safe_title = sanitize_filename(title)

                # 已下载跳过
                if any(f"{safe_title}.txt" == os.path.basename(f) for f in downloaded_files):
                    print(f"'{title}'的Transcript已经存在，跳过")
                    continue

                download_result = await download_transcript(page, clean_url, ticker_output_dir, safe_title)
```

**Step 6. 错误处理与延时控制**

捕获下载过程中的错误并记录到失败列表。每次请求后添加随机延时，避免过于频繁的访问。

```python
                if isinstance(download_result, tuple):
                    _, error_message = download_result
                    failed_links.append({
                        "title": title,
                        "link": clean_url,
                        "error": error_message
                    })

                delay = random.uniform(2, 5)
                print(f"Pausing for {delay:.2f} seconds...")
```

**Step 7. 启动步骤 B**

```python
# 运行正文下载
asyncio.run(main())
```

**Step 8. 主流程的完整代码如下**

```python
async def main():
    json_dir = "/Users/mac/Documents/RA/Olivia_Gu/text_analysis/scraper/7/scraper2.0/测试爬取文章/test_json"
    main_output_dir = "/Users/mac/Documents/RA/Olivia_Gu/text_analysis/scraper/7/scraper2.0/测试爬取文章/transcript"

    json_files = glob.glob(os.path.join(json_dir, "*.json"))
    print(f"Found {len(json_files)} JSON files to process.")

    async with async_playwright() as p:
        try:
            browser = await p.chromium.connect_over_cdp("http://localhost:9222")
            context = browser.contexts[0]
            page = context.pages[0]
        except Exception as e:
            print(f"❗ Failed to connect to browser instance: {e}")
            print("Please ensure you have run '.../Google Chrome --remote-debugging-port=9222 ...' in the terminal.")
            return

        for file_path in json_files:
            try:
                if os.stat(file_path).st_size == 0:
                    print(f"❗ Skipping empty file: {os.path.basename(file_path)}")
                    continue

                file_name = os.path.basename(file_path)
                ticker = os.path.splitext(file_name)[0].split("_")[-1]

                ticker_output_dir = os.path.join(main_output_dir, ticker)
                os.makedirs(ticker_output_dir, exist_ok=True)

                failed_links_file = os.path.join(ticker_output_dir, "failed.json")
                failed_links = []

                downloaded_files = glob.glob(os.path.join(ticker_output_dir, "*.txt"))

                print(f"\n--- Processing Ticker: {ticker} ---")

                with open(file_path, 'r', encoding='utf-8') as f:
                    data = json.load(f)

                results = [(item.get('title'), item.get('link')) for item in data]

                for title, link in results:
                    if not title or not link:
                        continue

                    safe_title = sanitize_filename(title)

                    # 已下载跳过
                    if any(f"{safe_title}.txt" == os.path.basename(f) for f in downloaded_files):
                        print(f"Transcript for '{title}' already exists. Skipping.")
                        continue

                    clean_url = link.split('#')[0]
                    if 'Earnings' in title:
                        download_result = await download_transcript(page, clean_url, ticker_output_dir, safe_title)

                        if isinstance(download_result, tuple):
                            _, error_message = download_result
                            failed_links.append({
                                "title": title,
                                "link": clean_url,
                                "error": error_message
                            })

                        delay = random.uniform(2, 5)
                        print(f"Pausing for {delay:.2f} seconds...")
                        await asyncio.sleep(delay)

                if failed_links:
                    with open(failed_links_file, 'w', encoding='utf-8') as f:
                        json.dump(failed_links, f, indent=4, ensure_ascii=False)
                    print(f"{len(failed_links)} failed links saved to {failed_links_file}")

            except json.JSONDecodeError:
                print(f"Error: {os.path.basename(file_path)} is not a valid JSON file. Skipping.")
                continue
            except Exception as e:
                print(f"An unexpected error occurred while processing {file_path}: {e}")

    print("\n--- All done. ---")

# 运行正文下载
asyncio.run(main())

```

&emsp;

## 5. 目录结构与输出示例

以下是运行代码输出的内容目录结构：

```yaml
project_root/
├─ transcripts/                 # 步骤 A 产出（每个 ticker 一个链接清单）
│  ├─ AAPL.json
│  ├─ MSFT.json
│  └─ ...
├─ transcript/                         # 步骤 B 产出（每个 ticker 一个文件夹）
│  ├─ AAPL/
│  │  ├─ 2024 Q2 Apple Inc. Earnings Call Transcript.txt
│  │  ├─ ...
│  │  └─ failed.json                   # 下载失败的链接列表
│  ├─ MSFT/
│  └─ ...
├─ progress.txt                 # 已完成的 tickers
├─ exclude_transcript.txt       # 无 transcripts 的 tickers
└─ failed.txt                   # 收集链接阶段失败的 tickers

```

&emsp;

## 6. 常见问题

根据我的实际运行经验，以下是一些常见问题及建议：

- **端口统一**：`connect_over_cdp("http://localhost:9222")` 与你启动 Chrome 的端口一致；爬取链接和爬取正文文本的浏览器端口统一，不要 步骤 A 用 9333、步骤 B 用 9222 。
- **验证码**：该流程按“**人工验证**”设计；若验证码频繁，可以考虑减慢爬取频率、增加随机等待时间，或使用**多个随机代理**（需遵守网站条款）。
- **标题过滤**：`if 'Earnings' in title:` 控制只下载“Earnings”类文章。若要全部下载，删除这行条件。
- **文件名冲突**：`sanitize_filename` 已处理非法字符，但同名不同文可能会覆盖上一个文件，建议再加上**发布日期**进行区分。
- **存储格式**：目前正文是 以`.txt`的格式储存。若后续需要做 NLP 处理，推荐转为 **JSONL/Parquet**（含 `{ticker, title, url, date, text}` 等字段），便于后续 pandas/pyarrow/duckdb 读取与并行处理。

&emsp;
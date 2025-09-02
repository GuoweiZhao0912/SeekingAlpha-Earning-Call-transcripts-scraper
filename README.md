# SeekingAlpha-Earning-Call-transcripts-scraper
seeking alpha是最常用的查看公司earning call transcript之一，在做金融与文本分析交叉研究中经常用到，以下是我爬取文本的策略。
我的策略分为两步：
1. 首先按照所有公司的代码获取每个代码包含的所有文本的页面url。
2. 其次对所有的url进一步爬取每个url背后的文本
这样做的好处是：
1. 直接爬取所有文本，所能获取到的文本数量有上限，在无限的往下滑动页面时，并不能获取到所有的transcript，大概只能获取一年左右的数据。
2. 其次，先爬取链接进行保存，会让整个爬取的过程更加稳健。

## 获取所有公司代码
- 如何获取seekingalpha的所有transcript的公司代码。
点击进入网页：https://seekingalpha.com/author/sa-transcripts/analysis
点击公司代码搜索框，点击搜索出现刷新出现所有的公司代码，然后右键进入开发者模式。
选中其中一个代码后，在elements中找到html结构中的关于公司代码选项的内容，例如：

```html
<button class="relative inline-flex my-8 mr-14" type="button"><span class="Ly4_J inline-flex items-center rounded-4 RXlLr py-10 px-18 items-center text-x-large-r KQfXX" data-test-id="chip"><span class="vIR0R opacity-100 transition-opacity duration-200">AGYS (70)</span></span></button>
```

进入console，输入代码，回车，会得到所有的公司代码，进行复制，保存到txt文件进行后续处理：

```javascript
// 选择所有具有 data-test-id="chip" 的元素
const chipElements = document.querySelectorAll('[data-test-id="chip"]');

// 提取每个chip内部的文本内容
const chipTexts = Array.from(chipElements).map(chip => {
  return chip.textContent.trim();
});

// 用换行符连接所有文本，方便复制
console.log(chipTexts.join('\n'));

// 或者直接输出数组查看
console.log(chipTexts);
```

从代码中可以看到有一个很明显的属性 `data-test-id="chip"`，这通常是为测试准备的，非常稳定且唯一。

## 爬取链接
在爬取文本链接时，对链接 https://seekingalpha.com/symbol/{ticker}/earnings/transcripts 进行爬取，(注意并不是 https://seekingalpha.com/author/sa-transcripts/analysis?ticker={ticker} ，该链接并不能获取完整的文本链接)，详细见文件`fetch records by ticker_调试`

1. 由于反爬机制的存在（我目前还没解决），我们需要手动做人机验证，我在程序中加入了暂停程序-手动验证的机制
2. 由于种种原因，我们依然可能对某些代码爬取失败（比如网络），所以在爬取过程会对爬取成功和失败的代码进行保存，另外某些代码并不存在transcript，也进行了另外的保存。分别保存在progress.txt, failed.txt和failed.txt文件中

## 保存后的txt文件，进行后续处理。
我们得到的公司代码形式为：AA (175)、AAUC.DB.U:CA (3)、ACB.WS.U:CA (12)等较为复杂的形式，其中数字表示该代码的transcript篇数，现在要对txt文件进行文本处理并且得到结构化的公司代码数据。同时也方便我们爬取到文本后进行对比，是否爬取完整。

## 爬取文章
详细可见 `download transcript_调试.ipynb`，需要会员账号才能爬取。同样的加入了检测爬取失败并保存失败链接的机制，以及检测验证码-暂停程序-手动验证的机制
## tips
- 使用notebook爬取，需要解决异步问题
- 终端打开chrome，注意保证端口与爬取程序中一致，此处为9333
```
/Applications/Google\ Chrome.app/Contents/MacOS/Google\ Chrome --remote-debugging-port=9333 --user-data-dir=/tmp/chrome-profile
```
- 爬取完后，会出现很多无法爬取的链接，页面显示Whoops, we’ve hit a bottom.这类code都是XXX-DEFUNCT的形式，通常该代码已经失效或者已经退市，因此我们需要删除后缀重新爬取，处理过程可见`clean_mismatched.ipynb`

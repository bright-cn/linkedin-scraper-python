# amazon-scraper-node

[![运行状态检查](https://github.com/bright-cn/amazon-scraper-node/actions/workflows/live.yml/badge.svg)](https://github.com/bright-cn/amazon-scraper-node/actions/workflows/live.yml)
[![最近验证时间](https://img.shields.io/badge/last%20verified-6%20Oct%202026-brightgreen)](https://github.com/bright-cn/amazon-scraper-node/actions/workflows/live.yml) <!-- verified: rewritten by the daily run -->

[快速开始](#快速开始) · [命令行使用](#作为命令运行) · [API 接口](#其他-api-接口) · [数据](#数据) · [错误处理](#出错时) · [编程智能体](#编程智能体) · [文档](https://docs.brightdata.com/products/scrapers/amazon/introduction) · [支持](#支持)

使用 JavaScript 将亚马逊商品、评论、卖家和搜索结果获取为 JSON。无需登录，也无需浏览器。基于 [Bright Data Amazon 爬虫 API](https://www.bright.cn/products/web-scraper/amazon?utm_source=github) 构建。

本项目使用 [Bright Data JavaScript SDK](https://github.com/bright-cn/sdk-js)。完整 API 文档：[Amazon 爬虫 API](https://docs.brightdata.com/products/scrapers/amazon/introduction)。

本仓库还提供用于获取商品数据的单命令 CLI，以及完全无需编写 JavaScript 的 [Bright Data CLI](#编程智能体)。

## 快速开始

需要 Node 20 或更新版本。这个包使用 ESM，因此请使用 `import`，不要使用 `require`。

```bash
npm install @brightdata/sdk
export BRIGHTDATA_API_TOKEN=YOUR_API_KEY
```

从 [Bright Data 控制面板](https://www.bright.cn/cp/setting/users)获取令牌。此 SDK 不会自行读取 `.env` 文件；可以让 Node 通过 `node --env-file=.env yourscript.mjs` 加载。

也可以不手动设置令牌。先运行一次 `npx -p @brightdata/cli bdata login`：它会打开浏览器。此后，SDK 会自行找到已保存的凭据，供你以及在该终端工作的任何编程智能体使用。智能体无法自行完成浏览器中的登录操作，因此请先亲自登录。

还没有账户？[创建账户](https://www.bright.cn/cp/start)；新账户每月可获得 [5,000 免费积分](https://docs.brightdata.com/general/account/billing-and-pricing/free-tier)。

```javascript
import { bdclient } from "@brightdata/sdk";

const client = new bdclient({ autoCreateZones: false });
const job = await client.scrape.amazon.collectProducts(
  ["https://www.amazon.com/dp/B0CRMZHDG8"],
  { async: true, includeErrors: true },
);
const result = await job.toResult({ pollTimeout: 600_000 });
if (!result.success) throw new Error(`${result.status}: ${result.error}`);
const [product] = result.data;
console.log(product.title);
console.log(product.final_price, product.currency, "|", product.rating, "stars |", product.reviews_count, "reviews");
await client.close();
```

```text
STANLEY Quencher H2.0 Flow State Tumbler, 40 oz, Fuchsia
39.95 USD | 4.7 stars | 205036 reviews
```

每件商品消耗 [1 个积分](https://www.bright.cn/pricing/web-scraper)，API 预估每条输入约需 7 秒。

上面的代码片段中，以下设置都不可省略。

`autoCreateZones: false` 会阻止 SDK 在启动时创建区域。这些区域用于网络解锁器和搜索引擎 API，是此爬虫工具不会用到的另外两款 Bright Data 产品。如果账户未添加付款方式，创建区域会失败。

`includeErrors: true` 会让 API 把无效 ASIN 作为一行结果返回。不设置它，这一行会被丢弃，你将收不到对应结果。

`async: true` 会让 `collectProducts` 返回任务，而不是使用一分钟后就会放弃的同步接口。

`pollTimeout` 的单位是毫秒，不是秒。如果直接复制 Python 示例中的数字，可能还没进行第一次状态检查就已超时。

SDK 的商品过滤器接受 `url`、`zipcode` 和 `language`。API 也接受 `asin`、`origin_url` 和 `all_variations`，但该过滤器会拒绝这三个字段。因此，请像上面那样将 ASIN 放进 `/dp/` URL 中传入。

## 作为命令运行

此仓库中的命令可以一次处理多个 ASIN，并写入一个 JSON 文件。

```bash
npm install -g github:bright-cn/amazon-scraper-node
amazon-scraper B0CRMZHDG8 B085DVHQ57
```

```text
Fetching 2 Amazon products: B0CRMZHDG8, B085DVHQ57
One job for all of them. One credit per product.
asking  2 products...
got     B0CRMZHDG8: 99 fields (STANLEY Quencher H2.0 Flow State Tumbler, 40 oz,)
got     B085DVHQ57: 100 fields (Owala FreeSip Stainless Steel Water Bottle 32 oz)

Saved 2 of 2 products as JSON to amazon.json
```

命令既接受 ASIN，也接受商品 URL。因此，`amazon-scraper https://www.amazon.com/dp/B0CRMZHDG8` 的效果相同。重复输入的 ASIN 只会抓取一次，不会因重复而产生费用。

在终端中，`asking` 那一行会被下面的进度显示替换，并原地更新，让你知道任务仍在运行以及已经运行了多久：

```text
⠹ 2 products 0:00:11
```

```text
--out PATH   output file, default amazon.json
```

请求的所有 ASIN 都放在同一个任务中。API 按记录而不是按任务计费；十件商品也只需等待一个任务完成，而不是分别等待十个任务。

也可以导入它，而不是将它作为命令运行。这样，每个 ASIN 都会得到 `ok` 和 `error` 状态，而不只是原始数据行。单个 ASIN 失败不会使 `scrape` 拒绝整个请求；读取 `product` 前请先检查 `ok`：

```javascript
import { scrape } from "@brightdata/amazon-scraper-node";

for (const outcome of await scrape(["B0CRMZHDG8", "B0ZZZZZZZZ"])) {
  if (outcome.ok) {
    console.log(`${outcome.asin}: ${outcome.product.title}`);
  } else {
    console.log(`${outcome.asin} failed: ${outcome.error}`);
  }
}
```

```text
B0CRMZHDG8: STANLEY Quencher H2.0 Flow State Tumbler, 40 oz, Fuchsia
B0ZZZZZZZZ failed: The navigation resulted in a dead page (404 status code)
```

## 其他 API 接口

上面的命令对应下表第一行。这里的每段代码都是完整示例，只需要 `@brightdata/sdk`，可直接粘贴运行。所有示例每周一都会在 Actions 中运行，其他日子还会执行规模较小的检查。页面顶部的徽章显示最近一次结果。

| 已有信息 | 想获取 | 调用方式 |
| --- | --- | --- |
| ASIN 或商品 URL | 对应商品 | `collectProducts([url], { async: true, includeErrors: true })` |
| 关键词 | 匹配的商品 | `discoverProductsByKeyword([{ keyword }], { async: true, includeErrors: true, limitPerInput: 3 })` |
| 分类 URL | 分类中的商品 | `discoverProductsByCategoryURL([{ url }], …)` |
| 畅销商品榜单 URL | 榜单上的商品 | `discoverProductsByBestSellerURL([{ category_url }], …)` |
| UPC | 匹配的商品 | `discoverProductsByUPC([{ upc }], …)` |
| 卖家 URL | 对应卖家 | `collectSellers([url], { async: true, includeErrors: true })` |
| 关键词和店铺 URL | 搜索结果页面 | `collectProductSearch([{ keyword, url }], …)` |
| 商品 URL | 商品评论 | `collectReviews([url], …)`；每条评论消耗 1 个积分，详见下文 |

这些方法都位于 `client.scrape.amazon` 下。

### 此处不运行评论抓取，原因是费用

评论按条计费，每条消耗 1 个积分；目前无法通过此 SDK 可靠地限制返回条数。

API 接受 `max_reviews` 输入参数，Bright Data 自己的文档示例将它设为 20。但 SDK 的评论过滤器只声明了 `url` 和 `reviews_to_not_include`，并会拒绝 `max_reviews`。SDK 自带的限制参数 `limitPerInput` 在评论数据集上返回零行：这一结果已在两件商品上测试三次；同一参数在商品数据上则正常工作（[sdk-js#37](https://github.com/bright-cn/sdk-js/issues/37)）。

因此，目前唯一能返回评论的调用不设数量上限。本 README「快速开始」中的商品显示有 205,036 条评论，而这只需一次调用就会触发。

请先查看商品的 `reviews_count`。在 SDK 支持 `max_reviews` 之前，如果只需要限定数量的评论，请直接调用 REST 接口。

### 数量上限，以及会让上限悄然失效的参数

`limitPerInput` 用于限制发现类调用的返回数量。它只有与 `async: true` 一起传入时才有效：如果没有 `async`，选项会匹配 SDK 选项结构中的同步分支；该分支会丢弃不认识的字段，数量上限也会在不报错的情况下消失（[sdk-js#36](https://github.com/bright-cn/sdk-js/issues/36)）。SDK 的 `DiscoverOptions` 类型却提示你省略 `async`，因此符合其类型定义的调用反而没有数量上限。

这里的每个商品发现示例都会同时传入这两个参数。

### 按关键词发现商品

```javascript
import { bdclient } from "@brightdata/sdk";

const client = new bdclient({ autoCreateZones: false });
const job = await client.scrape.amazon.discoverProductsByKeyword(
  [{ keyword: "stainless steel water bottle" }],
  { async: true, includeErrors: true, limitPerInput: 3 },
);
const result = await job.toResult({ pollTimeout: 900_000 });
if (!result.success) throw new Error(`${result.status}: ${result.error}`);
for (const product of result.data) {
  console.log(product.asin, "|", product.final_price, product.currency, "|", product.title.slice(0, 40));
}
await client.close();
```

```text
B0D2W1MKZX | 14.24 USD | Fijinhom Insulated Water Bottle with Han
B085DVHQ57 | 29.99 USD | Owala FreeSip Stainless Steel Water Bott
B0D8J2ZB8P | 14.24 USD | POWCAN 26 oz Insulated Water Bottle with
```

如果不设置 `limitPerInput`，该调用会返回所有匹配项，每项消耗 1 个积分。

### 多件商品，一个任务

一个 URL 数组只会创建一个任务，不会为每件商品各建一个任务。

```javascript
import { bdclient } from "@brightdata/sdk";

const client = new bdclient({ autoCreateZones: false });
const job = await client.scrape.amazon.collectProducts(
  ["https://www.amazon.com/dp/B0CRMZHDG8", "https://www.amazon.com/dp/B085DVHQ57"],
  { async: true, includeErrors: true },
);
const result = await job.toResult({ pollTimeout: 600_000 });
if (!result.success) throw new Error(`${result.status}: ${result.error}`);
for (const product of result.data) {
  console.log(product.asin, "|", product.brand, "|", product.final_price, product.currency);
}
await client.close();
```

```text
B085DVHQ57 | Owala | 29.99 USD
B0CRMZHDG8 | STANLEY | 39.95 USD
```

返回结果的顺序不作保证。请用每行自身的 `asin` 匹配请求的商品，不要依赖行的位置。

### 现在触发，稍后获取

如果要处理的不止几件商品，不要让进程阻塞一小时。先触发任务并保存快照 ID，待任务就绪后再获取结果。快照可在 30 天内下载。

```javascript
import { bdclient } from "@brightdata/sdk";

const client = new bdclient({ autoCreateZones: false });
const job = await client.scrape.amazon.collectProducts(
  ["https://www.amazon.com/dp/B0CRMZHDG8"],
  { async: true, includeErrors: true },
);
console.log("snapshot:", job.snapshotId);
await job.wait({ pollInterval: 5_000, pollTimeout: 600_000 });
console.log("status:", await job.status());
const [record] = await job.fetch();
console.log("fetched:", record.asin, "|", record.title.slice(0, 40));
await client.close();
```

```text
snapshot: sd_mu9ycsw7zyq04t1uw
status: ready
fetched: B0CRMZHDG8 | STANLEY Quencher H2.0 Flow State Tumbler
```

## 数据

大多数人关心的商品字段：

```text
asin  title  brand  final_price  currency  rating  reviews_count  availability
```

代码没有硬编码字段列表。API 返回的所有字段都会进入 `result.data`，也会写入命令生成的文件。

API 在数据结构中以 `pii: true` 将其中 3 个字段标记为个人数据：`seller_name`、`zipcode` 和 `coupon`。

<!-- fields:start -->
<details>
<summary>全部 119 个字段及其类型和说明</summary>

以下内容每天都会通过 `client.datasets.amazonProducts.getMetadata()` 从数据集的数据结构重新生成，因此不会过时。每件商品只包含适用于它的字段：示例文件包含这 119 个字段中的 97 个，另有数据结构未列出的 `timestamp` 和 `input`。

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `title` | 文本 | 商品标题 |
| `seller_name` | 文本 | 个人数据。卖家名称 |
| `brand` | 文本 | 商品品牌 |
| `description` | 文本 | 商品简介 |
| `initial_price` | 价格 | 初始价格 |
| `currency` | 文本 | 商品价格所用货币 |
| `availability` | 文本 | 商品供货状态 |
| `reviews_count` | 数字 | 评论数量 |
| `categories` | 数组 | 商品分类 |
| `parent_asin` | 文本 | 商品的父 ASIN |
| `asin` | 文本 | 每件商品的唯一标识符 |
| `buybox_seller` | 文本 | 购买框中的卖家 |
| `number_of_sellers` | 数字 | 该商品的卖家数量 |
| `root_bs_rank` | 数字 | 商品在大类中的畅销商品排名 |
| `ISBN10` | 文本 | 图书的 ISBN-10 标识符 |
| `answered_questions` | 数字 | 已回答的问题数量 |
| `domain` | URL | 商品所在域名的 URL |
| `images_count` | 数字 | 图片数量 |
| `url` | URL | 直达商品的 URL |
| `video_count` | 数字 | 视频数量 |
| `image_url` | URL | 直达商品图片的 URL |
| `item_weight` | 文本 | 商品重量 |
| `rating` | 数字 | 商品评分 |
| `product_dimensions` | 文本 | 商品尺寸 |
| `seller_id` | 文本 | 每位卖家的唯一标识符 |
| `image` | URL | 直达商品图片的 URL |
| `date_first_available` | 文本 | 商品首次上架日期 |
| `discount` | 文本 | 商品折扣信息 |
| `model_number` | 文本 | 商品型号 |
| `manufacturer` | 文本 | 商品制造商 |
| `department` | 文本 | 商品所属部门 |
| `plus_content` | 布尔值 | 是否存在附加内容 |
| `upc` | 文本 | 通用商品代码 |
| `video` | 布尔值 | 是否包含视频 |
| `top_review` | 文本 | 商品的精选评论 |
| `final_price_high` | 价格 | 最终价格为区间时的最高值 |
| `final_price` | 价格 | 商品最终价格 |
| `variations` | 数组 | 同一商品不同变体的详情 |
| `delivery` | 数组 | 配送相关信息 |
| `features` | 数组 | 商品特性 |
| `format` | 数组 | 图书格式相关信息 |
| `buybox_prices` | 对象 | 商品价格详情 |
| `input_asin` | 文本 | 输入的 ASIN（目前未启用） |
| `ingredients` | 文本 | 商品成分，主要适用于食品 |
| `origin_url` | URL | 用于提取这条记录的来源页面 URL |
| `bought_past_month` | 数字 | 过去一个月的购买件数（以亚马逊页面显示为准） |
| `is_available` | 布尔值 | 商品是否仍有货 |
| `root_bs_category` | 文本 | 畅销商品根分类 |
| `bs_category` | 文本 | 畅销商品分类 |
| `bs_rank` | 数字 | 商品在特定分类中的畅销排名 |
| `badge` | 文本 | 商品徽章，例如“#1 Best Seller”或“Amazon’s Choice” |
| `subcategory_rank` | 数组 | 按子分类列出的畅销商品排名 |
| `amazon_choice` | 布尔值 | 商品是否获得 Amazon’s Choice 标识 |
| `images` | 数组 | 商品图片的 URL |
| `product_details` | 数组 | 完整商品详情 |
| `prices_breakdown` | 对象 | 标价、常规价格及促销状态的明细 |
| `country_of_origin` | 文本 | 商品原产国 |
| `from_the_brand` | 数组 | 页面展示的品牌宣传素材 |
| `product_description` | 数组 | 嵌入商品描述部分的媒体内容 |
| `seller_url` | URL | 卖家在亚马逊上的店铺或资料页 URL |
| `customer_says` | 文本 | `customer_says` |
| `sustainability_features` | 数组 | 可持续发展徽章或认证及其参考信息 |
| `climate_pledge_friendly` | 布尔值 | 商品是否显示 Climate Pledge Friendly 徽章 |
| `videos` | 数组 | 商品视频的 URL |
| `other_sellers_prices` | 数组 | 其他卖家对同一商品的报价 |
| `downloadable_videos` | 数组 | 媒体文件的直接 URL |
| `editorial_reviews` | 数组 | 图书的编辑评论 |
| `about_the_author` | 文本 | 作者简介 |
| `zipcode` | 文本 | 个人数据。用于预估配送和供货情况的邮编 |
| `coupon` | 文本 | 个人数据。优惠券 |
| `sponsered` | 布尔值 | 是否为赞助内容 |
| `store_url` | URL | 商品所属店铺的 URL |
| `ships_from` | 文本 | 商品发货地 |
| `city` | 文本 | 与配送、卖家或位置相关的城市 |
| `customers_say` | 对象 | 从评论中提取的亚马逊“顾客评价”摘要 |
| `max_quantity_available` | 数字 | 允许加入购物车的最大数量 |
| `variations_values` | 数组 | 商品变体及其可选值 |
| `language` | 文本 | 商品页面或内容的语言 |
| `return_policy` | 文本 | 商品页面展示的退货政策文本 |
| `inactive_buy_box` | 对象 | 购买框不可用或未启用时的价格信息 |
| `buybox_seller_rating` | 数字 | 购买框卖家的评分 |
| `premium_brand` | 布尔值 | 是否为高端品牌 |
| `amazon_prime` | 布尔值 | 是否提供 Amazon Prime 配送 |
| `coupon_description` | 文本 | 优惠券说明 |
| `all_badges` | 数组 | 所有徽章 |
| `sponsored` | 布尔值 | 亚马逊赞助内容标记 |
| `variant_id` | 文本 | 特定商品变体的唯一标识符 |
| `product_category` | 文本 | 用分隔符连接的完整面包屑分类路径 |
| `category_tree` | 数组 | 分类层级，以包含名称和 URL 的对象数组表示 |
| `availability_date` | 文本 | 缺货商品的预计到货日期 |
| `listing_has_variations` | 布尔值 | 商品页面是否包含多个变体 |
| `variant_attributes` | 数组 | 当前变体的属性，按名称和值成对表示 |
| `variants` | 数组 | 按类型分组的结构化变体选项 |
| `seller_privacy_policy` | 文本 | 卖家隐私政策的 URL |
| `seller_tos` | 文本 | 卖家服务条款的 URL |
| `return_window` | 数字 | 退货期限的天数 |
| `target_countries` | 数组 | 商品可配送到的国家 |
| `store_country` | 文本 | `store_country` |
| `category_urls` | 数组 | 分类面包屑导航 |
| `all_variations` | 布尔值 | 用于采集所有变体的输入字段 |
| `safety_information` | 文本 | 安全信息 |
| `subcategory_link` | 数组 | 按子分类列出的畅销商品链接 |
| `all_inactive_buy_box` | 数组 | 所有未启用购买框的信息 |
| `is_frequently_returned_item_badge` | 布尔值 | 是否显示“经常退货”徽章 |
| `frequently_returned_item_message` | 文本 | 警告框中显示的文本 |
| `is_customers_usually_keep` | 布尔值 | 顾客是否通常会保留此商品 |
| `title_badge` | 文本 | 徽章标题 |
| `review_images` | 数组 | 评论图片 |
| `review_videos` | 数组 | 评论视频 |
| `also_viewed` | 数组 | 顾客还浏览了的商品 |
| `similar_items` | 数组 | 可供考虑的类似商品 |
| `bought_past_month_text` | 文本 | 过去一个月的购买件数（按亚马逊展示的文本格式） |
| `is_high_price` | 布尔值 | 商品是否被标记为高价 |
| `title_highlight` | 文本 | 亚马逊在商品页面标题后直接显示的补充营销或亮点文本 |
| `title_clean` | 文本 | 卖家或品牌定义的实际商品标题，不包含亚马逊可能在标题旁或标题后展示的亮点文本 |
| `customers_say_topics` | 数组 | “顾客评价”部分的结构化主题明细 |
| `brand_url` | URL | 商品品牌 URL |
| `variant_condition` | 文本 | 商品变体的成色或状态，例如全新、翻新、翻新－优秀 |
| `is_aplus_premium` | 布尔值 | 商品是否使用高级 A+ 内容 |

</details>
<!-- fields:end -->

<details>
<summary>真实输出文件的开头，来自 <code>amazon-scraper B0CRMZHDG8</code></summary>

```json
{
  "generated_at": "2026-09-20T15:10:50.120Z",
  "products": [
    {
      "asin": "B0CRMZHDG8",
      "product": {
        "title": "STANLEY Quencher H2.0 Flow State Tumbler, 40 oz, Fuchsia",
        "seller_name": "Avrix Brands",
        "brand": "STANLEY",
        "description": "Constructed of recycled stainless steel for sustainable sipping, our 40 oz Quencher H2.0 offers maximum hydration with fewer refills. Commuting, studio workouts, day trips or your front porch—you’ll want this tumbler by your side. Thanks to Stanley’s vacuum insulation, your water will stay ice-cold, hour after hour. The advanced FlowState™ lid features a rotating cover with three positions: a straw opening designed to resist splashes while holding the reusable straw in place, a drink opening, and a full-cover top. The ergonomic handle includes comfort-grip inserts for easy carrying, and the narrow base fits just about any car cup holder.",
        "initial_price": 45,
        "currency": "USD",
        "availability": "In Stock",
        "reviews_count": 205036,
        "categories": [
          "Home & Kitchen",
          "Kitchen & Dining",
          "Storage & Organization",
          "Thermoses",
  ...
```

包含一件商品及其全部字段的完整文件见 [examples/sample_output.json](examples/sample_output.json)。

</details>

## 出错时

| 看到的信息 | 含义 |
| --- | --- |
| `API token required but not found.` | 在发送任何请求之前以状态码 2 退出。请设置令牌。 |
| `failed  ASIN: ...` | 以状态码 1 退出。商品不存在，通常是 ASIN 输入有误。 |
| `failed  ASIN: the API returned no row for this ASIN` | 以状态码 1 退出。任务结果中没有该输入对应的数据行。请重新运行。 |
| `failed  ASIN: Polling timed out after 605s for sd_...` | 以状态码 1 退出。请求在等待 600 秒后放弃。请重新运行。 |

任何失败都会以状态码 1 退出，因此脚本可以安全地依据运行结果决定是否继续。

在 SDK 中，相同情况表现如下：

| 看到的信息 | 含义 |
| --- | --- |
| `AuthenticationError: No API token found.` | 在选项、环境变量和 CLI 登录信息中都找不到令牌。 |
| 状态码为 401 的 `APIError` | 已设置令牌，但令牌不正确。 |
| `result.success` 为 `false`，`result.status` 为 `"timeout"` | SDK 等待超时。增大以毫秒为单位的 `pollTimeout`，或重新运行。 |
| `result.data` 中某一行包含 `error` 键 | 这是启用 `includeErrors` 后 API 对某条输入返回的结果；其他行不受影响。 |
| 调用成功但返回零行 | 如果使用的是评论数据集，原因可能是 `limitPerInput`。移除它之前，请留意可能产生的费用。 |

## 编程智能体

无需编写 JavaScript，也无需预先安装任何内容。粘贴以下两行即可；第一行会打开一次浏览器。如果通过 SSH 或在 CI 中使用，请改用 `bdata login --device`：

```bash
npx -p @brightdata/cli bdata login
npx -p @brightdata/cli bdata pipelines amazon_product "https://www.amazon.com/dp/B0CRMZHDG8"
```

`bdata pipelines list` 会列出所有类型。亚马逊相关类型包括 `amazon_product`、`amazon_product_reviews` 和 `amazon_product_search`。它们接受 URL、输出 JSON，并按每条记录 1 个积分计费。

运行 `npx skills add brightdata/skills`，可以让 Claude Code、Cursor 和 Codex 学会这些命令并了解相关文档，此后就能用自然语言提出需求。完整指南：[让编程智能体使用 Bright Data](https://docs.brightdata.com/quickstart-coding-agent)。

使用托管助手、没有终端？[Bright Data MCP 服务器](https://github.com/bright-cn/brightdata-mcp#which-tool-to-use)的 `ecommerce` 工具组中包含亚马逊工具；该工具组默认关闭，需要明确启用：

```text
https://mcp.brightdata.com/mcp?token=YOUR_API_TOKEN&groups=ecommerce
```

智能体还可以自行创建账户，无需填写注册表单：[智能体注册](https://www.bright.cn/auth.md)。如需了解 Bright Data 与 LangChain、Zapier、n8n 等工具的其他集成方式，请参阅[集成文档](https://docs.brightdata.com/integrations/introduction)。

## 支持

发现此仓库中的问题？请[提交 issue](https://github.com/bright-cn/amazon-scraper-node/issues)，并参照 [CONTRIBUTING.md](CONTRIBUTING.md) 说明提供必要信息。

有关 API、账户或积分的问题，请联系 [Bright Data 支持团队](https://brightdata.zendesk.com/hc/en-us/requests/new)。

## 许可证

MIT。

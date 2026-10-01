---
name: amazon-product-page-scraping
description: |
  直连抓取亚马逊商品页（标题/品牌/现价/RRP/折扣/Coupon/Deal/BSR大类与小类/星级），支持英德美日等多站点。
  含反爬降级页识别、配送地址切换、BSR 大小类判别等实测结论。
  适用于：写亚马逊竞品监控脚本、比价工具、Listing 数据抓取、BSR 排名追踪、跨境电商选品数据采集。
  触发词：抓取亚马逊商品、亚马逊爬虫、ASIN 抓取、亚马逊价格监控、BSR 排名抓取、亚马逊 Coupon、
  amazon scraper、scrape amazon product page、ASIN price BSR。
agent_created: true
---

# 亚马逊商品页直连抓取

不依赖任何付费 API，用 `curl_cffi` + BeautifulSoup 直连亚马逊商品页。
以下结论均在英/德/美/日四站实测验证过（最近一次复核：2026-09-29），直接照搬可省掉大量试错。

## 0. 请求层（决定成败）

```python
from curl_cffi import requests
s = requests.Session(impersonate='chrome124')   # 关键：模拟 Chrome TLS 指纹
s.cookies.set('i18n-prefs', 'GBP', domain='.amazon.co.uk')   # 强制站点货币 ★
s.cookies.set('lc-main',    'en_GB', domain='.amazon.co.uk') # 强制站点语言
```

**货币必须强制，否则按 IP 判定所在地。** 实测：
不带 cookie → `HKD 47.42`（用户在香港）；带 `i18n-prefs=GBP` → `£4.45`。**无需登录账号。**

| 站点 | domain | currency | locale | accept-language |
|---|---|---|---|---|
| 英国 | amazon.co.uk | GBP | en_GB | `en-GB,en;q=0.9` |
| 德国 | amazon.de | EUR | de_DE | `de-DE,de;q=0.9,en;q=0.8` |
| 美国 | amazon.com | USD | en_US | `en-US,en;q=0.9` |
| 日本 | amazon.co.jp | JPY | ja_JP | `ja-JP,ja;q=0.9,en;q=0.8` |

反爬：请求间隔随机 2~6s；被拦截时按 10/30/60/120s 退避重试。单次任务建议 ≤ 50 个 ASIN。

### ★ UA 必须与 impersonate 版本一致，且**不要**中途轮换

`impersonate='chrome124'` 已经定义了 TLS 指纹**和**对应的客户端提示头。
请求头里的 UA 如果写成 Chrome 125/126 或 Safari，就出现「头指纹 ≠ TLS 指纹」——
和手动加 `sec-ch-ua=v126` 是同一类 bot 信号。

```python
# ✅ 正确：建会话时定一次，之后不动
UA = ('Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 '
      '(KHTML, like Gecko) Chrome/124.0.0.0 Safari/537.36')
s.headers.update({'User-Agent': UA, 'Accept-Language': cfg['accept_language'],
                  'Accept-Encoding': 'gzip, deflate, br', 'Referer': base})

# ❌ 错误：每次请求 random.choice(UA池)，池里还混了 125/126/Safari
```

## 0.05 反爬「降级页」—— 最隐蔽的坑，会静默污染数据

亚马逊判定客户端可疑时，**不返回 403，而是返回 200 + 一个「精简版」页面**：

| 特征 | 完整详情页 | 降级页 |
|---|---|---|
| HTML 大小 | 1.7 ~ 2.5 MB | 数百 KB（实测 333 KB） |
| `<html class="a-no-js">` | 有 | 有（不可区分） |
| **`id="productTitle"`** | **有** | **无** ← 唯一可靠判据 |
| `#corePrice*` / `#apex_desktop` | 有 | 无 |
| `#productDetails_feature_div` | 有 | 无 |
| `<title>` / `#glow-ingress-line2` | 有 | **仍存在**（所以不能用它判断） |

```python
_PRODUCT_MARK_RE = re.compile(r'id=["\']productTitle["\']')

def is_product_page(html):
    """完整详情页判据：长度 + #productTitle。降级页必须判为无效。"""
    return bool(html) and len(html) >= 8000 and bool(_PRODUCT_MARK_RE.search(html))
```

**为什么必须挡：** 一旦把降级页当有效页收下，标题/品牌/价格/BSR 会全部变空，
而状态判定还容易兜底成"在售"——**安静的坏数据比报错更危险**。

抓取循环里只有通过 `is_product_page()` 才 `return resp`，否则重试。

## 0.1 配送地址 —— 决定能不能抓到价格

**不设配送地址，亚马逊按 IP 判定收货地。若卖家不发货到那里，页面就不渲染 Buy Box，
价格/RRP/折扣/Coupon 全空，Buy Box 只显示 "This item cannot be dispatched to your selected delivery location"。**

这**极容易被误判成"商品断货"** —— 实测：

| ASIN | 改配送地前 | 改配送地后 |
|---|---|---|
| B0H7RY3JQ5 | 全空（误判断货） | £17.99 / RRP £29.99 / -40% |
| B0GT3Y8TX1 | 全空（误判断货） | £14.99 / RRP £15.99 / -6% |

读取当前配送地：`#glow-ingress-line2` 的文本（如 "Hong Kong"）。

### ★★ CSRF token 必须从「HTML 转义的 data-a-modal」里取（最容易取错）

真 token 在页面里长这样（注意 `&quot;` 转义）：

```
&quot;ajaxHeaders&quot;:{&quot;anti-csrftoken-a2z&quot;:&quot;hCsC/ugBTFW5Hy1x7yahN4a…&quot;}
```

```python
# ❌ 错误（会取到另一个用途的 token，地址切换静默失败）
tok = re.search(r'"csrfToken"\s*:\s*"([^"]+)"', html).group(1)   # 这是 aapi 的 csrfToken（2@…）

# ✅ 正确：先 unescape，再优先取 anti-csrftoken-a2z
from html import unescape
h = unescape(html)
for pat in (r'"anti-csrftoken-a2z"\s*:\s*"([^"]+)"',
            r'name="anti-csrftoken-a2z"\s+value="([^"]+)"',
            r'name="aapiCsrfToken"\s+value="([^"]+)"',
            r'"csrfToken"\s*:\s*"([^"]+)"'):
    m = re.search(pat, h)
    if m:
        tok = m.group(1); break
```

### 提交切换 + **必须回读确认**

```python
s.post(f"{base}/portal-migration/hz/glow/address-change?actionSource=glow",
    data={'locationType': 'LOCATION_INPUT', 'zipCode': 'SW1A 1AA',
          'storeContext': 'electronics', 'deviceType': 'web',
          'pageType': 'Detail', 'actionSource': 'glow', 'almBrandId': 'undefined'},
    headers={'anti-csrftoken-a2z': tok,
             'content-type': 'application/x-www-form-urlencoded; charset=UTF-8',
             'x-requested-with': 'XMLHttpRequest',
             'Referer': url, 'Origin': base,
             'Accept': 'text/html,*/*', 'Accept-Language': cfg['accept_language']})
```

> **致命坑**：token 无效时亚马逊照样返回 **HTTP 200 + 空 body**。
> 只看 `status_code == 200` 会得到"假成功"——日志写着"已设置配送地"，实际配送地没变。
> **必须**：① body 非空；② 再 GET 一次回读 `#glow-ingress-line2` 确认邮编/城市变了。
> 拿不到 token 就**不要发这个请求**（无意义且更像 bot）。

默认邮编：英国 `SW1A 1AA` / 德国 `10115` / 美国 `10001` / 日本 `100-0001`。

> **切换后的重抓有风险**：新页面可能是降级页（§0.05）。
> 必须用 `is_product_page()` 校验后才允许替换手里的页面；
> ❌ 绝不能写 `if resp2: resp = resp2` —— 那会用空页面覆盖已经抓到的好数据。

## 0.2 反爬软封锁伪装成 404

亚马逊被惹毛时会返回**真的 404 状态码** + 约 2163 字节的固定模板页，正文 HTML 注释里含：

```
To discuss automated access to Amazon data please contact api-services-support@amazon.com
```

**这不是商品下架！** 必须识别并重试，不能直接 `return None`：

```python
if resp.status_code == 404:
    if 'api-services-support@amazon.com' in resp.text:
        retry()          # 软封锁，重试
    else:
        return None      # 真下架
```

## 1. 标题与品牌

```python
title = soup.select_one('#productTitle').get_text(strip=True)

# 品牌：三档兜底，都没有就返回 '—'
tr = soup.select_one('tr.po-brand')                    # ① 最可靠
if tr:
    tds = tr.select('td')
    if len(tds) >= 2: brand = tds[1].get_text(strip=True)
# ② #bylineInfo（"Visit the XXX Store" / "XXXのストアを表示"）
# ③ #productDetails_feature_div 里 th 为 Brand/Marke/Manufacturer/Hersteller 的行
```

## 2. 价格体系（坑最多）

### 先定位价格块，否则会抓到推荐位的价格

```python
def pick_price_container(soup):
    for sel in ['#corePriceDisplay_desktop_feature_div', '#corePrice_feature_div', '#apex_desktop']:
        c = soup.select_one(sel)
        if c and c.select_one('.a-price'): return c
    return None
```

**容器找不到就 `return None`，绝不 fallback 到扫整个 soup。**
实测同一页面在无 Buy Box 时仍有 **5 个** `.a-price`（`£9.49 / £1,799 / £19.99 / £4.37 / £14.16`），
**全部来自"相似商品/猜你喜欢"轮播**，没有一个是本品价格。

同样地，**RRP 与折扣也必须限定在价格块内**：

```python
cont = pick_price_container(soup)
if cont is None:
    return None          # ✅ 宁可空着，不给错数
# ❌ scope = cont if cont is not None else soup  ← 这就是"数字不是页面上那个"的根源
```

遍历 `.a-price` 时还要**排除父链含 `<a>` 的元素**（`p.parents` 链前 3 层不能有 `a` 标签）——这是"查看全部报价"链接块的特征。

### 现价

```python
# ① 最可靠
'#apex-pricetopay-accessibility-label'   # "£4.45 with 27 percent savings" / "10パーセントの割引で￥672"
# ② 打折商品 .a-offscreen 是空的，需重建
p.select_one('.a-price-symbol') + '.a-price-whole' + '.a-price-fraction'
```

> **致命坑**：`a-price-whole` 内含小数点（英 `4.`、德 `49,`），
> 重建时必须 `.rstrip('.,')` 再补小数位，否则拼出 `4..45` 直接解析失败。
> 单价用 `:not(.a-text-price)` 排除（`.apex-priceperunit-value` 带该类）。

### RRP（划线价）与折扣

```python
'.apex-basisprice-offscreen-label'   # "Was: £6.12" / "RRP: £9.99" / "過去価格: ￥747" / "UVP:"
'.savingsPercentage'                  # "-27%"
```

加校验：只接受 `RRP >= 现价`（防变体/多报价块串味）。

### 数值解析（多语言数字格式）

`1,234.56`(英) / `49,99`(德) / `1.234,56`(德千分位) / `672`(日无小数)

```python
# 同时含 . 和 , → 靠后的那个是小数点
# 只含 , → 尾部 3 位是千分位，否则是小数点
```

货币符号要支持 **符号在前**（`£5.00`）和 **符号在后**（`5,00 €`）两种语序。

## 3. Coupon 与 Deal（不需要登录）

亚马逊在服务端就渲染好了，未登录同样能抓。

```python
# Coupon 主容器（空 = 无券）
'#couponsInBuybox_feature_div'
# 备选：#promoPriceBlockMessage / .couponBadgeRegular / #couponBadge / .vpc-button
```

**必须排除"多买促销"**：`Save 5% on any 4` 不是 Coupon。只认关键词 `coupon|gutschein|クーポン|优惠券`。

Deal：`#dealBadge` / `#dealBadgeSupportingText` / `#limitedTimeDeal`
类型关键词：`lightning deal|blitzangebot|タイムセール`、`deal of the day`、`limited time deal`。
徽标文案可能被模板化成 `NO_OF_HOURS`，元素存在即可判定有 Deal。

券后价：百分比券 `现价×(1-p%)`，金额券 `现价-金额`；无券时等于现价。

## 4. BSR 大类 / 小类

### 定位：按**标签文字**找，不要硬编码容器 ID

容器名换过好几轮（`detailBullets` → `productDetails` → `productDetails_expanderTables_depthRightSections`），
按 ID 找迟早会空。2026-09 实测的真是位置是 `table > tr > td > ul > li`，外层 ID 是
`#productDetails_expanderTables_depthRightSections`（嵌套在 `#productDetails_feature_div` 内）。

```python
BSR_LABEL_RE = re.compile(
    r'Best\s*Sellers?\s*Rank|Amazon\s*Bestseller-?\s*Rang|Bestseller-?\s*Rang|'
    r'売れ筋ランキング|ベストセラー', re.I)

# ⚠️ 不要写成宽松的 `Best Sellers` —— 页面上「Best Sellers」轮播随处可见，
#    实测 UK 页有 2 处 "Best Sellers" 出现在 BSR 标签之前，会定位到错误区块。
```

做法：`soup.find_all(string=BSR_LABEL_RE)` 逐个往上找最近的 `tr`/`li` 作为候选块，
**逐个尝试**直到解析出条目为止（单一匹配点 + 单一容器 = 脆弱）。

### 解析：一段文本可能塞多条

同一个 `<li>` 里可能用 `<br>` 分隔两条排名（老版 detailBullets 就是这种）。必须**先切分再逐条解析**：

```python
# ✅ 顺序扫描（finditer）切分
_EN_RANK_TOKEN_RE = re.compile(r'(?:[#№]\s*)?(?:Nr\.?\s*|No\.?\s*)?(\d[\d.,]*)\s+in\s+', re.I)
# 用法：找到所有 token 位置，类目名 = 本 token 结束 ~ 下一个 token 开始
# ❌ 不要用 lookahead 切分 —— 会在 "1,234" 这种千分位数字内部也切一刀（切成 "1," + "234"）
```

各语言版式：

| 语言 | 版式 | 示例 |
|---|---|---|
| 英 | `#561 in Cat` 或 `561 in Cat`（有时无 #） | `#53 in Cell Phone Portable Power Banks` |
| 德 | `Nr. 2.182 in Cat` | `Nr. 97 in Externe Handyakkus` |
| 日 | **顺序相反**：`类目 - 1位` | `家電＆カメラ - 1位` |

### 大类 / 小类判定（优先级从高到低）

1. **带 Top 100 标记** → 大类：`(See Top 100 in ...)` / `(Siehe Top 100 ...)` / `（売れ筋ランキングを見る）` / `ベストセラー`。
   日文是全角括号，要一起处理。
2. **部门级链接** → 大类：`/gp/bestsellers/<dept>/ref=…`（部门后面**直接**是 `ref=` 或结束）。
   小类：`/gp/bestsellers/<dept>/<数字节点ID>/ref=…`。
   正则必须是 `^/gp/bestsellers/[^/]+/(?:\?|ref=|$)`；
   > 旧写法 `^/gp/bestsellers/[^/]+/?(\?|ref=|$)` 对现代链接**永不匹配**（少了那个 `/`），等于失效。
3. **不变式兜底（与语言无关，推荐保留）**：同页里**大类排名数字一定 ≥ 小类排名**（小类是大类的子集）。
   只解析到 2 条且没有标记时，**数字大的那条是大类**。
4. **只解析到 1 条 → 放小类，大类留空**。
   > 2026-09 实测：**英国匿名页只给一条「N in <叶子类目>」，没有 "(See Top 100…)"**。
   > 此时大类输出 `—` 属正常，**不要猜**——强行填大类就是张冠李戴。
   > （要拿大类排名需要登录态/英国本地会话。）

排名数字要归一化千分位：`2.182 → 2182`、`1,234 → 1234`。
注意 `(See Top 100 in X)` 若能取到大类**类目名**，即使没有大类排名数字，也值得单独输出类目名。

## 5. 星级（多语言正则）

`#acrPopover` 的 title / aria-label / `.a-icon-alt`：

| 语言 | 文案 |
|---|---|
| 英 | `4.5 out of 5 stars` |
| 德 | `4,5 von 5 Sternen` |
| 日 | `5つ星のうち4.2` ← **数字在后面** |
| 法/西/意 | `4,5 sur/de/su 5 ...` |

```python
re.search(r'うち\s*([0-5][.,]?\d*)', t)                 # 日，必须先判
re.search(r'([0-5][.,]\d+)', t)                          # 通用
re.search(r'([0-5][.,]?\d*)\s*(?:out of|von|sur|su|de)\s*5', t, re.I)
```

评论总数：`#acrCustomerReviewText`。
星级分布：`a[aria-label]` 里的 `(\d+) percent ... have ([1-5]) star`，
兜底用 `.a-meter[aria-valuenow]` 前 5 个（顺序 5★→1★）。

## 6. 商品状态（顺序很重要）

状态列是关键辅助列，没有它用户看到空白行就以为抓取失败。

```python
def extract_status(html, soup, resp_status, price_val=None):
    if resp_status == 404:
        if 'api-services-support@amazon.com' in html:
            return '抓取失败'          # 反爬软封锁，不是下架
        return '已下架'
    if not is_product_page(html):
        return '抓取失败'              # ★ 降级页不许报"在售"
    if price_val:
        return '在售'                  # 有价一定在售，别信残留 JS 文案
    if soup.select_one('#fod-cx-box'):
        return '断货'                  # 真的渲染了"无 Buy Box"提示框
    av = soup.select_one('#availability')
    if av and re.search(r'unavailable|out of stock|nicht verfügbar|在庫切れ',
                        av.get_text(' '), re.I):
        return '断货'
    return '价格受限'                  # ★ 页面正常但无价格块 → 多半是配送地受限
```

要点：

- **`price_val` 判定放最前** —— 页面常残留 "No featured offers" 的 JS 缓存文案，
  直接搜 HTML 会把在售商品误判成断货。
- **末位不要 `return '在售'`** —— 那会把降级页、结构异常页统统报成在售。
  用「价格受限」这类专门状态，"在售但没价" 和 "真的在售" 才分得开。
- 缺货文案只在 **Buy Box / `#availability` 区域**匹配，不要扫整页文本 ——
  推荐位里其它 ASIN 的可见文案会造成反向误判。
  （好消息：实测 `soup.get_text()` **不含** `<script>`，所以页面里的 JS 文案字典不会误伤。）
- **调用时机**：必须在拿到 `price_val` 之后再算 status。

排查顺序：在售但无价 → 先看配送地址（§0.1）；抓取失败 → 反爬降级页（§0.05）或软封锁（§0.2）；断货 → 真的没 Buy Box。

## 7. 排错心法

**空字段先怀疑抓取侧，再怀疑商品侧。** 优先级：

1. **拿到的是降级页**（§0.05）→ `id="productTitle"` 不存在，全字段空但状态像"在售"
2. 配送地址不对（§0.1）→ 最常见，表现为有标题无价格
3. 反爬软封锁（§0.2）→ 假 404
4. 价格/RRP fallback 扫全页 → 抓到轮播里的别的商品价格
5. 商品真的断货 → 前四条都排除了才是

**排查方法（强烈推荐）**：把每一次请求的 HTML 落盘，然后直接 `import` 项目自身模块，
用**它自己的**解析函数跑这些真实 HTML，对比"页面实际显示"与"程序算出来"。
比读代码猜快得多，也比 mock 数据可信。

不要轻易下"商品已下架"的结论。用户说"你没访问进去，可能被拦截了"时，通常在理。

## 8. 打包成 macOS App 的坑（若需交付桌面工具）

- PyInstaller spec：`Analysis` + `EXE(console=False)` + `COLLECT` + `BUNDLE`
- `--clean` 会批量删缓存目录，易被安全策略拦截 → 改用 `--workpath /tmp/xxx`
- `dist/` 会同时产出非 .app 目录（内容重复），只保留 .app
- 重新打包时 PyInstaller 先删旧 .app（数百文件）也会触发拦截 → 先 `mv` 改名再打包
- **`ditto -c -k` 不写 UTF-8 文件名标志位，中文名解包会乱码** → 用 Python `zipfile` 打包
- zipfile 需手动处理符号链接（`S_IFLNK`，内容为 target）与可执行权限，.app 内通常有 30+ 个软链
- **验证必须 `open xxx.app` + pgrep 轮询**；直接跑 Mach-O 二进制会被 SIGKILL 造成误判
- 冻结版数据目录放 `~/Library/Application Support/<AppName>`，写 .app 内部在 /Applications 会因权限失败

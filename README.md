# amazon-scraper-skill

> Agent skill for scraping Amazon product pages without paid APIs — battle-tested selectors and anti-bot findings across UK/DE/US/JP: BSR category vs subcategory, multilingual star ratings, degraded-page detection, Buy Box & delivery-location gating, soft-block 404s.

亚马逊商品页直连抓取 Agent Skill：不依赖付费 API，沉淀英/德/美/日四站实测结论 —— BSR 大类/小类判别、星级多语言解析、反爬降级页识别、Buy Box 与配送地陷阱、软封锁伪装 404。

## 内容

| 章节 | 内容 |
|---|---|
| §0 | 请求层：`curl_cffi` impersonate、强制站点货币、UA 与 TLS 指纹一致性 |
| §0.05 | 反爬「降级页」识别（返回 200 的精简页，判据 `productTitle`） |
| §0.1 | 配送地址切换：CSRF token 取法、回读确认、默认邮编 |
| §0.2 | 软封锁伪装成 404（正文含 `api-services-support@amazon.com`） |
| §1 | 标题与品牌（三档兜底） |
| §2 | 价格体系：价格容器定位、现价、RRP、折扣、多语言数字格式 |
| §3 | Coupon 与 Deal（无需登录，排除「多买促销」） |
| §4 | BSR 大类 / 小类：语义化定位、多语言版式、四级优先级判定 |
| §5 | 星级与评论数（日站为倒序格式） |
| §6 | 商品状态判定顺序（断货 / 价格受限 / 抓取失败 / 已下架） |
| §7 | 排错心法：空字段先怀疑抓取侧，再怀疑商品侧 |
| §8 | 打包成 macOS App 的坑（PyInstaller / DMG / zip） |

## 用法

把 `SKILL.md` 放到你的 Agent 技能目录（如 `~/.workbuddy/skills/` 或对应平台的 skills 目录）即可被自动触发。

## 复核时间

英 / 德 / 美 / 日四站实测，最近一次复核 2026-09-29。

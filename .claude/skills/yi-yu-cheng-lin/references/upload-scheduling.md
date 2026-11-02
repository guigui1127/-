# 上傳與排程

主平台：Instagram Reels、Instagram Carousel、Threads。輔助：Skool（手動）、Email（Kit）、YouTube（手動上傳，文案由裂變設計師備好）。

## 鐵律

1. 一律先排草稿或先給清單過目，使用者說「確認」才真的排程，絕不跳過人審
2. 排程失敗要照實回報哪幾篇沒排上，不准含糊帶過
3. 所有貼文的 CTA 終點必須一致（品牌檔案裡的唯一終點），排程前逐篇檢查一次

## 頻率原則（30 天鋪法）

| 平台 | 頻率 | 備註 |
|---|---|---|
| IG 限動 | 每天 1 組（3 到 7 幀） | 14 天序列制，一組一次發完讓它過期，細節見 `references/story-funnel.md` |
| Reels（口播＋字卡＋精華） | 每週 4 到 5 支 | 三種型式輪替 |
| Carousel | 每週 1 到 2 篇 | 教學型為主 |
| Threads | 每天 1 到 2 則 | 短脆為主，長脆每週 1 到 2 則 |
| Skool 長文 | 每週 1 篇 | 固定日發 |
| Email | 每週 1 到 2 封 | 固定日發 |
| YouTube | 每月 1 到 2 支 | 母內容排最前面 |

排列原則：

- 母內容先發，短內容當根系往外鋪
- 同一痛點分散開，不連續兩天打同一個痛
- 三個走向每週都要出現，權威型放在 CTA 高峰日前後

## 排程工具優先序

排程工具（如 Blotato 或 Metricool，任一有連接就用）：

### 若連接 Blotato MCP

1. `blotato_list_accounts` 確認已連的 IG 與 Threads 帳號
2. 逐篇 `blotato_create_post`：文字類（Threads）直接排，影片與輪播帶 media URL（先用 `blotato_create_presigned_upload_url` 傳素材）
3. IG 限動：target 帶 `mediaType: "story"`，一幀一則，同一天的一組排同一時段連發（限動每則只吃一個素材）
4. 排完 `blotato_get_post_status` 逐篇核對，回報實際排上數量
5. 注意：Blotato 留言 API 只支援 IG／FB，Threads 留言不歸這裡管

### 若連接 Metricool MCP

1. `getBrandSettings` 確認品牌與平台
2. `getBestTimeToPostByNetwork` 拿最佳時段套進日曆
3. 逐篇 `createScheduledPost`（或 `createScheduledPostForReview` 走審核流）
4. `getScheduledPosts` 核對

### 都沒有連接時

輸出 Metricool 可匯入 CSV（欄位：Text, Date, Time, Network, Media URL）＋人工發文 checklist

## 日曆檔案

排程前先落地一份 `內容日曆-主題-月份.md`：

| 日期 | 平台 | 型式 | 編號 | 標題或 hook | CTA | 狀態 |
|---|---|---|---|---|---|---|

狀態：待錄製／待生成／待設計／可直接發／已排程／已發布

## 每週回收（排完不是結束）

1. 看數據：完播、存檔分享、CTA 行動（Blotato `blotato_get_post_analytics` 或 Metricool analytics）
2. 收語言：留言私訊裡受眾的原話，存進素材地圖
3. 餵回去：表現好的角度下輪加倍，新問題變成下一顆種子

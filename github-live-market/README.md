# 浩煒的 BEYBLADE X 共用線上行情

這份部署包改為 GitHub Pages + GitHub Actions。網頁不再內建 59,877 筆歷史售價，也不讀訪客 localStorage 的成交快照。所有訪客開啟網站時，從同站 snapshot.json 讀取共用資料。

**完成的是定時更新與共用資料的程式；四個 Facebook 社團的完整成交來源仍未接通。僅部署，不會自行產生已核實的新成交。**

## 更新方式

- GitHub Actions：每小時第 17 分鐘執行一次（UTC；台灣也是每小時第 17 分）。可在 Actions 手動執行。
- 每次執行：讀取已設定的成交 feed；若有 Firecrawl API key，再搜尋四個社團的公開節錄；保存共用資料與檢查狀態，重新部署 Pages。
- 網頁：開啟立即讀線上資料，停留時每 5 分鐘重讀，回到分頁或恢復連線也會檢查。
- GitHub 排程可能延遲；公開 repo 超過 60 天無活動時可能停用排程，需在 Actions 重新啟用。
- 預設查近 30 天、成交記錄。近 7／14／30 天不會偷偷回退成歷史全部資料。
- 更新失敗保留舊記錄，但上次成功同步時間不會假裝變成新的。

## Windows 部署步骤

1. 解壓縮 ZIP，開啟 github-live-market 資料夾。
2. 建立 GitHub repository，預設分支使用 main。將資料夾裡的 index.html、data、scripts、tests、.github、README.md 上傳到 repo 根目錄；不要多包一層 github-live-market。
3. 確認 repo 中存在 `.github/workflows/update-and-deploy.yml`。若網頁上傳漏了 .github，使用 Add file → Create new file，以這個完整路徑建立檔案，再貼入 ZIP 中同名檔內容。
4. Settings → Pages → Build and deployment → Source，選 **GitHub Actions**。
5. Settings → Actions → General → Workflow permissions，選 **Read and write permissions**，Save。若 repo 規則禁止 bot 直接寫入 main，需允許這個更新流程或調整保存策略。
6. Settings → Secrets and variables → Actions → New repository secret，依下表設定。
7. Actions → Update market and deploy Pages → Run workflow → Run workflow。
8. 執行完成後，開啟 Settings → Pages 顯示的網站網址。
9. 網站展開「線上資料狀態」，確認「資料服務上次檢查」及「成交來源上次成功同步」；有搜尋節錄但沒有成交來源時，網頁會明確顯示尚未接通。

| Secret | 用途 | 是否必須 |
| --- | --- | --- |
| FIRECRAWL_API_KEY | Firecrawl 公開搜尋 API；只蒐集社團搜尋索引節錄 | 要定時更新搜尋參考時必須 |
| MARKET_FEED_URLS | 提供原文與成交欄位的 HTTPS JSON feed；多個網址各占一行 | 要自動更新近期成交行情時必須 |

ChatGPT 已連接 Firecrawl，不代表 GitHub 已取得 API key。請把你自己的 key 直接存到 GitHub Secrets，不要貼到聊天、HTML 或公開 repo。搜尋會消耗你 Firecrawl 帳戶額度；預設每次執行 4 次搜尋。如要降低頻率，可將 workflow 的 cron 改成 `17 */6 * * *`（每六小時）。

## 成交資料來源：還需要什麼

目前可取得搜尋節錄；嘗試讀取 Facebook 原文時，服務明確回覆不支援該站。因此沒有承諾自動完整抓取社團成交文，也不以「今天蒐集」當成「今天成交」。

可以接入：你已授權、能穩定輸出原文及成交欄位的資料服務；或由賣家／社團管理員確認後提供的 JSON feed。四個指定社團以外的貼文不會接受。

`MARKET_FEED_URLS` 需要是真正提供下列格式的資料網址，**不是 Facebook 社團首頁、也不是 GitHub Pages 網站首頁**。上游需持续新增或修正資料，GitHub 排程才會取得新成交。

沒有這個 feed，Firecrawl 只會讓下方的「公開貼文參考」更新，上方近期成交仍然會空白。這是目前仍待接通的關鍵。

## Feed 欄位

最外層 JSON 為 `{"posts": [...]}`。每篇原文包括：

| 欄位 | 內容 |
| --- | --- |
| post_url | 四個指定社團 `/groups/社團ID/posts/原文ID/` 的 HTTPS 原貼文網址 |
| group_name | 社團名稱 |
| published_at | 真實原文發文時間，ISO8601 含秒與時區 |
| capture_method | `original_post` |
| excerpt | 實際原文節錄 |
| transaction_evidence | 可選的其他佐證文字 |
| listings | 該貼文的完整商品列表；同一原文的新版本會替換先前列表 |

每件商品包括 `model`（data/models.json 中的完整型號）、`price_twd`（新台幣整數）、`status`（成交／在售／收購／競標）、`kind`（整顆／零件／組合／拆賣／頂重）、`condition`（沒寫／全新／拆檢／二手）、`version`（留空或日版／台版／亞版／美版／韓版／港版／泰版）。

**成交必須再提供以下欄位，否則整個來源批次會被拒收，保留先前資料：**

- `sold_at`：實際成交時間，ISO8601 含秒與時區。不得早於發文或晚於目前時間。
- `price_basis`：固定為 `seller_confirmed_final`。
- `evidence_excerpt`：賣家確認最終成交價的原文節錄；僅有「已售出」及原開價，不符合這個要求。

零件單賣再填 `part_category`、`part_id`、`part_name`。整批合售不拆成單顆價；不同版本、零件、頂重保留分類。

來源提供的聲明仍需信任與核驗；程式只能檢查格式、一致性與日期，不會神奇證明交易真實。

## 驗證與操作

- `python -m unittest discover -s tests -v`：執行成交欄位、防重、失敗保留測試。
- `python scripts/update_market.py`：在有相同環境變數的環境更新 data/snapshot.json。
- Actions 執行紀錄會显示來源成功數與狀態。拒收資料的來源不會把舊紀錄清空。
- 搜尋結果一律視為未分類參考，沒有精確成交日期，不能進成交統計；內容可能含舊文或相鄰貼文，需開原文核對。
- `.github`、scripts、tests 不會作為 Pages 公開網站檔案；Pages 部署只包含 index.html、snapshot.json。

參考官方文件：
- https://docs.github.com/en/pages/getting-started-with-github-pages/using-custom-workflows-with-github-pages
- https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows#schedule
- https://docs.firecrawl.dev/api-reference/endpoint/search

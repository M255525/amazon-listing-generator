# CLAUDE.md — amazon-listing-generator（亞馬遜5點式產品說明產生器）

本檔案為 Claude Code 在此子資料夾工作時的指引。此資料夾**本身是獨立 git 儲存庫**，不受根目錄工作區規則約束（除語言等全域偏好）。

## 這是什麼

單檔前端工具：輸入商品名稱＋商品資訊（可貼上或上傳 .txt/.md 文字檔），產出符合 Amazon「商品標題」與「關於本商品」5點式說明規範的中英對照文案。是工作區 generator 家族最新一支，領域上最接近 `product-title-generator`（跨境商品標題產生器），但那個工具只產標題、這個工具產標題＋完整5點說明。

## 架構

單一 `index.html`：內嵌 CSS/JS、無外部資源、無建置步驟。視覺主題深色底＋琥珀金 `--accent:#f59e0b`（呼應 Amazon 品牌色，且與姊妹專案色票 blue/indigo/cyan/purple/magenta/green/mint/orange 皆不重複）。四個獨立 IIFE：跑馬燈／PWA安裝／序號授權閘門／主程式，互不相依。

### 商品資訊輸入

`productName`（文字）＋`productInfo`（textarea，可貼上）。「上傳文字檔帶入」按鈕用 `FileReader.readAsText()` 讀 `.txt`/`.md`，內容直接**取代**（非附加）textarea 內容——刻意不做複雜檔案解析，維持單一文字來源。5 組內建虛構商品範例（個人清潔用品／戶外露營／兒童家具／寵物用品／3C 配件）。`DRAFT_KEY='amzListingDraft'` 存 `{productName, productInfo}` 到 localStorage。

### 產生邏輯（雙軌）

- **規則式離線 fallback（零金鑰，`ruleBasedGenerate()`）**：`splitSentences()` 把商品資訊依換行與**全形句讀**（。！？；）切句，取前5句包成「【特色N】原文」／`(FEATURE N) 原文　[未經AI翻譯...]` 格式。**刻意不切半形句點/驚嘆號**——原本第一版用 `/\r?\n|(?<=[。.!！])/` 切句，會把 `$19.99`、`www.example.com` 這類含半形句點的內容從中間切斷成殘片（已用瀏覽器實測發現並修正，見 `splitSentences()` 的註解）。這條路徑**不做任何翻譯**，純粹是格式預覽。
- **AI 路徑（主要，BYOK，`buildPrompt()`+`callLLM()`+`extractJsonObject()`+`validateAiResult()`）**：與 `new-product-strategy-studio`／`business-idea-generator` 同一套 `AI_PROVIDERS`/`callLLM()` 實作（Claude 需 `anthropic-dangerous-direct-browser-access` header；429/500/503/529 重試3次；180秒逾時）。Prompt 要求回傳 `{titleEn,titleZh,bullets:[{en,zh}×5]}` 的 JSON，`validateAiResult()` 逐欄位（標題英/中＋5點各自的英/中，共12個欄位）驗證，缺漏個別退回規則式結果的對應欄位，不整批放棄，`textSource:'ai'|'mixed'|'rule'`。
- **Prompt 明確要求「基於事實、禁止捏造」（2026-08-31 應使用者要求新增，規則0）**：使用者填的「商品資訊」是賣家本人親自提供的真實內容，prompt 明示 AI 只能依這些事實撰寫，不可捏造/誇大商品資訊裡沒提到的功能/規格/材質/認證，任務是把既有事實用有吸引力、聚焦使用者效益的行銷語言表達，而非新增賣點；商品資訊不足以自然涵蓋5點時，要求從同一事實延伸不同敘述角度（例如同一功能分別談「解決什麼問題」與「使用情境」），而不是編造新內容。manual.html 的「關於規則式與AI優化」卡片已補一段 tip 說明此設計。

### Amazon 合規檢查（規則式，兩層防護的第一層）

`checkCompliance(text)`／`COMPLIANCE_RULES` 用 regex 偵測：價格符號、URL/email/電話、主觀宣傳詞（免運/限時/特價/保證/最好/第一名等）、HTML標籤，加上字數（`LEN_LIMIT=200`）警告。套用在**每一則標題與每一點內容**（不論規則式或AI產出）上，標紅/標黃顯示但不強制刪除——AI prompt 另外要求模型自行規避這些規則（第二層防護）。這只是常見類型的粗略掃描，不等於 Amazon 官方審核，manual.html／頁尾警語皆已明確揭露。

### 匯出

複製全部文字／下載 TXT／下載 CSV（`csvCell()`+UTF-8 BOM，逐字複製自 `product-title-generator` 的寫法）。

### 練習證明下載（2026-09-21 新增，含真實個資但刻意公開部署）

商品資訊區塊下方新增 4 個選填欄位——姓名／學號／系所／組別（`studentName`/`studentId`/`studentDept`/`studentGroup`），併入既有 `DRAFT_KEY='amzListingDraft'` 一起存 localStorage（非獨立 key）。產生完成後，匯出區塊新增「🎓 下載練習證明（PDF）」按鈕：`buildProofHtml()` 把身分資料＋商品名稱＋產出日期＋標題（中英）＋5點說明組成一段 HTML，寫入 `#printReportRoot`（隱藏 div，比照 `new-product-strategy-studio` 的 `#printReportRoot`＋`@media print{body>*{display:none!important}}` 手法）後呼叫 `window.print()`，使用者在列印視窗選「另存為 PDF」即可下載存證文件。**這是工作區少數刻意在公開部署（GitHub Pages）的工具裡收集真實姓名/學號的例外**——比照 `crispe-game`／`costar-game`／`amazon-logistics-game` 等遊戲類工具本機排行榜「填姓名/學號記錄成績」的既有慣例，資料只存使用者自己瀏覽器 localStorage、PDF 產生過程完全本機（`window.print()`），不經任何伺服器，因此與工具本身「不可輸入真實個資」的一般警語（商品名稱/商品資訊欄位）並不衝突——footer 警語已改成分別針對兩類欄位的措辭。未加簡易 PDF 浮水印（`new-product-strategy-studio`／`restaurant-feasibility-calculator` 那套 base64 圖片浮水印機制未套用於本次新增，因為證明文件用途不同、非商業文案輸出，且使用者未要求）。

## 序號授權（鎖定整個工具，12 個月）

比照 `new-product-strategy-studio` 的「單一工具、整個鎖住」模式：`#licenseGate` 全螢幕遮罩預設鎖定，驗證通過才加上 `.hidden`；載入時一律對後端即時重驗，背景每 20 分鐘重驗一次。`localStorage` key：`alGenSerial`。

- **綁定的 Google Sheet**：使用者指定沿用 `product-title-generator` 目前使用的既有表 <https://docs.google.com/spreadsheets/d/1pqGlCvUstowBzZh7J4xEa0jy3KoK4UeHUiyMTzcSGo4/edit>。為避免跟該表裡 `product-title-generator` 既有分頁的序號池混淆，`Code.gs` 固定操作一個**獨立分頁**「AmazonListing序號」（`SHEET_NAME` 常數），**不做跨分頁掃描比對**（跟 `product-title-generator` 那套「雙層掃描找表頭」的保守版本不同，因為這次分頁名稱是自己定義、位置已知）。分頁不存在時 `getLicenseSheet_()` 會自動 `insertSheet()` 並寫入表頭。
- **部署方式**：`clasp create --parentId <SheetID>`（不加 `--type`）→ 複製 `Code.gs` → `appsscript.json` 加 `webapp:{executeAs:"USER_DEPLOYING",access:"ANYONE_ANONYMOUS"}` → `clasp push --force` → `clasp deploy`，全程在 `.gas-deploy/`（已加入 `.gitignore`，不進版控）內操作。已部署完成：`LICENSE_CHECK_URL = https://script.google.com/macros/s/AKfycbxw9QjxS26AFmetLlW4yvhk4cYtQRjilRAiT8Jgp58IDPIoR-z3pJS6i98P_oyFnFXU/exec`，Apps Script 編輯器：<https://script.google.com/d/18pc6Rb0Uff0FSEeGrzBVRQZdDoTgAWvtY4INKwH10MIYCFfrBttMYL1b/edit>。
- **✅ 已完成部署與端對端驗證（2026-08-31）**：使用者已完成一次性 OAuth 授權，健康檢查與序號驗證皆已用瀏覽器 `fetch()` 實測成功（curl POST 測驗證會遇到已知的 Apps Script 轉址假失敗，見 `SETUP-授權伺服器設定.md`「常見問題」，需用真實瀏覽器測）。用 `product-title-generator` 共用的既有測試序號 `mark0131`（該試算表另一分頁「工作表1」也在用）驗證成功，`licenseGate` 正確解鎖並顯示「🔑 剩餘 487 天」。
  - **已知瑕疵（不影響功能）**：「AmazonListing序號」分頁在測試過程中被使用者不慎重複貼上舊的「任務追蹤」表格內容數十次（`任務/優先順序/負責人/狀態/序號/開始日期/結束日期/交件/附註`，含 `mark0131`／`x$6SzyoKLU3z` 兩組序號各重複約70次）。`checkOrActivate()` 用逐列掃描找第一筆序號相符的列即回傳，重複列不影響驗證正確性，純粹是視覺雜亂——使用者選擇不清理，直接沿用現狀，之後如需要新增其他序號，一樣是在這個分頁的「序號」欄找一個空位新增即可（欄位用文字比對非固定順序，即使夾在舊資料中間也能運作）。

## 頂部共用跑馬燈

`#marqueeBar` 內容抓自工作區既有的共用授權伺服器，做法逐字比照 `new-product-strategy-studio`。`localStorage` key：`amzListingMarquee`。跟本工具自己的序號授權後端是兩個互不相干的系統。

## PWA

`manifest.json`＋`service-worker.js`＋`icons/`（用 PIL 產生：深色圓角方塊＋琥珀金外框＋中央「5」字樣，呼應「5點式說明」）＋獨立安裝 IIFE，逐字複製既有模式。已用 fetch 實測 manifest/icons/service-worker 皆可正常存取（HTTP 200）。

## Port 分配

固定用 **8805**（已核對 `.claude/launch.json` 與全工作區 `launcher.py`，8765-8804 皆已占用）。`launcher.py` 已就緒，本次未打包桌面版 exe。

## 隱私與警語

無伺服器端經手使用者資料（序號授權後端除外，只傳送序號本身）；商品資料、產生內容、API設定皆只存在使用者瀏覽器的 localStorage。首頁與 manual.html 皆明列使用警語：合規檢查非官方審核、AI內容需自行查核、請勿輸入真實個資或機密資料、僅供教學與個人使用禁止商業化。

## 本次未做（後續視需要再處理）

- 桌面版 exe 未打包。
- 未實測真實 AI API 金鑰的端對端生成（`callLLM`/`extractJsonObject`/`validateAiResult` 邏輯與姊妹專案完全同款、已在其他專案端對端驗證過，本次僅靜態複製未重新測試）。

## 部署狀態：已全部完成（2026-08-31）

本機、序號授權後端（OAuth已完成、端對端驗證通過）、GitHub repo（<https://github.com/M255525/amazon-listing-generator>，public）、GitHub Pages（<https://m255525.github.io/amazon-listing-generator/>，Actions workflow `deploy-pages.yml` 部署，比照 `new-product-strategy-studio` 模式）皆已就緒並驗證通過。

## 指令

無建置/測試指令。修改 `index.html` 或 `manual.html` 後直接用瀏覽器開啟驗證，或暫起 `python -m http.server 8805 --directory 行銷內容工具/amazon-listing-generator` 測完關閉（或用 Preview MCP，設定於根目錄 `.claude/launch.json` 的 `amazon-listing-generator` 項目）。修改內嵌 `<script>` 後可用以下方式快速檢查語法：

```bash
python -c "
import re
html = open('index.html', encoding='utf-8').read()
open('_check.js','w',encoding='utf-8').write(re.findall(r'<script>(.*?)</script>', html, re.S)[0])
"
node --check _check.js
```

**測試序號授權邏輯前，需先完成 `SETUP-授權伺服器設定.md` 裡「還差最後一步」的手動 OAuth 授權**，否則會顯示「無法連線授權伺服器」並停留在鎖定畫面；開發階段要測試生成/合規檢查等其他功能，可在瀏覽器 devtools 對 `#licenseGate` 加上 `hidden` class 暫時繞過（本次即用此方式配合 Chrome 自動化實測過）。

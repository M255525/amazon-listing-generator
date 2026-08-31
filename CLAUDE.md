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

### Amazon 合規檢查（規則式，兩層防護的第一層）

`checkCompliance(text)`／`COMPLIANCE_RULES` 用 regex 偵測：價格符號、URL/email/電話、主觀宣傳詞（免運/限時/特價/保證/最好/第一名等）、HTML標籤，加上字數（`LEN_LIMIT=200`）警告。套用在**每一則標題與每一點內容**（不論規則式或AI產出）上，標紅/標黃顯示但不強制刪除——AI prompt 另外要求模型自行規避這些規則（第二層防護）。這只是常見類型的粗略掃描，不等於 Amazon 官方審核，manual.html／頁尾警語皆已明確揭露。

### 匯出

複製全部文字／下載 TXT／下載 CSV（`csvCell()`+UTF-8 BOM，逐字複製自 `product-title-generator` 的寫法）。

## 序號授權（鎖定整個工具，12 個月）

比照 `new-product-strategy-studio` 的「單一工具、整個鎖住」模式：`#licenseGate` 全螢幕遮罩預設鎖定，驗證通過才加上 `.hidden`；載入時一律對後端即時重驗，背景每 20 分鐘重驗一次。`localStorage` key：`alGenSerial`。

- **綁定的 Google Sheet**：使用者指定沿用 `product-title-generator` 目前使用的既有表 <https://docs.google.com/spreadsheets/d/1pqGlCvUstowBzZh7J4xEa0jy3KoK4UeHUiyMTzcSGo4/edit>。為避免跟該表裡 `product-title-generator` 既有分頁的序號池混淆，`Code.gs` 固定操作一個**獨立分頁**「AmazonListing序號」（`SHEET_NAME` 常數），**不做跨分頁掃描比對**（跟 `product-title-generator` 那套「雙層掃描找表頭」的保守版本不同，因為這次分頁名稱是自己定義、位置已知）。分頁不存在時 `getLicenseSheet_()` 會自動 `insertSheet()` 並寫入表頭。
- **部署方式**：`clasp create --parentId <SheetID>`（不加 `--type`）→ 複製 `Code.gs` → `appsscript.json` 加 `webapp:{executeAs:"USER_DEPLOYING",access:"ANYONE_ANONYMOUS"}` → `clasp push --force` → `clasp deploy`，全程在 `.gas-deploy/`（已加入 `.gitignore`，不進版控）內操作。已部署完成：`LICENSE_CHECK_URL = https://script.google.com/macros/s/AKfycbxw9QjxS26AFmetLlW4yvhk4cYtQRjilRAiT8Jgp58IDPIoR-z3pJS6i98P_oyFnFXU/exec`，Apps Script 編輯器：<https://script.google.com/d/18pc6Rb0Uff0FSEeGrzBVRQZdDoTgAWvtY4INKwH10MIYCFfrBttMYL1b/edit>。
- **⚠️ 尚未完成最後一步**：`clasp deploy` 跳過瀏覽器部署精靈附帶的一次性 OAuth 授權，目前開部署網址仍顯示 Google「需要存取權」擋案頁（已用 curl 實測確認）。需要使用者本人到 Apps Script 編輯器手動執行一次 `doGet` 完成授權（涉及 Google 帳號互動，Claude 無法代勞），步驟詳見 `SETUP-授權伺服器設定.md`。**完成授權前，序號驗證會一直顯示「無法連線授權伺服器」並停留在鎖定畫面**——開發階段測試其他功能可在瀏覽器 devtools 對 `#licenseGate` 加 `hidden` class 暫時繞過（已用 Playwright/Chrome 這樣測過規則式生成與合規檢查兩條路徑，皆正常）。

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
- 「AmazonListing序號」分頁內尚未有真實序號可測試「解鎖成功＋剩餘天數顯示」這條路徑（目前只驗證過「查無序號/連線失敗」的拒絕路徑，以及繞過閘門後的規則式生成/合規檢查功能）。
- 未實測真實 AI API 金鑰的端對端生成（`callLLM`/`extractJsonObject`/`validateAiResult` 邏輯與姊妹專案完全同款、已在其他專案端對端驗證過，本次僅靜態複製未重新測試）。
- 未推公開 GitHub repo / 未部署 GitHub Pages。

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

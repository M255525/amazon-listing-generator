# 授權伺服器設定指南

這份工具（亞馬遜5點式產品說明產生器）在頁面一開啟就會顯示「授權序號」的鎖定畫面，**必須輸入序號並按「確認」驗證通過，才能使用整個工具**。序號需連到 Google Sheet 確認是否還在 12 個月使用期限內，透過 **Google Apps Script**（Google Sheet 內建、免費）架設的一支小型 API 完成。

## 綁定的 Google Sheet

<https://docs.google.com/spreadsheets/d/1pqGlCvUstowBzZh7J4xEa0jy3KoK4UeHUiyMTzcSGo4/edit>

這份 Sheet 也被 `product-title-generator` 用來做序號驗證。為了不互相混淆，本工具固定操作一個**獨立分頁**「AmazonListing序號」（`Code.gs` 裡的 `SHEET_NAME` 常數），第一次呼叫時若這個分頁不存在會**自動建立**並寫入表頭（序號／開始日期／結束日期）。

## 部署狀態：已用 clasp 完成部署

已用 `clasp create --parentId <此Sheet的檔案ID>`（不加 `--type`，才能正確綁定到既有 Sheet）建立綁定腳本專案 → 複製 `Code.gs` → `appsscript.json` 加上 `webapp:{executeAs:"USER_DEPLOYING", access:"ANYONE_ANONYMOUS"}` → `clasp push --force` → `clasp deploy`，全程跳過瀏覽器複製貼上與部署精靈。

部署網址已回填到 `index.html` 的 `LICENSE_CHECK_URL`：
```
https://script.google.com/macros/s/AKfycbxw9QjxS26AFmetLlW4yvhk4cYtQRjilRAiT8Jgp58IDPIoR-z3pJS6i98P_oyFnFXU/exec
```
Apps Script 編輯器（若之後要手動改程式碼或管理部署）：<https://script.google.com/d/18pc6Rb0Uff0FSEeGrzBVRQZdDoTgAWvtY4INKwH10MIYCFfrBttMYL1b/edit>

### ⚠️ 還差最後一步：需要手動授權一次（Claude 無法代勞）

`clasp deploy` 用 API 建立部署會跳過瀏覽器的「部署」精靈畫面，**但也因此跳過了 Google 要求的一次性 OAuth 授權**（讓這支腳本有權限讀寫這份 Sheet）。目前直接開啟部署網址會看到 Google 的存取阻擋頁面（已用 curl 實測確認，回應「需要存取權」）。修法：

1. 開啟上面的 Apps Script 編輯器連結。
2. 上方工具列函式下拉選單選 `doGet`，點「執行（Run）」▷ 按鈕。
3. 會跳出「需要授權」→ 選你的帳號 → 若出現「Google 尚未驗證這個應用程式」，點左下角「進階」→「前往...(不安全)」→ 允許。這是正常現象（因為這是你自己寫的私人腳本，沒有送 Google 審查），不是真的有安全疑慮。
4. 執行完成後（下方「執行紀錄」顯示成功），部署網址就會正常運作，不需要重新部署。

## 驗證部署是否成功

完成上面的手動授權步驟後，把部署網址直接貼到瀏覽器網址列開啟（GET 請求），應該會看到：

```json
{"ok":true,"message":"授權伺服器運作中。請用 POST 傳送 JSON body，例如 {\"serial\":\"your-serial-here\"}"}
```

看到這個就代表部署成功可用。**注意：不要用 `curl` 測試實際的驗證（POST 請求）**，Apps Script 的轉址機制會讓 curl 出現誤導性錯誤（不代表真的壞了）；請直接開啟工具測試，到「AmazonListing序號」分頁新增一組測試序號並按「確認」，確認鎖定畫面消失、狀態欄顯示「✓ 剩餘 N 天可用」。

## 之後每次要發新的序號要做什麼

**不需要重新部署 Apps Script。** 只要：

1. 打開 Google Sheet，切到「AmazonListing序號」分頁（第一次會由程式自動建立）。
2. 「序號」欄填一組你要發出去的序號（例如用工作區的 `SN-maker` 序號產生器批次產生）。
3. 「開始日期」「結束日期」兩欄**留空**——第一次有人驗證這組序號時，系統會自動把「開始日期」寫成當下時間，「結束日期」自動算成開始日期 + 12 個月。
4. 把這組序號發給該使用者。

## 修改使用期限長度

在 `Code.gs` 開頭的 `const VALID_AMOUNT = 12;` 改掉這個數字，改完要回到 Apps Script 編輯器貼上新版程式碼（或 `clasp push --force`），再「部署 → 管理部署作業 → 編輯 → 部署」一次（**部署網址不會變**）。已經驗證啟用過、結束日期已寫入的序號不會回溯套用新期限。

## 常見問題

- **使用者按確認一直顯示「無法連線授權伺服器」或畫面顯示存取阻擋**：多半是還沒完成上面「還差最後一步」的手動授權動作。
- **改了 Code.gs 之後網址失效或行為沒變**：Apps Script 修改程式碼後，必須到「部署 → 管理部署作業 → 編輯（鉛筆圖示）→ 版本選「新版本」→ 部署」才會生效。
- **想收回某組序號的使用權**：把該列的「結束日期」改成一個過去的日期即可，之後驗證都會回傳逾期，畫面會在最多 20 分鐘內自動重新鎖定。

# 維基國際短訊 自動化

每週自動產生維基媒體運動「國際短訊」，用本機 `mail` 寄給秘書長過目、篩選後採用至月訊。
台灣維基媒體協會用。

短訊的**產生規則**（來源、挑選、翻譯、格式、收件人、寄送方式）全部寫在 `wmtw-briefs.md`，
本文件不重複，只談簡介、部署與維護。

運作方式：cron 每週觸發一次 Claude Code headless（`claude -p`），把 `wmtw-briefs.md` 當提示
餵進去，它讀 wikimedia-l 過去七天內容、產繁中純文字短訊、用本機 `mail` 寄出，全程無人工介入。
用訂閱登入憑證執行，不另計付費 API。

## 部署

### 前置需求

執行 cron 的主機需具備：

- **Claude Code CLI**，且已用會員帳號 `/login` 過（headless 才有訂閱憑證、不另計費）；
  cron 須掛在這個已登入的使用者底下。
- **本機 mail**（如 exim4／sendmail，免 SMTP 認證即可送出）。
- **locale `zh_TW.UTF-8`**（中文主旨與內文才正常）。

### 步驟

部署位置（系統路徑、crontab）為固定處所，以下用絕對路徑；其餘指涉本專案檔案處皆用相對路徑。

1. 把指示檔放到系統位置：

   ```
   sudo cp wmtw-briefs.md /etc/wmtw-briefs.md
   ```

2. 找出 claude 執行檔的完整路徑（cron 的 PATH 很精簡，必須寫完整路徑）：

   ```
   command -v claude
   ```

3. 在**已登入會員帳號的使用者**底下 `crontab -e`，加入下列一行
   （把 `<claude完整路徑>` 換成上一步結果）：

   ```
   0 18 * * 1 <claude完整路徑> -p "$(cat /etc/wmtw-briefs.md)" --allowedTools WebFetch Bash >> /tmp/wmtw-briefs.log 2>&1
   ```

   - `0 18 * * 1`：每週一 18:00，可自行調整時段。
   - headless 放行的工具：`WebFetch`、`Bash`。
   - 收件人寫在 `/etc/wmtw-briefs.md` 裡，不在 crontab。

## 維護

- **改產生規則或收件人**：編輯部署後的 `/etc/wmtw-briefs.md`，不必動 crontab。
- **沒收到信**：查 crontab 導向的 log；確認郵件佇列沒卡（`mailq`）；
  確認 claude 訂閱憑證未過期（在該使用者底下重跑互動式 `claude`，若要求重新登入即為過期）。
- **中文亂碼**：檢查 crontab 那行有沒有帶 `LANG=zh_TW.UTF-8`。
- **內容偏少**：wikimedia-l 某些週本來就稀疏，指示檔已交代據實產短、不硬湊，屬正常。
- **上線前測試**：複製一份指示檔、把收件人改成自己（勿寄秘書長），手動跑
  `<claude完整路徑> -p "$(cat <測試指示檔>)" --allowedTools WebFetch Bash`，
  確認中文、純文字排版與內容無誤後再正式啟用。

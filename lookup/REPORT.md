# REPORT 開發回報

> 開發師每完成或卡住一張單就在最上面新增一段。

## T-001 ｜ 2026-11-22 ｜ 待驗收

**做了什麼**
- 寫好唯讀查詢端點 `doGet` 的程式碼（下方），用模擬資料測過四種結果。
- 程式只「讀」開通紀錄，不寫入、不寄信、不碰 Drive 權限。

**驗收標準核對**
- [x] 只讀取「開通紀錄」分頁，不寫入任何資料
- [x] 沒有 GmailApp、MailApp、DriveApp
- [x] 學號格式錯誤回傳 `status: "invalid"`
- [x] 帳號遮罩：保留前 2 碼和網域，例如 `ab****@gmail.com`
- [x] CacheService 快取 10 分鐘
- [x] 附上部署步驟

**回傳的狀態**

| status | 意思 | 判斷方式 |
|---|---|---|
| `approved` | 已開通 | 該學號有任何一筆「已開通」 |
| `pending` | 處理中 | 只有「需人工確認」 |
| `revoked` | 已收回 | 只有「已撤銷」 |
| `notfound` | 查無申請 | 沒有任何紀錄（含用個人信箱寄信的人） |
| `invalid` | 學號格式錯誤 | 不是 6～7 位數字 |

`approved` 會附上 `grants` 清單（國中部、國三加開高中部會分兩筆），每筆有遮罩帳號和到期日。

**部署步驟（啾咪手動，約 5 分鐘）**
1. 用能開啟「考古題申請系統」試算表的 Google 帳號，打開 https://script.new 建立**新的** Apps Script 專案，命名「考古題查詢端點」。原本的「衛道考古題自動開通」不用動。
2. 刪掉預設內容，貼上下方程式碼。
3. 把第一行的 `請貼上試算表ID` 換成 `1pL89ZxOZgaJPOZjXr5--PysWLD1HpKHo3YfSww5SAtI`。（只改在 Apps Script 裡，不要寫回這個公開 repo。）
4. 右上角「部署」→「新增部署作業」→ 齒輪選「網頁應用程式」
   - 執行身分：**我**
   - 誰可以存取：**所有人**
5. 按「部署」，第一次會要求授權，只需要「查看試算表」的權限。如果授權畫面出現 Gmail 或 Drive 權限，表示貼錯程式，請停下來。
6. 複製「網頁應用程式網址」（結尾是 `/exec`），貼給 Claude，就能接著做 T-002。
7. 自我測試：在網址後面加上 `?sid=1150630`，用瀏覽器打開，應該看到 `"status":"approved"`。

**建議程式碼**

```javascript
const SHEET_ID = '請貼上試算表ID';
const SHEET_NAME = '開通紀錄';
const CACHE_SECONDS = 600; // 10 分鐘

function doGet(e) {
  const sid = String((e && e.parameter && e.parameter.sid) || '').trim();
  const out = ContentService.createTextOutput(JSON.stringify(lookup_(sid)));
  out.setMimeType(ContentService.MimeType.JSON);
  return out;
}

function lookup_(sid) {
  if (!/^\d{6,7}$/.test(sid)) return { status: 'invalid' };

  const cache = CacheService.getScriptCache();
  const hit = cache.get('sid_' + sid);
  if (hit) return JSON.parse(hit);

  const rows = SpreadsheetApp.openById(SHEET_ID)
    .getSheetByName(SHEET_NAME)
    .getDataRange()
    .getValues()
    .slice(1);

  const result = judge_(sid, rows);
  cache.put('sid_' + sid, JSON.stringify(result), CACHE_SECONDS);
  return result;
}

// 欄位：A 時間｜B 學校信箱｜C 開通帳號｜D 到期日｜E 狀態｜F 備註
function judge_(sid, rows) {
  const mail = sid + '@vtsh.tc.edu.tw';
  const mine = rows.filter(r => String(r[1]).trim().toLowerCase() === mail);
  if (mine.length === 0) return { status: 'notfound' };

  const approved = mine.filter(r => r[4] === '已開通');
  if (approved.length > 0) {
    const seen = {};
    const grants = [];
    approved.forEach(r => {
      const exp = fmtDate_(r[3]);
      const acct = mask_(String(r[2]).trim());
      const key = acct + '|' + exp;
      if (!seen[key]) { seen[key] = true; grants.push({ account: acct, expire: exp }); }
    });
    grants.sort((a, b) => a.expire < b.expire ? -1 : 1);
    return { status: 'approved', approvedDate: fmtDate_(approved[0][0]), grants: grants };
  }

  if (mine.some(r => r[4] === '需人工確認')) {
    const last = mine.filter(r => r[4] === '需人工確認').pop();
    return { status: 'pending', hint: hint_(String(last[5])) };
  }
  if (mine.some(r => r[4] === '已撤銷')) return { status: 'revoked' };
  return { status: 'pending', hint: '管理員確認中' };
}

// 不把內部備註原文公開，只給申請者看得懂的提示
function hint_(note) {
  if (/帳號/.test(note) && /(多個|剛好一個|沒有)/.test(note)) return '信件內文請只寫一個要開通的 Google 帳號，再重新寄一次';
  if (/並非使用 Google 帳戶|分享失敗/.test(note)) return '您填的帳號不是 Google 帳號，請確認拼字後重新寄一次';
  return '管理員確認中，請過幾天再查詢';
}

function mask_(email) {
  const at = email.indexOf('@');
  if (at < 0) return '****';
  const local = email.slice(0, at);
  return local.slice(0, 2) + '****' + email.slice(at);
}

function fmtDate_(v) {
  if (!(v instanceof Date)) return '';
  return Utilities.formatDate(v, 'Asia/Taipei', 'yyyy/MM/dd');
}
```

**需要啾咪決定**
- 任何人輸入別人的學號都能查到「是否開通」和遮罩後的帳號。我認為風險可以接受（看不到完整帳號）；如果你希望更嚴格，可以改成只顯示「已開通／未開通」。

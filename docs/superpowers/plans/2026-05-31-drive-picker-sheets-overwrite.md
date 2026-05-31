# Drive Picker + Sheets上書きエクスポート 実装計画

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `google-sheets-poc.html` に Google Picker API によるDriveファイル選択と、Sheetsエクスポートの上書き機能を追加する。

**Architecture:** 単一HTMLファイルへの追加。gapi.js を `<head>` に追加し、Picker起動関数・Sheets上書き関数・localStorage復元ロジックをインラインJSに追加する。状態変数 `exportedSheetsId` / `exportedSheetsTitle` で上書き先を管理する。

**Tech Stack:** Vanilla JS、Google Picker API（gapi.js）、Google Sheets API v4、localStorage

---

## ファイル変更マップ

| ファイル | 変更内容 |
|---|---|
| `google-sheets-poc.html` | gapi.js追加・Pickerボタン追加・Picker関数追加・Sheets上書きUI追加・overwriteSheets()追加・exportToSheets()修正・onload修正 |

---

## Task 1: gapi.js の読み込みを追加

**Files:**
- Modify: `google-sheets-poc.html`（`<head>` 内）

- [ ] **Step 1: gapi.js の script タグを追加**

`<head>` の既存 `gsi/client` スクリプトの直後に1行追加する:

```html
  <script src="https://accounts.google.com/gsi/client" async defer></script>
  <script src="https://apis.google.com/js/api.js" async defer></script>
```

- [ ] **Step 2: ブラウザで読み込み確認**

```bash
python3 -m http.server 8080
```

`http://localhost:8080/google-sheets-poc.html` を開き、DevTools Console に `gapi` と入力して `Object` が返ることを確認。エラーが出ないこと。

- [ ] **Step 3: コミット**

```bash
git add google-sheets-poc.html
git commit -m "feat: gapi.js を追加（Picker用）"
```

---

## Task 2: 「Driveから選ぶ」ボタンをUIに追加

**Files:**
- Modify: `google-sheets-poc.html`（`#file-section` のHTML）

- [ ] **Step 1: ボタンを追加**

`#file-section` の `<button onclick="loadFile()">開く</button>` の直後にボタンを追加:

```html
<div id="file-section" class="section hidden">
  <label>ファイルID: <input id="file-id-input" type="text" size="44" placeholder="Google Drive の file ID"></label>
  <button onclick="loadFile()">開く</button>
  <button onclick="showDrivePicker()">Driveから選ぶ</button>
</div>
```

- [ ] **Step 2: ブラウザで表示確認**

ログイン後に「Driveから選ぶ」ボタンが表示されること。クリックしてもまだ関数未定義のエラーが Console に出るのは正常。

- [ ] **Step 3: コミット**

```bash
git add google-sheets-poc.html
git commit -m "feat: 「Driveから選ぶ」ボタンをUIに追加"
```

---

## Task 3: Drive Picker 関数を実装

**Files:**
- Modify: `google-sheets-poc.html`（`<script>` 内）

- [ ] **Step 1: 状態変数とPicker関数を追加**

`let currentFileId = null;` の直後に追加:

```js
let exportedSheetsId = null;
let exportedSheetsTitle = null;
```

`signOut()` 関数の直後に追加:

```js
function showDrivePicker() {
  if (!accessToken) { showStatus('先にログインしてください', true); return; }
  gapi.load('picker', () => {
    const view = new google.picker.DocsView()
      .setMimeTypes('application/json');
    const picker = new google.picker.PickerBuilder()
      .addView(view)
      .setOAuthToken(accessToken)
      .setDeveloperKey(API_KEY)
      .setCallback(pickerCallback)
      .build();
    picker.setVisible(true);
  });
}

function pickerCallback(data) {
  if (data.action === google.picker.Action.PICKED) {
    const file = data.docs[0];
    document.getElementById('file-id-input').value = file.id;
    loadFile();
  }
}
```

- [ ] **Step 2: ブラウザで動作確認**

1. ログイン
2. 「Driveから選ぶ」クリック → Google Pickerのモーダルが開くこと
3. JSONファイルを選択 → ファイルIDが入力欄にセットされ、データが読み込まれること
4. キャンセル → モーダルが閉じるだけで何も起きないこと
5. ログアウト状態で「Driveから選ぶ」→「先にログインしてください」が出ること

- [ ] **Step 3: コミット**

```bash
git add google-sheets-poc.html
git commit -m "feat: Google Picker APIによるDriveファイル選択を実装"
```

---

## Task 4: Sheets上書きUIを追加

**Files:**
- Modify: `google-sheets-poc.html`（`#data-section` 内のボタン部分）

- [ ] **Step 1: ボタン構成を変更**

`#data-section` 内の Sheets エクスポートボタン部分を以下に変更:

```html
  <div style="margin-top: 16px">
    <button class="primary" onclick="saveToDrive()">Driveに保存</button>
    <button id="export-new-btn" onclick="exportToSheets()">新規エクスポート</button>
    <button id="export-overwrite-btn" style="display:none" onclick="overwriteSheets()"></button>
  </div>
```

- [ ] **Step 2: updateSheetsButtons() ヘルパー関数を追加**

`showStatus()` 関数の直前に追加:

```js
function updateSheetsButtons() {
  const btn = document.getElementById('export-overwrite-btn');
  if (exportedSheetsId) {
    btn.textContent = `「${exportedSheetsTitle}」を上書き`;
    btn.style.display = '';
  } else {
    btn.style.display = 'none';
  }
}
```

- [ ] **Step 3: window.onload で localStorage から復元**

`window.onload` の末尾（`if (fileId) ...` の直後）に追加:

```js
  exportedSheetsId = localStorage.getItem('sheetsId');
  exportedSheetsTitle = localStorage.getItem('sheetsTitle');
  if (exportedSheetsId) updateSheetsButtons();
```

- [ ] **Step 4: ブラウザで表示確認**

ページを開いた時点で localStorage に保存済みデータがあれば「上書き」ボタンが表示されること。なければ「新規エクスポート」のみ表示されること。

- [ ] **Step 5: コミット**

```bash
git add google-sheets-poc.html
git commit -m "feat: Sheets上書きUIボタンとlocalStorage復元を追加"
```

---

## Task 5: exportToSheets() を修正して localStorage に保存

**Files:**
- Modify: `google-sheets-poc.html`（`exportToSheets()` 関数）

- [ ] **Step 1: exportToSheets() の末尾に保存処理を追加**

`exportToSheets()` 内の最終 `showStatus(...)` 行を以下に置き換え:

```js
  const ssUrl = `https://docs.google.com/spreadsheets/d/${ssId}`;
  exportedSheetsId = ssId;
  exportedSheetsTitle = ss.properties.title;
  localStorage.setItem('sheetsId', exportedSheetsId);
  localStorage.setItem('sheetsTitle', exportedSheetsTitle);
  updateSheetsButtons();
  showStatus(`エクスポート完了 → <a href="${ssUrl}" target="_blank" style="color:#065f46">スプレッドシートを開く</a>`);
```

- [ ] **Step 2: ブラウザで動作確認**

1. ログイン・ファイル読み込み
2. 「新規エクスポート」クリック → Sheetsが作成されリンクが表示されること
3. 「上書き」ボタンが現れ、タイトルが表示されること
4. ページをリロード → ログイン後に「上書き」ボタンが復元されること

- [ ] **Step 3: コミット**

```bash
git add google-sheets-poc.html
git commit -m "feat: エクスポート後にSheetsIDをlocalStorageへ保存"
```

---

## Task 6: overwriteSheets() を実装

**Files:**
- Modify: `google-sheets-poc.html`（`exportToSheets()` の直後）

- [ ] **Step 1: overwriteSheets() を追加**

`exportToSheets()` 関数の直後に追加:

```js
async function overwriteSheets() {
  const nodes = collectTable('nodes', NODE_FIELDS);
  const links = collectTable('links', LINK_FIELDS);
  showStatus('上書き中...');

  const clearRes = await fetch(
    `https://sheets.googleapis.com/v4/spreadsheets/${exportedSheetsId}/values:batchClear`,
    {
      method: 'POST',
      headers: { Authorization: `Bearer ${accessToken}`, 'Content-Type': 'application/json' },
      body: JSON.stringify({ ranges: ['Nodes!A:Z', 'Links!A:Z'] }),
    }
  );
  if (!clearRes.ok) {
    if (clearRes.status === 404) {
      exportedSheetsId = null;
      exportedSheetsTitle = null;
      localStorage.removeItem('sheetsId');
      localStorage.removeItem('sheetsTitle');
      updateSheetsButtons();
      showStatus('スプレッドシートが見つかりません。新規エクスポートしてください', true);
    } else {
      showStatus(`クリアエラー: ${clearRes.status} ${clearRes.statusText}`, true);
    }
    return;
  }

  const writeRes = await fetch(
    `https://sheets.googleapis.com/v4/spreadsheets/${exportedSheetsId}/values:batchUpdate`,
    {
      method: 'POST',
      headers: { Authorization: `Bearer ${accessToken}`, 'Content-Type': 'application/json' },
      body: JSON.stringify({
        valueInputOption: 'RAW',
        data: [
          { range: 'Nodes!A1', values: [NODE_FIELDS, ...nodes.map(n => NODE_FIELDS.map(f => n[f] || ''))] },
          { range: 'Links!A1', values: [LINK_FIELDS, ...links.map(l => LINK_FIELDS.map(f => l[f] || ''))] },
        ],
      }),
    }
  );
  if (!writeRes.ok) { showStatus(`書き込みエラー: ${writeRes.status} ${writeRes.statusText}`, true); return; }

  const ssUrl = `https://docs.google.com/spreadsheets/d/${exportedSheetsId}`;
  showStatus(`上書き完了 → <a href="${ssUrl}" target="_blank" style="color:#065f46">スプレッドシートを開く</a>`);
}
```

- [ ] **Step 2: ブラウザで動作確認**

1. ログイン・ファイル読み込み
2. 「新規エクスポート」でSheetsを作成
3. データを編集
4. 「上書き」ボタンをクリック → 同じスプレッドシートに上書きされること
5. Sheetsを開いてデータが更新されていること
6. Sheetsファイルを手動削除し「上書き」→「スプレッドシートが見つかりません」エラーが出て、上書きボタンが消えること

- [ ] **Step 3: コミット**

```bash
git add google-sheets-poc.html
git commit -m "feat: Sheets上書きエクスポート（overwriteSheets）を実装"
```

---

## 完了チェック

- [ ] Driveから選ぶボタンでPickerが開き、JSONファイルを選択してデータ読み込みができる
- [ ] キャンセルしても何も起きない
- [ ] 新規エクスポートでSheetsが作成され、上書きボタンが出現する
- [ ] リロード後も上書きボタンが復元される
- [ ] 上書きで同じSheets IDにデータが反映される
- [ ] 削除済みSheets上書き時にエラーメッセージが出てボタンがリセットされる

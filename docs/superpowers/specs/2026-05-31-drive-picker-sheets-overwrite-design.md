# Design: Drive Picker + Sheets上書きエクスポート

**日付:** 2026-05-31  
**対象ファイル:** `google-sheets-poc.html`  
**目的:** PoCとして Google Drive Picker API と Sheets上書きエクスポートの挙動を確認する

---

## 背景

現状、ファイルの読み込みはFileIDを手入力するしかなく、SheetsエクスポートはAPIコールのたびに新規スプレッドシートが作られる。PoCとして以下2機能を追加する。

---

## 変更スコープ

`google-sheets-poc.html` の1ファイルのみ変更。新規ファイル作成なし。

---

## 変更点1: Drive Picker によるファイル選択

### UI

```
ファイルID: [__________]  [開く]  [Driveから選ぶ]
```

### 必要な定数

```js
const API_KEY = 'YOUR_API_KEY_HERE'; // Cloud ConsoleのAPIキー（Picker用）
```

`CLIENT_ID` と同様に HTML 直書き。Cloud Console → 「認証情報」→「APIキーを作成」で取得。

### 読み込み

`<head>` に下記を追加:

```html
<script src="https://apis.google.com/js/api.js" async defer></script>
```

### Picker 起動フロー

1. 「Driveから選ぶ」クリック → ログイン未済なら「先にログインしてください」をステータス表示して終了
2. `gapi.load('picker', callback)` で Picker ライブラリを遅延読み込み
3. `PickerBuilder` を組み立て:
   - `.setOAuthToken(accessToken)`
   - `.setDeveloperKey(API_KEY)`
   - `DocsView` に `.setMimeTypes('application/json')` を設定
4. `.build().setVisible(true)` でモーダル表示
5. コールバック内で `action === google.picker.Action.PICKED` のとき:
   - `docs[0].id`（fileId）と `docs[0].name`（fileName）を取得
   - `#file-id-input` に fileId をセット
   - `loadFile()` を呼び出す
6. キャンセル時は何もしない

---

## 変更点2: Sheets エクスポートの上書き対応

### UI（初回・localStorage未保存）

既存の「Sheetsにエクスポート」ボタンのみ表示。挙動は現在と同じ（新規作成）。

### エクスポート成功後

- `localStorage.setItem('sheetsId', ssId)` と `localStorage.setItem('sheetsTitle', ss.properties.title)` に保存
- ボタン表示を以下に切り替える:

```
[新規エクスポート]  [「system-map エクスポート」を上書き ↗]
```

「↗」は `https://docs.google.com/spreadsheets/d/${sheetsId}` を `target="_blank"` で開くリンク。

### 上書き処理フロー

1. `batchClear` で既存データをクリア（範囲: `Nodes!A:Z` / `Links!A:Z`）
2. `batchUpdate` で現在の編集内容を書き込む（新規作成と同じロジックを流用）
3. ステータスに「上書き完了 → スプレッドシートを開く」を表示

### ページリロード後の復元

`window.onload` 内で localStorage を読み取り:

```js
const savedSheetsId = localStorage.getItem('sheetsId');
const savedSheetsTitle = localStorage.getItem('sheetsTitle');
if (savedSheetsId) {
  exportedSheetsId = savedSheetsId;
  exportedSheetsTitle = savedSheetsTitle;
  // 上書きボタンを表示状態にする
}
```

---

## 状態変数（追加）

| 変数名 | 型 | 用途 |
|---|---|---|
| `exportedSheetsId` | string \| null | 最後にエクスポートした Sheets の ID |
| `exportedSheetsTitle` | string \| null | 同タイトル（ボタンラベル用） |

---

## エラーハンドリング

| ケース | 処理 |
|---|---|
| ログイン前に「Driveから選ぶ」を押した | 「先にログインしてください」をステータス表示 |
| Picker キャンセル | 何もしない |
| 上書き先の Sheets が削除済み | `batchClear` / `batchUpdate` のHTTPエラーをキャッチして「エラー: スプレッドシートが見つかりません。新規作成してください」を表示し、localStorageをクリア |

---

## 対象外（今回スコープ外）

- Drive Picker を Sheets ファイル選択にも使う（将来検討）
- OAuth トークンのリフレッシュ処理
- `drive` スコープを `drive.file` に絞る（本番移行時に対応）

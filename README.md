# Google Drive PoC

Google Drive API / Sheets API の動作確認用 PoC アプリ。

## 機能

- Googleアカウントでログイン（OAuth2）
- Google DriveのJSONファイルを読み込み・編集・保存
- `?fileId=XXXX` でURLから直接ファイルを開く
- ノード/矢印一覧をGoogleスプレッドシートにエクスポート

## セットアップ手順

### 1. Google Cloud Console でプロジェクトを作成

https://console.cloud.google.com/ にアクセス → 「新しいプロジェクト」で作成

### 2. APIを有効化

「APIとサービス」→「ライブラリ」で以下を有効化:
- Google Drive API
- Google Sheets API

### 3. OAuth同意画面を設定

「APIとサービス」→「OAuth同意画面」:
- ユーザーの種類: 外部
- スコープ: `drive`、`spreadsheets`
- テストユーザー: 使用するGmailアドレスを追加

### 4. OAuth2クライアントIDを作成

「認証情報」→「OAuth クライアント ID」→ ウェブアプリケーション

承認済みの JavaScript 生成元に追加:
- `http://localhost:8080`（ローカル確認用）
- `https://[GitHubユーザー名].github.io`（GitHub Pages用）

### 5. クライアントIDをHTMLに設定

`google-sheets-poc.html` の以下の行を書き換える:

```
const CLIENT_ID = 'YOUR_CLIENT_ID_HERE'; // ← クライアントIDを貼る
```

### 6. ローカルで確認

```
python3 -m http.server 8080
```

http://localhost:8080/google-sheets-poc.html を開く

### 7. GitHub Pagesで公開（任意）

Settings → Pages → Source: `main` ブランチ → Save

公開URL: `https://[ユーザー名].github.io/gdrive-poc/google-sheets-poc.html`

## テスト用サンプルJSON

以下のJSONをGoogleドライブにアップロードして動作確認に使用する:

```
{
  "nodes": [
    { "key": "node1", "nodeType": "system",  "title": "会計システム",  "segment1": "財務会計", "notes": "本番稼働中" },
    { "key": "node2", "nodeType": "process", "title": "月次締め処理", "segment1": "管理会計", "notes": "" }
  ],
  "links": [
    { "id": "L001", "from": "node1", "to": "node2", "title": "仕訳データ連携", "notes": "月次バッチ" }
  ]
}
```

ドライブ上のファイルURLから fileId を取り出す:

```
https://drive.google.com/file/d/[ここがfileId]/view
```

## 注意事項

- OAuth2トークンの有効期限は1時間。期限切れ後は再ログインが必要。
- 「Sheetsにエクスポート」は毎回新規スプレッドシートを作成する（既存を上書きしない）。
- PoCのOAuthスコープは `drive`（広め）。本番組み込み時は `drive.file` に絞ることを推奨。

# Google Drive 連携 PoC メモ

### やったこと

Google Drive API を使ってブラウザアプリからデータを読み書きできるか確認するため、本番アプリ（system-map-visual）とは別のリポジトリ（`gdrive-poc`）を新設してPoCを実施した。

**確認できた機能:**
- GoogleアカウントでのOAuth2ログイン
- DriveのJSONファイルを読み込み・編集・書き戻し
- URLパラメータ（`?fileId=XXXX`）でファイルを直接開く
- スプレッドシートへのエクスポート

---

### 仕組みのポイント

- ブラウザアプリからGoogle APIを呼ぶには、**静的Webサーバーが必須**（`file://` では動かない）
- Google Cloud Console でプロジェクトを作成し、**OAuth2クライアントID**を取得する必要がある
- そのクライアントIDをアプリの HTML に記載することで、APIアクセスが可能になる

---

### 注意点

- **今回のCloud ConsoleプロジェクトはプライベートのGoogleアカウント（tokyotax.net）で作成した。** fout.jp 組織内でのプロジェクト作成には請求先アカウントの設定が必要なため、情シスへの確認が必要。
- レポジトリに実際のJSONデータ（機密情報）が含まれないよう注意が必要。

---

### 後で削除するもの

| 削除対象 | 場所 |
|---------|------|
| テスト用リポジトリ | GitHub: `t25x/gdrive-poc` |
| テスト用プロジェクト | Google Cloud Console（tokyotax.netアカウント） |

---

### 本番環境を作るのに必要なこと

1. **静的Webサーバーの用意**（GitHub Pages でも可。ただしレポジトリにデータを含めないこと）
2. **fout.jp 組織で Google Cloud Console プロジェクトを作成**（情シスに確認要）
   - 作成手順は下記「セットアップ手順」を参照
   - 完成したら CLIENT_ID をアプリ側の HTML に記載する

---

## セットアップ手順（本番環境構築時に使う）

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

---

## 技術メモ

- OAuth2トークンの有効期限は1時間。期限切れ後は再ログインが必要（本番ではリフレッシュ処理を追加する）。
- 「Sheetsにエクスポート」は毎回新規スプレッドシートを作成する（既存を上書きしない）。
- PoCのOAuthスコープは `drive`（広め）。本番組み込み時は `drive.file` に絞ることを推奨。

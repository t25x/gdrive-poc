# Google Drive 連携 PoC メモ

> ⚠️ **このリポジトリは GitHub で公開されています。機密情報・実データをコミットしないこと。**
> CLIENT_ID と API_KEY はコードに含めて問題ない（詳細は後述）。

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
      - 完成したら CLIENT_ID と API_KEY をアプリ側の HTML に記載する（API_KEY はリファラー制限設定後）

---

## セットアップ手順（本番環境構築時に使う）

### 1. Google Cloud Console でプロジェクトを作成

https://console.cloud.google.com/ にアクセス → 「新しいプロジェクト」で作成

### 2. APIを有効化

「APIとサービス」→「ライブラリ」で以下を有効化:
- Google Drive API
- Google Sheets API
- Google Picker API（Drive ファイル選択UIに必要）

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

### 5. APIキーを作成（Picker用）

「認証情報」→「認証情報を作成」→「APIキー」をクリック。

API作成時に以下のとおり設定する。または、キー作成後に「キーを制限」ボタンをクリックして編集画面を開く（後からキー名をクリックしても同じ画面に入れる）。

**名前**（任意・デフォルトでもOK）
- 例: `gdrive-poc-picker` など分かりやすい名前に変えておくと管理しやすい

**APIの制限**（このキーで呼べるAPIを絞る）
- 「キーを制限する」を選択
- 「Google Picker API」**のみ**チェック（Drive API・Sheets API はチェック不要）
  > Drive API・Sheets API は OAuth トークン経由で呼ぶためAPIキー不要。最小権限にしておく。

**「サービスアカウントを介してAPI呼び出しを認証する」**
- チェック不要

**アプリケーションの制限**（どこから使えるかの制限）
- 「ウェブサイト(HTTPリファラー)」を選択
  > ※ OAuth2クライアントIDの「承認済みの JavaScript 生成元」とは**別の設定**
- 「アイテムを追加」で以下を3件追加:
  - `http://localhost:8080/*`
  - `https://[GitHubユーザー名].github.io/*`
  - `https://[GitHubユーザー名].github.io/gdrive-poc/*`
  > Picker が内部的に呼ぶリクエストの Referer がサブパス抜きの `github.io/*` になる場合があるため、両方設定が必要。

設定後「保存」をクリック。

> **CLIENT_ID と API_KEY のコード埋め込みについて**
>
> - **CLIENT_ID（OAuth 2.0クライアントID）**: 公開して問題ない。Webアプリ用OAuthクライアントIDは公開前提の設計で、セキュリティは「承認済みリダイレクトURI」の設定で担保されている。
> - **API_KEY**: HTTPリファラー制限を設定すれば公開して問題ない。制限なしのまま公開するとクォータを第三者に消費されるリスクがあるため、必ず制限を入れること。

### 6. クライアントIDとAPIキーをHTMLに設定

`google-sheets-poc.html` の以下の行を書き換える:

```js
const CLIENT_ID = 'YOUR_CLIENT_ID_HERE'; // ← OAuth2クライアントIDを貼る
const API_KEY   = 'YOUR_API_KEY_HERE';   // ← APIキーを貼る（リファラー制限設定後）
```

### 7. ローカルで確認

```
python3 -m http.server 8080
```

http://localhost:8080/google-sheets-poc.html を開く

### 8. GitHub Pagesで公開（任意）

Settings → Pages → Source: `main` ブランチ → Save

公開URL: `https://[ユーザー名].github.io/gdrive-poc/google-sheets-poc.html`

---

## 技術メモ

- OAuth2トークンの有効期限は1時間。期限切れ後は再ログインが必要（本番ではリフレッシュ処理を追加する）。
- 「Sheetsにエクスポート」は初回のみ新規作成。2回目以降は同じスプレッドシートに上書きするか、新規作成するかを選択できる（IDを localStorage に保持）。
- PoCのOAuthスコープは `drive`（広め）。本番組み込み時は `drive.file` に絞ることを推奨。

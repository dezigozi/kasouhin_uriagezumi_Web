# GitHub と Cloudflare Pages へのデプロイ

対象リポジトリ例: [dezigozi/kasouhin_uriagezumi_Web](https://github.com/dezigozi/kasouhin_uriagezumi_Web)

## 前提（重要）

- **Cloudflare Pages は静的サイト用**です。Excel を読む **Node API（`server.js`）は Pages 上では動きません**（ファイル共有・Excel 復号が必要なため）。
- 構成は次の二段です。
  1. **Pages**: フロント（`build/`）
  2. **別ホスト**: API（Docker や社内サーバー、Cloudflare Tunnel など）

## 1. GitHub にプッシュ

このフォルダ（`lease-report-app`）をリポジトリのルートにする想定です。

```bash
cd lease-report-app
git init
git add .
git commit -m "Initial commit: lease report web + deploy config"
git branch -M main
git remote add origin https://github.com/dezigozi/kasouhin_uriagezumi_Web.git
git push -u origin main
```

すでに上位フォルダで Git 管理している場合は、サブフォルダだけを別リポジトリにしたいときは [git subtree](https://git-scm.com/book/en/v2/Git-Tools-Subtree-Merging) や新規クローンへファイルをコピーする方法を使ってください。

## 2. API サーバーを用意

Excel 共有フォルダに **API を動かすマシンから** アクセスできる必要があります。

例: Docker（本リポジトリの `Dockerfile`）

```bash
docker build -t lease-report-api .
docker run -d -p 3001:3001 \
  -e LEASE_REPORT_CORS_ORIGIN=https://<あなたの>.pages.dev \
  lease-report-api
```

- ヘルスチェック: `GET /api/health`
- 本番で API トークンを使う場合: コンテナに `LEASE_REPORT_API_TOKEN` を渡し、フロント側で `public/index.html` などに  
  `window.__LEASE_REPORT_API_TOKEN__ = '...'` を置くか、後述のビルド後スクリプトで注入してください。

## 3. Cloudflare の設定（Pages と Workers のどちらか）

このリポジトリは **静的フロント（`build/`）** なので、Cloudflare では次のどちらかです。

### A. Cloudflare **Pages**（推奨・シンプル）

1. [Cloudflare Dashboard](https://dash.cloudflare.com/) → **Workers & Pages** → **Create** → **Pages**（※「Workers」ではなく **Pages**）→ **Connect to Git**。
2. ビルド設定:

| 項目 | 値 |
|------|-----|
| Build command | `npm ci && npm run build` |
| Build output directory | `build` |
| **Deploy command** | **空欄** |
| Root directory | `/` |

### B. いまのように **Workers の Git 連携**（Build 画面に Deploy command があるタイプ）

ダッシュボードの **Runtime → Build** で次を確認します。

| 項目 | 値 |
|------|-----|
| Build command | `npm run build`（または `npm ci && npm run build`） |
| Deploy command | `npx wrangler deploy`（そのままで可） |
| Root directory | `/` |

リポジトリの **`wrangler.toml`** で **`name`** を Workers プロジェクトの **Name**（例: `kasouhinuriagezumiweb`）と **完全一致**させ、`[assets]` で **`./build`** を指定します（このリポジトリに同梱済み）。SPA のため `not_found_handling = "single-page-application"` を付けています。詳細は [Workers Static Assets / SPA](https://developers.cloudflare.com/workers/static-assets/routing/single-page-application) を参照。

**Pages 用の `wrangler pages deploy`** を CLI で使う例: `npx wrangler pages deploy build --project-name=<Pages のプロジェクト名>`（[Direct Upload](https://developers.cloudflare.com/pages/get-started/direct-upload/)）。

3. **Variables and secrets**（ダッシュボードの **Settings** 付近）で **ビルド時**に参照する変数を追加します。Cloudflare の **Workers** 連携では「Environment variables」ではなく **Variables and secrets** に出ることがあります。

**Production** に少なくとも次を設定:

| Name | 例 |
|------|-----|
| `LEASE_REPORT_API_BASE` | `https://api.example.com`（末尾スラッシュなし。API のオリジンだけ） |

ビルド時に `webpack` がこの値をバンドルに埋め込み、ブラウザは `https://api.example.com/api/...` にリクエストします。

4. API 側で CORS を絞る場合、API の環境変数 `LEASE_REPORT_CORS_ORIGIN` に **Pages の URL** を設定します（プレビュー用に複数ある場合はカンマ区切り）。

   ```
   LEASE_REPORT_CORS_ORIGIN=https://kasouhin-uriagezumi-web.pages.dev,https://main.kasouhin-uriagezumi-web.pages.dev
   ```

## 4. wrangler.toml

- **Workers + `npx wrangler deploy`**: リポジトリの `wrangler.toml` がそのまま使われます（`name` と `[assets].directory = "./build"`）。
- **Pages**: **Deploy command は空**。`wrangler.toml` は必須ではありません（参考用）。

## 5. 社内ネットワークだけで見せる場合

API を社内 PC で動かし、[Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/) で `https://report.example.com` のように公開する方法もあります。その場合もフロントは Pages、API は Tunnel の先のオリジン、という組み合わせが可能です。

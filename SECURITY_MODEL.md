# 脅威モデル: xxxxxtech/juice-shop

最終更新: 2026-11-27 / 対象コミット: `c9f99228`

## 1. システム概要と信頼境界

OWASP Juice Shop（Express + Angular の EC サイト）。上流では意図的に脆弱なトレーニング用アプリだが、**ユーザーの回答により、本デプロイでは課題（`data/static/challenges.yml`）の脆弱性も含めすべて実際のリスクとして扱う**。

- デプロイ（ユーザー回答: `infrastructure/terraform/` と `terraform/` の両方を実デプロイに使用、インターネット公開、ALB 前段に WAF・認証プロキシなし）:  AWS ECS Fargate + ALB（HTTP 80 / HTTPS 443、`0.0.0.0/0` から到達可）、パブリックサブネット、`assign_public_ip = true`、SQLite を EFS に永続化、CloudWatch Logs。コンテナは `Dockerfile`（distroless、UID 65532）または `infrastructure/Dockerfile`。
- 信頼境界:
  1. インターネット ⇔ ALB/アプリ（未認証の誰でも到達可）
  2. 未認証 ⇔ 認証済みユーザー（JWT, `lib/insecurity.ts`）
  3. ロール間: customer / deluxe / accounting / admin（`security.roles`）
  4. ユーザー間（他人のバスケット・住所・カード・注文）
  5. アプリプロセス ⇔ ホスト/コンテナ・ファイルシステム・AWS 基盤（ECS タスクロール、EFS、VPC からの外向き通信）
  6. Terraform 運用者/CI ⇔ AWS アカウント

## 2. コンポーネント

| 名前 | 対象コード |
|---|---|
| Web Client | `frontend/` |
| API Server | `server.ts`, `app.ts`, `routes/`（下記以外）, finale-rest 自動生成 `/api/*` |
| Auth | `lib/insecurity.ts`, `routes/login.ts`, `routes/2fa.ts`, `routes/passkey.ts`, `routes/resetPassword.ts`, `routes/changePassword.ts`, `routes/currentUser.ts`, `routes/securityQuestion.ts` |
| Files & Uploads | `routes/fileServer.ts`, `routes/keyServer.ts`, `routes/logfileServer.ts`, `routes/quarantineServer.ts`, `routes/fileUpload.ts`, `routes/profileImage*Upload.ts`, `routes/memory.ts`, `lib/xml.ts`, `ftp/`, `encryptionkeys/`, `uploads/` |
| Data Stores | `models/`, `data/`（SQLite/Sequelize, MarsDB）, `data/static/users.yml` |
| Challenge Engine | `lib/challengeUtils.ts`, `lib/antiCheat.ts`, `lib/codingChallenges.ts`, `routes/verify.ts`, `routes/vulnCode*.ts`, `routes/continueCode.ts`, `routes/restoreProgress.ts`, `data/static/challenges.yml`, `data/static/codefixes/` |
| Infrastructure | `infrastructure/`, `terraform/`, `Dockerfile`, `docker-compose.test.yml`, `vagrant/`, `.github/workflows/` |

## 3. エントリポイントと到達可能者

インターネット上の誰でも（未認証）:
- Angular SPA と静的ファイル、`/api-docs`（Swagger）、`/metrics`（Prometheus）
- ディレクトリ一覧: `/ftp`, `/encryptionkeys`, `/support/logs`, `/infrastructure`, `/.well-known`（`serve-index`）
- `/rest/user/login`, `/api/Users`（登録 POST）, `/rest/user/reset-password`, `/rest/products/search`, `/rest/track-order/:id`, `/redirect`, `/file-upload`, `/profile/image/*`, `/rest/chatbot`, `/rest/web3/*`, WebSocket（`lib/startup/registerWebsocketEvents.ts`）
- finale-rest による `/api/<Model>` CRUD（`server.ts` の `security.isAuthorized()` / `denyAll()` / `appendUserId()` で個別に制限）

公開ユーザー登録があるため、**到達できる者は誰でも customer アカウントを取得できる**（コードから判明、公開インターネット＝ユーザー回答）。

認証済み: `/rest/basket/*`, `/api/BasketItems`, `/api/Cards`, `/api/Addresss`, `/rest/order-history`, `/rest/user/data-export`, `/rest/2fa/*`, `/rest/passkey/*`, `/b2b/v2/orders`（`routes/b2bOrder.ts`）, `/rest/memories` など。
accounting: `/api/Quantitys/:id`（`isAccounting()`）。admin: 管理画面 `/#/administration`（フロント側のみのルートガード）。

## 4. 資産とデータの重要度

| 資産 | 重要度 |
|---|---|
| 顧客 PII・パスワードハッシュ（MD5）・TOTP 秘密・セキュリティ回答（`models/user.ts`, `securityAnswer.ts`）。ユーザー回答: 本番は**実在の顧客データを保持** | 致命的 |
| 決済カード（`models/card.ts`、ユーザー回答: 実カード情報を保持）、ウォレット残高、注文・クーポン | 致命的 |
| JWT 署名鍵・HMAC 鍵・TLS 秘密鍵（`lib/insecurity.ts`, `infrastructure/terraform/networking.tf` に平文で記載） | 致命的 |
| ホスト/コンテナでのコード実行、EFS 上の DB、AWS 資格情報（ECS ロール）、VPC 内部 | 致命的 |
| アクセスログ（`logs/`）、バックアップ・機密文書（`ftp/`） | 中〜高 |
| 課題進捗・スコア（`continueCode`） | 低 |

## 5. 想定する攻撃者（ユーザー回答に基づく）

- **インターネット上の未認証攻撃者**（公開デプロイ＝ユーザー回答）。
- **自己登録した customer**（公開登録があり誰でも取得可能）。他ユーザー・上位ロールへの昇格を狙う。
- **乗っ取った/偽造したトークンを持つ攻撃者**（鍵がリポジトリ内に平文）。
- **リポジトリ閲覧者**: 公開リポジトリの秘密情報を流用し、本番（Terraform で同じ鍵を使用）を攻撃。
- ALB 前段に WAF・認証プロキシがないため（ユーザー回答）、アプリの脆弱性はインターネットから直接到達可能。
- ユーザー回答により、課題として意図された脆弱性も除外しない。除外・受容リスクとされた攻撃者はいない。

## 6. 既存の対策

- JWT RS256（`isAuthorized()` = express-jwt、`denyAll()` はランダム秘密で全拒否）、`appendUserId()` で所有者 ID を付与
- `helmet.noSniff()` / `frameguard()`、`express-rate-limit`（パスワードリセット・2FA）
- `sanitize-html`（`sanitizeSecure`）、`sanitize-filename`、リダイレクト許可リスト（`isRedirectAllowed`）
- アップロードサイズ制限（multer 200KB）、`notevil` + `vm` タイムアウト（B2B 注文）、XML/YAML パースのタイムアウト（`lib/xml.ts`）
- distroless・非 root コンテナ、ECS SG はアプリポートを ALB からのみ許可
- `challenges.safetyMode`（`config/default.yml`、Docker/Heroku で危険な課題を無効化）

## 7. セキュリティ不変条件

- **認証・セッション（CWE-287/347/798）**: すべてのトークンは `security.isAuthorized()` / `jws.verify` で `encryptionkeys/jwt.pub` に対して RS256 で検証し、`alg` を固定する。署名鍵・HMAC 鍵はコードやリポジトリに置かず、本番ではシークレットストアから注入する。ギャップ: `lib/insecurity.ts` に秘密鍵と HMAC 鍵がハードコード、`/encryptionkeys` が公開、パスワードは MD5（`security.hash`）。
- **認可・IDOR・テナント分離（CWE-639/285/269）**: 他ユーザーのリソース（Basket, BasketItems, Cards, Addresss, Orders, PrivacyRequests, Feedbacks, Reviews）へのアクセスは、トークンの `data.id` と所有者 ID を DB クエリで照合する（`appendUserId()` + ルート内照合）。ロール（admin/accounting/deluxe）はサーバー側でトークンから判定し、リクエストボディの `role` を信用しない。finale-rest の自動 CRUD は `server.ts` で明示的に保護しないと公開される（エスケープハッチ）。ギャップ: 登録時の `role` 指定、`/rest/basket/:id`、`PUT /api/Products/:id` のコメントアウト等。
- **SQL / NoSQL インジェクション（CWE-89/943）**: Sequelize のモデル API またはバインドパラメータ（`replacements`）を使う。`models.sequelize.query` へのテンプレート文字列連結（`routes/login.ts`, `routes/search.ts`）と MarsDB の `$where`/文字列連結（`routes/trackOrder.ts`, `routes/showProductReviews.ts`, `routes/updateProductReviews.ts`）は禁止。
- **コード実行・サンドボックス（CWE-94/95/1321）**: ユーザー入力を `eval`/`vm`/`notevil`（`routes/b2bOrder.ts`）、テンプレート（`views/*.pug` / `routes/userProfile.ts`）、`js-yaml` の非安全ロードに渡さない。`vm` はセキュリティ境界ではない。
- **XXE・パーサ（CWE-611/776）**: `lib/xml.ts` は外部エンティティ解決（`noent`）とエンティティ展開を無効にし、YAML は安全なスキーマで解析する。
- **SSRF・外向き通信（CWE-918）**: `routes/profileImageUrlUpload.ts` の `fetch(url)` はスキーム・ホストを許可リストで制限し、内部アドレス（169.254.169.254 / ECS メタデータ 169.254.170.2、VPC 10.0.0.0/16）を拒否する。ECS タスクは外向き `0.0.0.0/0` が開いているため影響が大きい。
- **XSS・CSRF（CWE-79/352）**: Angular のデフォルトエスケープに依存し、`bypassSecurityTrustHtml` / `[innerHTML]` を使う箇所ではサーバー側で `sanitizeSecure` を通す。`sanitizeLegacy` は不十分。CORS は `cors()` で全オリジン許可のため、Cookie 認証の状態変更 API は CSRF 対策が必要。
- **パストラバーサル・ファイル処理（CWE-22/434/158）**: `/ftp`, `/encryptionkeys`, `/support/logs` の配信は `path.resolve` 後に基準ディレクトリ配下を確認し、`cutOffPoisonNullByte` の後に拡張子を検査する。ZIP 展開（`routes/fileUpload.ts`）はエントリ名を正規化し zip-slip を防ぐ。ディレクトリ一覧（`serve-index`）は機密ディレクトリで無効にする。
- **オープンリダイレクト（CWE-601）**: `/redirect` は `isRedirectAllowed` で完全一致を判定する（部分一致は不可）。
- **情報漏えい（CWE-200/532）**: `/metrics`, `/api-docs`, `/support/logs`, エラーページ（`errorhandler`）のスタックトレースは本番で未認証に公開しない。
- **IaC・サプライチェーン（CWE-798/732）**: Terraform に秘密鍵を直書きしない（`aws_iam_server_certificate.private_key` は外部シークレットから）。ALB の HTTP 80 は HTTPS にリダイレクトする。`local-exec` による `docker build/push` は信頼できる CI でのみ実行する。依存パッケージ（`package.json`）の既知脆弱性を追跡する。

## 8. 対象外・受容リスク（ユーザー回答）

- なし。ユーザーは課題の脆弱性も含めすべて実リスクとして扱うと回答（2026-11-27）。

## 9. 未解決の質問

- 本番で使用している `config/*.yml`（`NODE_ENV`）と `safetyMode` の値。

（回答済み 2026-11-27: Terraform は両ディレクトリとも本番使用、実データ保持、WAF・認証プロキシなし）

# 脅威モデル: xxxxxtech/juice-shop

> 出典: リポジトリ直下 `threat-model.json`（OWASP Threat Dragon 形式、upstream 作成）、`SECURITY.md`、`AGENTS.md`、`infrastructure/README.md` とコード調査。このフォークは upstream (juice-shop/juice-shop) の最新コミットと同一内容に見える。

## 1. システム概要と信頼境界
OWASP Juice Shop は **意図的に脆弱な** EC サイト型トレーニングアプリ（`SECURITY.md`、`threat-model.json` の description）。OWASP Top 10 全般を含む約 118 件のハッキングチャレンジ（`data/static/challenges.yml`）と、`vuln-code-snippet` コメントで脆弱行をマークしたコーディングチャレンジ（`data/static/codefixes/`）を持つ。

- 信頼境界 1: ブラウザ ⇔ Express サーバー（`server.ts`、ポート 3000）。認証は RS256 JWT（`lib/insecurity.ts` の `authorize` / `isAuthorized`）。
- 信頼境界 2: Express ⇔ データストア（SQLite `data/juiceshop.sqlite` via Sequelize、`models/`。MarsDB インメモリ `data/mongodb.ts`）。
- 信頼境界 3: アプリプロセス ⇔ ホスト／コンテナ（ファイルシステム、`vm` サンドボックス、外部 URL 取得）。
- upstream はルートの `Dockerfile` を推奨し `infrastructure/` を「意図的に不安全・デプロイ禁止」としている（`infrastructure/README.md`）が、**ユーザー回答によりこのフォークは `infrastructure/terraform` を実デプロイに使っている**。

## 2. コンポーネント
| 名前 | 範囲 |
|---|---|
| Frontend | `frontend/src/`（Angular SPA） |
| API | `server.ts`、`routes/`、`views/`（REST / Sequelize 自動 API `/api/*`、`/rest/*`、`/b2b/v2`、Socket.IO） |
| Auth | `lib/insecurity.ts`、`routes/login.ts`、`routes/2fa.ts`、`routes/passkey.ts`、`routes/resetPassword.ts`、`routes/changePassword.ts` |
| Data | `models/`、`data/`（シードデータ、`data/static/`、MarsDB） |
| Challenge Engine | `lib/challengeUtils.ts`、`lib/antiCheat.ts`、`lib/codingChallenges.ts`、`routes/vulnCodeSnippet.ts`、`routes/vulnCodeFixes.ts`、`routes/continueCode.ts`、`rsn/` |
| Infrastructure | `Dockerfile`、`infrastructure/`、`terraform/`、`vagrant/`、`.github/workflows/` |

## 3. エントリーポイントと到達者
- 未認証で到達可能: SPA、`/rest/user/login`、`/api/Users`(POST 登録)、`/rest/products/search`、`/ftp`、`/encryptionkeys`、`/support/logs`、`/.well-known`、`/api-docs`、`/metrics`、`/file-upload`、`/profile/image/url` 等。
- 要 JWT: `/rest/basket`、`/api/BasketItems`、`/b2b/v2/orders`、`/api/Complaints` 等（`server.ts` の `security.isAuthorized()`）。
- 役割: customer / deluxe / accounting / admin（`lib/insecurity.ts` の `roles`）。アカウントは誰でもセルフ登録可能。

## 4. 資産とデータ感度
- **最重要（ユーザー指定）**: 顧客の個人情報・アカウント、決済情報（カード・ウォレット）。
- ユーザー、パスワードハッシュ、カード、住所、注文（`models/`、SQLite）。現状のシードは架空データ（`data/static/users.yml` 等）だが、実運用前提のため実顧客データ相当として扱う。
- 実際に価値のある資産: アプリを動かすホスト／コンテナとその周辺ネットワーク、CI のシークレット（`.github/workflows/`）。
- JWT 秘密鍵（`lib/insecurity.ts` にハードコード）、`encryptionkeys/`、`ctf.key`: upstream では意図的公開だが、実運用前提では漏えい＝全アカウント偽造に直結する重要資産。
- AWS 環境（`infrastructure/terraform` で実デプロイ: ECS、ALB、EFS、IAM ロール）。

## 5. 攻撃者プロファイル
- 未認証のインターネット上の攻撃者: 公開エンドポイント（§3）を直接攻撃。
- 自己登録ユーザー（customer）: 認可・IDOR・権限昇格を狙う。
- インフラ到達者: SSRF・コンテナ脱出・不安全な Terraform 設定（公開 SG、IAM、EFS）を経由して AWS 環境へ波及。
- CI / サプライチェーン攻撃者: `.github/workflows/`、依存パッケージ。
- **ユーザー回答（2026-11-27）**: このフォークは**実運用またはその土台**として使い、**インターネットに公開**する。したがって攻撃者はインターネット上の誰でも（セルフ登録で customer アカウントを取得可能）。チャレンジとして意図された脆弱性も含め、**すべて本物の脆弱性として扱う**。

## 6. 既存の対策
- `challenges.safetyMode`（`config/default.yml`、`lib/utils.ts`）: Docker/Heroku 等で危険なチャレンジ（RCE 系など `disabledEnv`）を無効化。
- `vm.runInContext` + `notevil` によるサンドボックス（`routes/b2bOrder.ts`、`routes/fileUpload.ts` の YAML）、タイムアウト 2 秒。
- `helmet.noSniff` / `frameguard`、一部ルートの `express-rate-limit`、ファイルアップロードサイズ制限（`checkUploadSize`）。
- Docker イメージは distroless・非 root。

## 7. セキュリティ不変条件
upstream の Juice Shop では多くのクラスで不変条件が意図的に破られている。**ユーザー回答により、このフォークではチャレンジとして意図された破れも本物のギャップとして扱う**。チャレンジ該当性（`data/static/challenges.yml`、`vuln-code-snippet vuln-line <key>` マーカー）は `Challenge` タグの付与にだけ使い、優先度を下げる理由にはしない。

- **認証 (CWE-287/798/347)**: トークンは `security.authorize` で RS256 署名、`security.isAuthorized()`（express-jwt）で検証。秘密鍵ハードコード、`login.ts` の SQLi ログイン、弱い `security.hash`（MD5）はチャレンジ。チャレンジ外で JWT 検証をバイパスする経路はギャップ。
- **認可・IDOR (CWE-639/285)**: `security.appendUserId()` でリクエストにユーザー ID を付与、ルート単位で `isAuthorized` / `denyAll` / `isAccounting`。バスケット IDOR、`PUT /api/Products/:id` 無保護（`changeProductChallenge`）等はチャレンジ。
- **インジェクション (CWE-89/943/94)**: 安全なパターンは Sequelize モデル API / バインドパラメータ。`routes/search.ts`・`routes/login.ts` の文字列連結 `sequelize.query`、MarsDB の `$where`（NoSQL）、`b2bOrder` の `notevil` はチャレンジ。`vm` サンドボックスからホストへ脱出できる経路は想定外。
- **SSRF / 外部取得 (CWE-918)**: `routes/profileImageUrlUpload.ts` がユーザー指定 URL を取得（SSRF チャレンジ）。クラウドメタデータや内部ネットワークへの到達は公開環境では実害。
- **XSS / CSRF (CWE-79/352)**: Angular の既定エスケープが基本。`bypassSecurityTrustHtml` 使用箇所や `helmet.xssFilter` 無効化はチャレンジ。CORS は全許可（`cors()`）。
- **パストラバーサル / ファイル処理 (CWE-22/434/611)**: `/ftp` は `servePublicFiles` で拡張子チェック（Poison Null Byte チャレンジ）、ZIP Slip（`handleZipFileUpload`）、XXE（`lib/xml.ts`、libxml2-wasm）はチャレンジ。アプリディレクトリ外への任意書き込みはホスト影響。
- **シークレット / 暗号 (CWE-321/327)**: `encryptionkeys/`、`ctf.key`、JWT 秘密鍵、`/support/logs` 公開は意図的。CI ワークフローの実シークレット漏えいはギャップ。
- **IaC / サプライチェーン**: `infrastructure/` の不安全な Terraform / Dockerfile / nginx はチャレンジ（`vulnerableDockerImageChallenge` 等）。`.github/workflows/` の `pull_request_target` 等による権限昇格は実害。

## 8. スコープ外・受容リスク
- ユーザーはスコープ外・受容リスクを指定していない。意図的なチャレンジ脆弱性も受容しない（ユーザー回答）。

## 9. 未解決の質問
- 実運用で有効にする設定ファイル（`config/default.yml` 以外か、`safetyMode` の値）

## 10. 優先度付けの方針（ユーザー回答に基づく）
- 用途＝実運用、公開＝インターネット、Terraform＝実使用。よって攻撃者・資産・面を除外する根拠はなく、原則スキャナの重大度をそのまま採用する。
- 「意図的なチャレンジだから」という理由で下げない。
- **最重要資産（ユーザー回答 2026-11-27）**: 顧客の個人情報・アカウント（`Users`、`Addresses`、`PrivacyRequests`、JWT／認証）と決済情報（`Cards`、`Wallets`、`routes/payment.ts`、`routes/wallet.ts`）。スキャナが到達経路を確認し悪用容易と評価した検出で、これらを窃取・改ざん・乗っ取りできるものは一段引き上げ得る。

API リファレンス
OAuth 2.0 認可サーバーの統合に必要な情報をまとめています

エンドポイント
GET
/api/oauth/authorize
認可リクエスト → consent ページへリダイレクト
POST
/api/oauth/token
トークン発行（全グラントタイプ）
GET
/api/oauth/userinfo
OIDC UserInfo — Bearer トークンが必要
POST
/api/oauth/introspect
RFC 7662 トークン検査（クライアント認証必要）
POST
/api/oauth/revoke
RFC 7009 トークン失効（常に 200）
GET
/.well-known/oauth-authorization-server
RFC 8414 メタデータ Discovery
グラントタイプ
authorization_code
推奨
ブラウザ経由でユーザーを認可。PKCE (S256) と組み合わせて使用してください。

code
redirect_uri
code_verifier
client_credentials
M2M
ユーザーを介さないサーバー間通信。client_id と client_secret のみで認証します。

scope
refresh_token
更新
アクセストークンを更新します。使用のたびにリフレッシュトークンがローテーションされます。

refresh_token
標準スコープ
openid
sub（ユーザーID）クレームを含む ID トークンを要求します
profile
name / picture / updated_at を UserInfo で取得できます
email
email / email_verified を UserInfo で取得できます
offline_access
リフレッシュトークンを発行します（authorization_code のみ）
discord_id
Discord ユーザー ID を取得できます。Discord 連携済みの場合のみ値が返ります（未連携は null）
discord_roles
Evex Developers サーバーでのロール一覧を取得できます。discord_id スコープが自動的に付与されます。未連携・非メンバーは [] または null が返ります
トークン仕様
形式
Opaque（不透明）— JWT ではありません
格納
DB には SHA-256 ハッシュのみ保存
アクセストークン TTL
1 時間
リフレッシュトークン TTL
30 日（ローテーション）
送信方法
Authorization: Bearer <token>
プレフィックス
evex_at_ / evex_rt_ / evex_code_
クライアント認証
client_secret_basic
HTTP Basic 認証ヘッダーで送信

Authorization: Basic base64(client_id:client_secret)
client_secret_post
リクエストボディに含めて送信

client_id=...&client_secret=...
PKCE (RFC 7636)
パブリッククライアント（SPA・モバイル）では PKCE S256 を必ず使用してください。

code_verifier — ランダム 43〜128 文字の文字列を生成
code_challenge = BASE64URL(SHA-256(code_verifier))
認可リクエストに code_challenge と code_challenge_method=S256 を付与
トークンリクエストに code_verifier を付与してサーバーが検証
クイック統合例（authorization_code + PKCE）
// 1. 認可リクエスト
GET /api/oauth/authorize
  ?response_type=code
  &client_id=<YOUR_CLIENT_ID>
  &redirect_uri=https://yourapp.com/callback
  &scope=openid profile email offline_access
  &state=<RANDOM_STATE>
  &code_challenge=<BASE64URL_SHA256_VERIFIER>
  &code_challenge_method=S256

// 2. コールバックで code を受け取ったらトークンを交換
POST /api/oauth/token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&code=<CODE>
&redirect_uri=https://yourapp.com/callback
&client_id=<YOUR_CLIENT_ID>
&client_secret=<YOUR_SECRET>
&code_verifier=<VERIFIER>

// 3. レスポンス
{
  "access_token": "evex_at_...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "evex_rt_...",
  "scope": "openid profile email offline_access"
}


OIDC ドキュメント
OpenID Connect 1.0 対応エンドポイントの一覧と利用方法
Discovery エンドポイント
OpenID Connect Discovery 1.0 準拠のメタデータを提供します。

GET /.well-known/openid-configuration
GET /.well-known/oauth-authorization-server
JWKS エンドポイント
ID トークン署名検証用の公開鍵（RS256）を JWK Set 形式で返します。

GET /api/oauth/jwks
ID トークン
openid スコープを含む認可コードフローで トークンエンドポイントから ID トークンが返されます。署名アルゴリズムは RS256 です。

{
"access_token": "...",
"id_token": "eyJ...",
"token_type": "Bearer",
"expires_in": 3600
}
ID トークンのペイロードに含まれるクレーム：sub, iss, aud, iat, exp, nonce (要求時), email (emailスコープ時), name (profileスコープ時)

Discord クレーム
Discord 連携ユーザーの情報を取得するための拡張スコープです。

discord_id スコープ
UserInfo レスポンスに discord_id クレームが追加されます。Discord 未連携の場合は null が返ります。

{
"discord_id": "123456789012345678"
}
discord_roles スコープ
UserInfo レスポンスに discord_roles クレームが追加されます。 Evex Developers サーバーでのロール一覧をオブジェクト配列で返します。 このスコープを要求すると discord_id が自動的に付与されます。

{
"discord_roles": [
{ "id": "111...", "name": "Developer", "color": 5793266, "position": 5 },
{ "id": "222...", "name": "Member", "color": 0, "position": 1 }
]
}
null — Discord 未連携
[] — サーバー未参加、またはボットサービス一時停止中
Cloudflare Access との連携
Cloudflare Access の OIDC プロバイダーとして設定する手順：

Cloudflare Zero Trust ダッシュボード → Settings → Authentication → Add new → OpenID Connect
以下の情報を入力します：
Auth URL: /api/oauth/authorize
Token URL: /api/oauth/token
Certificate URL: /api/oauth/jwks
スコープ：openid, email, profile を設定
Client ID / Client Secret は開発者ページの「OAuthアプリ」タブで作成したアプリの情報を使用
認可コードフロー例
# 1. 認可リクエスト
GET /api/oauth/authorize?
response_type=code&
client_id=YOUR_CLIENT_ID&
redirect_uri=https://your-app/callback&
scope=openid+email+profile&
state=RANDOM_STATE&
nonce=RANDOM_NONCE
# 2. トークン取得
POST /api/oauth/token
grant_type=authorization_code
code=AUTH_CODE
redirect_uri=https://your-app/callback
client_id=YOUR_CLIENT_ID
client_secret=YOUR_CLIENT_SECRET
# Issue #1, #2, #22 統合実装計画

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** リフレッシュトークン機構の導入、Gemini AIパイプラインのStructured Output化、問題報告機能の追加を直列3フェーズで実装する。

**Architecture:** Phase 1でバックエンド/フロントエンドにリフレッシュトークンを追加（アクセス30min/リフレッシュ14d/ローテーション付き）。Phase 2でjlpt-app-scriptsのGemini API呼び出しにStructured Output・system_instruction・負例Few-shotを導入。Phase 3で既存問題のメタデータ一括タグ付けと最小限の問題報告機能を追加。

**Tech Stack:** Rust (Axum 0.8), Next.js 16, Firestore, google-generative-ai-rs 0.3.4, SHA-256, UUID v4

**Design Doc:** `docs/plans/2026-03-18-issues-1-2-22-design.md`

---

## Phase 1: Issue #22 — リフレッシュトークン機構

### Task 1: RefreshTokenモデルの追加 (jlpt-app-backend)

**Files:**
- Create: `jlpt-app-backend/src/models/refresh_token.rs`
- Modify: `jlpt-app-backend/src/models/mod.rs`
- Modify: `jlpt-app-backend/Cargo.toml`

**Step 1: Cargo.tomlにsha2依存を追加**

`jlpt-app-backend/Cargo.toml` の `[dependencies]` に追加:

```toml
sha2 = "0.10"
rand = "0.8"
hex = "0.4"
```

**Step 2: RefreshTokenモデルを作成**

`jlpt-app-backend/src/models/refresh_token.rs`:

```rust
use serde::{Deserialize, Serialize};
use sha2::{Digest, Sha256};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct RefreshToken {
    pub token_hash: String,
    pub user_id: String,
    pub token_family: String,
    pub issued_at: i64,
    pub expires_at: i64,
    pub is_revoked: bool,
    pub revoked_at: Option<i64>,
}

impl RefreshToken {
    pub fn new(raw_token: &str, user_id: String, token_family: String) -> Self {
        let now = chrono::Utc::now().timestamp();
        Self {
            token_hash: Self::hash(raw_token),
            user_id,
            token_family,
            issued_at: now,
            expires_at: now + 14 * 24 * 60 * 60, // 14 days
            is_revoked: false,
            revoked_at: None,
        }
    }

    pub fn hash(raw_token: &str) -> String {
        let mut hasher = Sha256::new();
        hasher.update(raw_token.as_bytes());
        hex::encode(hasher.finalize())
    }

    pub fn is_expired(&self) -> bool {
        chrono::Utc::now().timestamp() > self.expires_at
    }

    pub fn generate_raw_token() -> String {
        uuid::Uuid::new_v4().to_string()
    }
}
```

**Step 3: mod.rsにエクスポート追加**

`jlpt-app-backend/src/models/mod.rs` に追加:

```rust
pub mod refresh_token;
```

**Step 4: ビルド確認**

Run: `cd jlpt-app-backend && cargo check`
Expected: PASS (新構造体のみ、依存なし)

**Step 5: コミット**

```bash
git add jlpt-app-backend/Cargo.toml jlpt-app-backend/src/models/refresh_token.rs jlpt-app-backend/src/models/mod.rs
git commit -m "feat(backend): RefreshTokenモデル追加（SHA-256ハッシュ、token_family、14日有効期限）"
```

---

### Task 2: Claimsにjtiクレーム追加 + 有効期限30分化 (jlpt-app-backend)

**Files:**
- Modify: `jlpt-app-backend/src/models/claim.rs`

**Step 1: Claims構造体にjtiフィールド追加**

`jlpt-app-backend/src/models/claim.rs` Claims構造体を変更:

```rust
// 変更前
#[derive(Debug, Serialize, Deserialize)]
pub struct Claims {
    pub user_id: String,
    pub email: String,
    pub exp: i64,
    pub role: Option<String>,
}

// 変更後
#[derive(Debug, Serialize, Deserialize)]
pub struct Claims {
    pub user_id: String,
    pub email: String,
    pub exp: i64,
    pub role: Option<String>,
    pub jti: String,
}
```

**Step 2: Claims::new()の有効期限を30分に変更、jti追加**

```rust
// 変更前
impl Claims {
    pub fn new(user_id: String, email: String, role: Option<String>) -> Self {
        let after24h = chrono::Utc::now().timestamp() + 60 * 60 * 24;
        Self {
            user_id,
            email,
            exp: after24h,
            role,
        }
    }

// 変更後
impl Claims {
    pub fn new(user_id: String, email: String, role: Option<String>) -> Self {
        let after30min = chrono::Utc::now().timestamp() + 30 * 60;
        Self {
            user_id,
            email,
            exp: after30min,
            role,
            jti: uuid::Uuid::new_v4().to_string(),
        }
    }
```

**Step 3: ビルド確認**

Run: `cd jlpt-app-backend && cargo check`
Expected: PASS

**Step 4: コミット**

```bash
git add jlpt-app-backend/src/models/claim.rs
git commit -m "feat(backend): Claimsにjtiクレーム追加、アクセストークン有効期限を30分に変更"
```

---

### Task 3: トークンユーティリティの作成 (jlpt-app-backend)

**Files:**
- Create: `jlpt-app-backend/src/common/token.rs`
- Modify: `jlpt-app-backend/src/common/mod.rs`

**Step 1: token.rsを作成**

`jlpt-app-backend/src/common/token.rs`:

```rust
use axum_extra::extract::cookie::{Cookie, SameSite};

pub fn build_refresh_cookie(raw_token: &str) -> Cookie<'static> {
    let is_production = std::env::var("FRONTEND_URL")
        .map(|u| u.starts_with("https://"))
        .unwrap_or(false);

    let mut cookie = Cookie::build(("refresh_token", raw_token.to_string()))
        .http_only(true)
        .same_site(SameSite::Lax)
        .path("/api/auth/refresh")
        .max_age(time::Duration::days(14));

    if is_production {
        cookie = cookie.secure(true);
    }

    cookie.build()
}

pub fn build_clear_refresh_cookie() -> Cookie<'static> {
    Cookie::build(("refresh_token", ""))
        .http_only(true)
        .same_site(SameSite::Lax)
        .path("/api/auth/refresh")
        .max_age(time::Duration::seconds(0))
        .build()
}
```

**Step 2: mod.rsにエクスポート追加**

`jlpt-app-backend/src/common/mod.rs` に追加:

```rust
pub mod token;
```

**Step 3: ビルド確認**

Run: `cd jlpt-app-backend && cargo check`
Expected: PASS

**Step 4: コミット**

```bash
git add jlpt-app-backend/src/common/token.rs jlpt-app-backend/src/common/mod.rs
git commit -m "feat(backend): リフレッシュトークンCookie構築ユーティリティ追加"
```

---

### Task 4: signinハンドラにリフレッシュトークン発行を追加 (jlpt-app-backend)

**Files:**
- Modify: `jlpt-app-backend/src/api/user.rs`

**Step 1: use文にRefreshToken関連を追加**

`jlpt-app-backend/src/api/user.rs` のimport部分に追加:

```rust
use crate::models::refresh_token::RefreshToken;
use crate::common::token::{build_refresh_cookie, build_clear_refresh_cookie};
```

**Step 2: build_auth_cookieの有効期限を30分に変更**

```rust
// 変更前
    let mut cookie = Cookie::build(("access_token", token.to_string()))
        .http_only(true)
        .same_site(SameSite::Lax)
        .path("/api")
        .max_age(time::Duration::hours(24));

// 変更後
    let mut cookie = Cookie::build(("access_token", token.to_string()))
        .http_only(true)
        .same_site(SameSite::Lax)
        .path("/api")
        .max_age(time::Duration::minutes(30));
```

**Step 3: signinハンドラにリフレッシュトークン生成・保存を追加**

signin関数内、JWTのCookie設定後に追加:

```rust
    // Set httpOnly Cookie (access token)
    let cookie = build_auth_cookie(&to_token);
    let jar = jar.add(cookie);

    // Generate refresh token
    let raw_refresh = RefreshToken::generate_raw_token();
    let token_family = uuid::Uuid::new_v4().to_string();
    let refresh_record = RefreshToken::new(&raw_refresh, effective_user_id.clone(), token_family);

    // Save to Firestore
    let doc_id = refresh_record.token_hash.clone();
    if let Err(e) = db
        .create::<RefreshToken>("refresh_tokens", &doc_id, refresh_record)
        .await
    {
        error!("リフレッシュトークン保存失敗: {:?}", e);
        return (
            jar,
            response_handler(
                StatusCode::INTERNAL_SERVER_ERROR,
                "error".to_string(),
                None,
                Some("認証処理に失敗しました".to_string()),
            ),
        );
    }

    // Set refresh token cookie
    let refresh_cookie = build_refresh_cookie(&raw_refresh);
    let jar = jar.add(refresh_cookie);
```

**Step 4: ビルド確認**

Run: `cd jlpt-app-backend && cargo check`
Expected: PASS

**Step 5: コミット**

```bash
git add jlpt-app-backend/src/api/user.rs
git commit -m "feat(backend): signinでリフレッシュトークン発行・Firestore保存・Cookie設定"
```

---

### Task 5: POST /api/auth/refresh エンドポイント作成 (jlpt-app-backend)

**Files:**
- Create: `jlpt-app-backend/src/api/auth.rs`
- Modify: `jlpt-app-backend/src/api/mod.rs`
- Modify: `jlpt-app-backend/src/main.rs`

**Step 1: auth.rsを作成**

`jlpt-app-backend/src/api/auth.rs`:

```rust
use std::sync::Arc;

use axum::{extract::State, response::IntoResponse};
use axum_extra::extract::CookieJar;
use http::StatusCode;
use log::error;
use serde_json::json;

use crate::api::utils::response_handler;
use crate::common::database::Database;
use crate::common::token::{build_clear_refresh_cookie, build_refresh_cookie};
use crate::models::claim::Claims;
use crate::models::refresh_token::RefreshToken;

use super::user::build_auth_cookie;

pub async fn refresh_token(
    State(db): State<Arc<Database>>,
    jar: CookieJar,
) -> impl IntoResponse {
    // 1. Cookieからrefresh_tokenを取得
    let raw_token = match jar.get("refresh_token") {
        Some(cookie) => cookie.value().to_string(),
        None => {
            return (
                jar,
                response_handler(
                    StatusCode::UNAUTHORIZED,
                    "error".to_string(),
                    None,
                    Some("リフレッシュトークンがありません".to_string()),
                ),
            );
        }
    };

    // 2. SHA-256ハッシュでFirestore検索
    let token_hash = RefreshToken::hash(&raw_token);
    let stored = match db
        .read::<RefreshToken>("refresh_tokens", &token_hash)
        .await
    {
        Ok(Some(record)) => record,
        _ => {
            let jar = jar.add(build_clear_refresh_cookie());
            return (
                jar,
                response_handler(
                    StatusCode::UNAUTHORIZED,
                    "error".to_string(),
                    None,
                    Some("無効なリフレッシュトークンです".to_string()),
                ),
            );
        }
    };

    // 3. 再利用検出（is_revokedなら漏洩）→ family全無効化
    if stored.is_revoked {
        // 漏洩検知: このfamilyの全トークンを無効化
        if let Err(e) = revoke_token_family(&db, &stored.token_family).await {
            error!("token family無効化失敗: {:?}", e);
        }
        let jar = jar.add(build_clear_refresh_cookie());
        return (
            jar,
            response_handler(
                StatusCode::UNAUTHORIZED,
                "error".to_string(),
                None,
                Some("セキュリティ上の理由でセッションが無効化されました。再ログインしてください".to_string()),
            ),
        );
    }

    // 4. 有効期限チェック
    if stored.is_expired() {
        let jar = jar.add(build_clear_refresh_cookie());
        return (
            jar,
            response_handler(
                StatusCode::UNAUTHORIZED,
                "error".to_string(),
                None,
                Some("リフレッシュトークンの有効期限が切れています".to_string()),
            ),
        );
    }

    // 5. 旧トークンを失効
    let mut revoked = stored.clone();
    revoked.is_revoked = true;
    revoked.revoked_at = Some(chrono::Utc::now().timestamp());
    if let Err(e) = db
        .update::<RefreshToken>("refresh_tokens", &token_hash, revoked)
        .await
    {
        error!("旧トークン失効失敗: {:?}", e);
        return (
            jar,
            response_handler(
                StatusCode::INTERNAL_SERVER_ERROR,
                "error".to_string(),
                None,
                Some("トークン更新に失敗しました".to_string()),
            ),
        );
    }

    // 6. ユーザー情報を取得してロール判定
    let role = {
        let admin_emails = std::env::var("ADMIN_EMAILS").unwrap_or_default();
        // user_idからemailを取得するため、Firestoreからユーザーを探す
        // signinで保存したemailはClaimsから取れないため、refresh_tokenにuser_idがある
        // ここでは簡易的にadmin判定を省略し、新JWTにはroleなしで発行
        // → 次のauth/meでrole含めて返すので問題なし
        // ただしadmin判定が必要な場合はユーザーDBから取得
        None::<String> // 簡易実装: roleはauth/meで再判定
    };

    // user情報取得（emailとrole）
    // refresh_tokenのuser_idからユーザーを検索
    let user_email = match find_user_email_by_id(&db, &stored.user_id).await {
        Some(email) => email,
        None => {
            let jar = jar.add(build_clear_refresh_cookie());
            return (
                jar,
                response_handler(
                    StatusCode::UNAUTHORIZED,
                    "error".to_string(),
                    None,
                    Some("ユーザーが見つかりません".to_string()),
                ),
            );
        }
    };

    let role = {
        let admin_emails = std::env::var("ADMIN_EMAILS").unwrap_or_default();
        let is_admin = admin_emails
            .split(',')
            .map(|e| e.trim())
            .any(|e| e == user_email);
        if is_admin {
            Some("admin".to_string())
        } else {
            None
        }
    };

    // 7. 新アクセストークン発行
    let claims = Claims::new(stored.user_id.clone(), user_email, role);
    let access_token = match claims.to_token() {
        Ok(token) => token,
        Err(e) => {
            error!("アクセストークン生成失敗: {:?}", e);
            return (
                jar,
                response_handler(
                    StatusCode::INTERNAL_SERVER_ERROR,
                    "error".to_string(),
                    None,
                    Some("トークン更新に失敗しました".to_string()),
                ),
            );
        }
    };

    // 8. 新リフレッシュトークン発行（同じfamily）
    let new_raw = RefreshToken::generate_raw_token();
    let new_record = RefreshToken::new(&new_raw, stored.user_id.clone(), stored.token_family);
    let new_hash = new_record.token_hash.clone();
    if let Err(e) = db
        .create::<RefreshToken>("refresh_tokens", &new_hash, new_record)
        .await
    {
        error!("新リフレッシュトークン保存失敗: {:?}", e);
        return (
            jar,
            response_handler(
                StatusCode::INTERNAL_SERVER_ERROR,
                "error".to_string(),
                None,
                Some("トークン更新に失敗しました".to_string()),
            ),
        );
    }

    // 9. Cookie設定
    let jar = jar.add(build_auth_cookie(&access_token));
    let jar = jar.add(build_refresh_cookie(&new_raw));

    (
        jar,
        response_handler(
            StatusCode::OK,
            "success".to_string(),
            Some(json!({"refreshed": true})),
            None,
        ),
    )
}

async fn revoke_token_family(db: &Database, family: &str) -> Result<(), String> {
    // Firestoreでtoken_familyが一致するドキュメントを検索して全て失効
    use firestore::*;
    let docs: Vec<RefreshToken> = db
        .client
        .fluent()
        .select()
        .from("refresh_tokens")
        .filter(|q| {
            q.for_all([
                q.field(path!(RefreshToken::token_family)).eq(family),
                q.field(path!(RefreshToken::is_revoked)).eq(false),
            ])
        })
        .obj()
        .query()
        .await
        .map_err(|e| format!("family検索失敗: {}", e))?;

    let now = chrono::Utc::now().timestamp();
    for mut doc in docs {
        doc.is_revoked = true;
        doc.revoked_at = Some(now);
        let _ = db
            .update::<RefreshToken>("refresh_tokens", &doc.token_hash, doc)
            .await;
    }
    Ok(())
}

async fn find_user_email_by_id(db: &Database, user_id: &str) -> Option<String> {
    // usersコレクションからuser_idで検索
    use firestore::*;
    let users: Vec<crate::models::user::User> = db
        .client
        .fluent()
        .select()
        .from("users")
        .filter(|q| {
            q.for_any([
                q.field(path!(crate::models::user::User::id)).eq(user_id),
                q.field(path!(crate::models::user::User::user_id)).eq(user_id),
            ])
        })
        .obj()
        .query()
        .await
        .ok()?;

    users.first().map(|u| u.email.clone())
}
```

**Step 2: api/mod.rsにエクスポート追加**

```rust
pub mod auth;
```

**Step 3: build_auth_cookieをpubに変更**

`jlpt-app-backend/src/api/user.rs` で:

```rust
// 変更前
fn build_auth_cookie(token: &str) -> Cookie<'static> {

// 変更後
pub fn build_auth_cookie(token: &str) -> Cookie<'static> {
```

**Step 4: main.rsにルート追加**

`jlpt-app-backend/src/main.rs` のルート定義部分に追加:

```rust
    .route("/api/auth/refresh", post(api::auth::refresh_token))
```

**Step 5: ビルド確認**

Run: `cd jlpt-app-backend && cargo check`
Expected: PASS

**Step 6: コミット**

```bash
git add jlpt-app-backend/src/api/auth.rs jlpt-app-backend/src/api/mod.rs jlpt-app-backend/src/api/user.rs jlpt-app-backend/src/main.rs
git commit -m "feat(backend): POST /api/auth/refresh エンドポイント（ローテーション+漏洩検知）"
```

---

### Task 6: logoutハンドラの改修（全トークン無効化） (jlpt-app-backend)

**Files:**
- Modify: `jlpt-app-backend/src/api/user.rs`

**Step 1: auth_logoutを改修**

```rust
// 変更前
pub async fn auth_logout(jar: CookieJar) -> impl IntoResponse {
    let cookie = Cookie::build(("access_token", ""))
        .http_only(true)
        .same_site(SameSite::Lax)
        .path("/api")
        .max_age(time::Duration::seconds(0))
        .build();
    let jar = jar.add(cookie);
    (
        jar,
        response_handler(StatusCode::OK, "success".to_string(), None, None),
    )
}

// 変更後
pub async fn auth_logout(
    State(db): State<Arc<crate::common::database::Database>>,
    claims: Option<Claims>,
    jar: CookieJar,
) -> impl IntoResponse {
    // ユーザーの全リフレッシュトークンを無効化
    if let Some(claims) = claims {
        if let Err(e) = revoke_all_user_tokens(&db, &claims.user_id).await {
            log::error!("全トークン無効化失敗: {:?}", e);
        }
    }

    let access_cookie = Cookie::build(("access_token", ""))
        .http_only(true)
        .same_site(SameSite::Lax)
        .path("/api")
        .max_age(time::Duration::seconds(0))
        .build();
    let refresh_cookie = build_clear_refresh_cookie();
    let jar = jar.add(access_cookie).add(refresh_cookie);

    (
        jar,
        response_handler(StatusCode::OK, "success".to_string(), None, None),
    )
}

async fn revoke_all_user_tokens(
    db: &crate::common::database::Database,
    user_id: &str,
) -> Result<(), String> {
    use crate::models::refresh_token::RefreshToken;
    use firestore::*;

    let tokens: Vec<RefreshToken> = db
        .client
        .fluent()
        .select()
        .from("refresh_tokens")
        .filter(|q| {
            q.for_all([
                q.field(path!(RefreshToken::user_id)).eq(user_id),
                q.field(path!(RefreshToken::is_revoked)).eq(false),
            ])
        })
        .obj()
        .query()
        .await
        .map_err(|e| format!("トークン検索失敗: {}", e))?;

    let now = chrono::Utc::now().timestamp();
    for mut token in tokens {
        token.is_revoked = true;
        token.revoked_at = Some(now);
        let _ = db
            .update::<RefreshToken>("refresh_tokens", &token.token_hash, token)
            .await;
    }
    Ok(())
}
```

**Step 2: main.rsのauth_logoutルートにStateを渡せるよう確認**

auth_logoutは既にRouter配下なので、AppState(Arc<Database>)は自動注入される。変更不要。

**Step 3: ビルド確認**

Run: `cd jlpt-app-backend && cargo check`
Expected: PASS

**Step 4: コミット**

```bash
git add jlpt-app-backend/src/api/user.rs
git commit -m "feat(backend): logoutで全リフレッシュトークン無効化+リフレッシュCookieクリア"
```

---

### Task 7: フロントエンドの透過的リフレッシュ実装 (jlpt-app-frontend)

**Files:**
- Modify: `jlpt-app-frontend/src/lib/api.ts`

**Step 1: api.tsにリフレッシュロジックを追加**

`jlpt-app-frontend/src/lib/api.ts` のファイル先頭（API_BASE定義の後）に追加:

```typescript
// --- Transparent Token Refresh ---
let refreshPromise: Promise<boolean> | null = null;

async function tryRefresh(): Promise<boolean> {
  try {
    const res = await fetch(`${API_BASE}/api/auth/refresh`, {
      method: "POST",
      credentials: "include",
    });
    return res.ok;
  } catch {
    return false;
  }
}

async function fetchWithRefresh(
  input: RequestInfo | URL,
  init?: RequestInit
): Promise<Response> {
  const res = await fetch(input, init);

  if (res.status !== 401) return res;

  // 401 → リフレッシュ試行（同時に1回だけ）
  if (!refreshPromise) {
    refreshPromise = tryRefresh().finally(() => {
      refreshPromise = null;
    });
  }

  const refreshed = await refreshPromise;
  if (!refreshed) return res; // リフレッシュ失敗 → 元の401を返す

  // リトライ
  return fetch(input, init);
}
```

**Step 2: 認証が必要なAPI呼び出しをfetchWithRefreshに置換**

以下の関数内の `fetch(` を `fetchWithRefresh(` に置き換え:

- `fetchAuthMe()` — **変更しない**（401でnullを返す既存動作を維持）
- `recordAnswer()` → `fetchWithRefresh`
- `fetchHistory()` → `fetchWithRefresh`
- `fetchUserStats()` → `fetchWithRefresh`
- `fetchMistakes()` → `fetchWithRefresh`
- `adminFetchSummary()` → `fetchWithRefresh`
- `adminFetchBadQuestions()` → `fetchWithRefresh`
- `adminDeleteQuestion()` → `fetchWithRefresh`
- `adminBulkDelete()` → `fetchWithRefresh`
- `adminFetchStats()` → `fetchWithRefresh`
- `adminFetchCoverage()` → `fetchWithRefresh`

**パターン例:**

```typescript
// 変更前
export async function fetchHistory(limit = 50) {
  const res = await fetch(`${API_BASE}/api/users/me/history?limit=${limit}`, {
    credentials: "include",
  });

// 変更後
export async function fetchHistory(limit = 50) {
  const res = await fetchWithRefresh(`${API_BASE}/api/users/me/history?limit=${limit}`, {
    credentials: "include",
  });
```

**注意**: `fetchMeta()`, `fetchQuestions()`, `fetchQuestionById()`, `submitVote()`, `signup()`, `signin()`, `logout()` は認証不要または特殊処理のため変更しない。

**Step 3: セッション切れイベントのエクスポート**

api.tsの末尾に追加:

```typescript
// Session expired event (for UI to show modal)
type SessionExpiredHandler = () => void;
let onSessionExpired: SessionExpiredHandler | null = null;

export function setOnSessionExpired(handler: SessionExpiredHandler) {
  onSessionExpired = handler;
}

export function notifySessionExpired() {
  onSessionExpired?.();
}
```

**Step 4: ビルド確認**

Run: `cd jlpt-app-frontend && npm run build`
Expected: PASS

**Step 5: コミット**

```bash
git add jlpt-app-frontend/src/lib/api.ts
git commit -m "feat(frontend): 透過的トークンリフレッシュ（401→自動リフレッシュ→リトライ、mutex付き）"
```

---

### Task 8: セッション切れモーダルの追加 (jlpt-app-frontend)

**Files:**
- Create: `jlpt-app-frontend/src/components/SessionExpiredModal.tsx`
- Modify: `jlpt-app-frontend/src/app/layout.tsx`

**Step 1: SessionExpiredModal.tsxを作成**

```typescript
"use client";
import { useEffect, useState } from "react";
import { useRouter } from "next/navigation";
import { setOnSessionExpired } from "@/lib/api";

export default function SessionExpiredModal() {
  const router = useRouter();
  const [show, setShow] = useState(false);

  useEffect(() => {
    setOnSessionExpired(() => setShow(true));
    return () => setOnSessionExpired(() => {});
  }, []);

  if (!show) return null;

  return (
    <div className="fixed inset-0 z-50 flex items-center justify-center bg-black/40">
      <div className="bg-white rounded-xl shadow-lg p-6 max-w-sm w-full mx-4">
        <h2 className="text-lg font-bold text-gray-900 mb-2">セッションが切れました</h2>
        <p className="text-sm text-gray-600 mb-4">再度ログインしてください。</p>
        <button
          onClick={() => {
            setShow(false);
            router.push("/login");
          }}
          className="w-full py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700 transition"
        >
          ログイン画面へ
        </button>
      </div>
    </div>
  );
}
```

**Step 2: layout.tsxにモーダルを追加**

`jlpt-app-frontend/src/app/layout.tsx` のbody内にコンポーネント追加:

```tsx
import SessionExpiredModal from "@/components/SessionExpiredModal";

// body内に追加
<SessionExpiredModal />
```

**Step 3: fetchWithRefreshにセッション切れ通知を追加**

`jlpt-app-frontend/src/lib/api.ts` のfetchWithRefresh関数を修正:

```typescript
async function fetchWithRefresh(
  input: RequestInfo | URL,
  init?: RequestInit
): Promise<Response> {
  const res = await fetch(input, init);

  if (res.status !== 401) return res;

  if (!refreshPromise) {
    refreshPromise = tryRefresh().finally(() => {
      refreshPromise = null;
    });
  }

  const refreshed = await refreshPromise;
  if (!refreshed) {
    notifySessionExpired(); // ← 追加: モーダル表示
    return res;
  }

  return fetch(input, init);
}
```

**Step 4: ビルド確認**

Run: `cd jlpt-app-frontend && npm run build`
Expected: PASS

**Step 5: コミット**

```bash
git add jlpt-app-frontend/src/components/SessionExpiredModal.tsx jlpt-app-frontend/src/app/layout.tsx jlpt-app-frontend/src/lib/api.ts
git commit -m "feat(frontend): セッション切れモーダル（リフレッシュ失敗時に表示）"
```

---

### Task 9: Firestoreインデックス設定 + ドキュメント更新

**Files:**
- Create: `jlpt-app-backend/firestore.indexes.json` (なければ)
- Modify: `docs/gcp-environment.md`

**Step 1: Firestoreの複合インデックスを設定**

`refresh_tokens`コレクションに以下のインデックスが必要:

1. `user_id` ASC + `is_revoked` ASC（ログアウト時の全無効化）
2. `token_family` ASC + `is_revoked` ASC（漏洩検知時のfamily無効化）

Firebaseコンソールまたはfirestore.indexes.jsonで設定:

```json
{
  "indexes": [
    {
      "collectionGroup": "refresh_tokens",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "user_id", "order": "ASCENDING" },
        { "fieldPath": "is_revoked", "order": "ASCENDING" }
      ]
    },
    {
      "collectionGroup": "refresh_tokens",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "token_family", "order": "ASCENDING" },
        { "fieldPath": "is_revoked", "order": "ASCENDING" }
      ]
    }
  ]
}
```

**Step 2: docs/gcp-environment.mdを更新**

リフレッシュトークン機構の追加内容を反映:
- アクセストークン: 24h → 30min
- リフレッシュトークン: 14日、httpOnly Cookie
- Firestoreコレクション: `refresh_tokens` 追加
- エンドポイント: `POST /api/auth/refresh` 追加

**Step 3: コミット**

```bash
git add jlpt-app-backend/firestore.indexes.json docs/gcp-environment.md
git commit -m "docs: リフレッシュトークン機構のFirestoreインデックス・ドキュメント更新"
```

---

## Phase 2: Issue #2 — Gemini AIパイプライン改善

### Task 10: Gemini API呼び出しにStructured Output導入 (jlpt-app-scripts)

**Files:**
- Modify: `jlpt-app-scripts/bin/utils.rs`

**Step 1: request_gemini_api関数のgeneration_configを変更**

`jlpt-app-scripts/bin/utils.rs` の `request_gemini_api()` 関数を変更:

```rust
// 変更前
    let generation_config = Some(GenerationConfig {
        temperature: Some(0.8),
        top_p: None,
        top_k: None,
        candidate_count: None,
        max_output_tokens: Some(8192),
        stop_sequences: None,
        response_mime_type: None,
        response_schema: None,
    });

// 変更後
    let generation_config = Some(GenerationConfig {
        temperature: Some(0.8),
        top_p: Some(0.95),
        top_k: None,
        candidate_count: None,
        max_output_tokens: Some(8192),
        stop_sequences: None,
        response_mime_type: Some("application/json".to_string()),
        response_schema: None, // google-generative-ai-rs 0.3.4ではresponse_schemaフィールドの型確認が必要
    });
```

**注意**: `google-generative-ai-rs` 0.3.4のGenerationConfig構造体がresponse_schemaフィールドをサポートしているか要確認。サポートしていない場合は `response_mime_type: "application/json"` のみで対応し、JSON Schema強制はプロンプト側で担保する。

**Step 2: ビルド確認**

Run: `cd jlpt-app-scripts && cargo check`
Expected: PASS

**Step 3: コミット**

```bash
git add jlpt-app-scripts/bin/utils.rs
git commit -m "feat(scripts): Gemini APIにresponse_mime_type=application/json、top_p=0.95を設定"
```

---

### Task 11: SYSTEM_INSTRUCTIONの強化 (jlpt-app-scripts)

**Files:**
- Modify: `jlpt-app-scripts/bin/utils.rs`

**Step 1: SYSTEM_INSTRUCTIONを改善**

```rust
// 変更前
pub const SYSTEM_INSTRUCTION: &str = r#"あなたはJLPT（日本語能力試験）の公式問題作成の専門家です。以下を厳守してください：
1. 出力は必ず有効なJSONのみ（マークダウン記法禁止、説明文禁止）
2. 選択肢は必ず4つ。正解は必ず1つだけ
3. 正解の位置（1〜4）を偏らせない
4. 誤答は「一見正しそうだが明確な理由で不正解」であること
5. 選択肢の長さ・構造・語彙レベルを揃えること
6. 指定されたレベルの語彙・文法範囲を厳守すること
7. 指定されたカテゴリの問題のみを生成すること"#;

// 変更後
pub const SYSTEM_INSTRUCTION: &str = r#"あなたはJLPT（日本語能力試験）の公式問題作成の専門家です。テスト理論（項目応答理論）に基づき、弁別力の高い問題を作成してください。

## 厳守事項
1. 出力は必ず有効なJSONのみ（マークダウン記法禁止、説明文禁止）
2. 選択肢は必ず4つ。正解は必ず1つだけ
3. 正解の位置（1〜4）を偏らせない
4. 誤答は「一見正しそうだが明確な理由で不正解」であること
5. 選択肢の長さ・構造・語彙レベルを揃えること
6. 指定されたレベルの語彙・文法範囲を厳守すること
7. 指定されたカテゴリの問題のみを生成すること

## 選択肢設計の原則（最重要）
- 全ての誤答（ディストラクター）は、正解と同じ品詞・語彙分野から選ぶこと
- 消去法で正解が分かる問題は品質不良。4つの選択肢すべてが文脈なしでは正解に見えること
- 選択肢の文字数を±1文字以内に揃えること

## 良い問題の例
問題: 彼は会議で重要な（　　）を果たした。
選択肢: 1.役割 2.役目 3.配役 4.役者
正解: 1
良い理由: 全ての選択肢が「役」を含む名詞で、文脈で区別する必要がある

## 悪い問題の例（このような問題を生成してはいけない）
問題: 彼は（　　）を食べた。
選択肢: 1.りんご 2.テレビ 3.電車 4.建物
正解: 1
悪い理由: 消去法で即答可能。「食べる」の目的語になりうる選択肢が1つしかない"#;
```

**Step 2: ビルド確認**

Run: `cd jlpt-app-scripts && cargo check`
Expected: PASS

**Step 3: コミット**

```bash
git add jlpt-app-scripts/bin/utils.rs
git commit -m "feat(scripts): SYSTEM_INSTRUCTIONにテスト理論・負例付きFew-shot・選択肢設計原則を追加"
```

---

### Task 12: バリデーションに歩留まり率レポート追加 (jlpt-app-scripts)

**Files:**
- Modify: `jlpt-app-scripts/bin/1_5_validate_questions.rs`

**Step 1: ValidationReport構造体を追加**

ファイル先頭に追加:

```rust
use std::collections::HashMap;

#[derive(Serialize, Debug)]
struct ValidationReport {
    timestamp: String,
    total: usize,
    passed: usize,
    failed: usize,
    overall_yield_rate: f64,
    by_category: HashMap<String, CategoryYield>,
}

#[derive(Serialize, Debug)]
struct CategoryYield {
    total: usize,
    passed: usize,
    failed: usize,
    yield_rate: f64,
}
```

**Step 2: validate後にレポートを生成・出力**

バリデーション実行後に追加:

```rust
    // 歩留まりレポート生成
    let mut cat_stats: HashMap<String, (usize, usize)> = HashMap::new(); // (total, passed)
    // ... validated/rejectedのカウントをカテゴリ別に集計

    let report = ValidationReport {
        timestamp: chrono::Utc::now().to_rfc3339(),
        total: all_questions.len(),
        passed: validated.len(),
        failed: rejected.len(),
        overall_yield_rate: validated.len() as f64 / all_questions.len().max(1) as f64,
        by_category: cat_stats
            .into_iter()
            .map(|(name, (total, passed))| {
                let yield_rate = passed as f64 / total.max(1) as f64;
                if yield_rate < 0.6 {
                    log::warn!("⚠ 歩留まり60%未満: {} ({:.1}%)", name, yield_rate * 100.0);
                }
                (name, CategoryYield {
                    total,
                    passed,
                    failed: total - passed,
                    yield_rate,
                })
            })
            .collect(),
    };

    let report_json = serde_json::to_string_pretty(&report).unwrap();
    utils::write_file(&output_dir.join("validation_report.json"), &report_json)?;
    info!("歩留まりレポート出力: validation_report.json (全体: {:.1}%)", report.overall_yield_rate * 100.0);
```

**Step 3: ビルド確認**

Run: `cd jlpt-app-scripts && cargo check`
Expected: PASS

**Step 4: コミット**

```bash
git add jlpt-app-scripts/bin/1_5_validate_questions.rs
git commit -m "feat(scripts): バリデーションに歩留まり率レポート出力追加（60%未満警告付き）"
```

---

### Task 13: scriptsドキュメント更新

**Files:**
- Modify: `jlpt-app-scripts/docs/pipeline.md`
- Modify: `jlpt-app-scripts/docs/quality-management.md`

**Step 1: pipeline.mdにStructured Output・新SYSTEM_INSTRUCTIONの記述を追加**

**Step 2: quality-management.mdに歩留まりレポートの説明を追加**

**Step 3: コミット**

```bash
git add jlpt-app-scripts/docs/
git commit -m "docs(scripts): Structured Output導入・歩留まりレポートの説明追加"
```

---

## Phase 3: Issue #1 — 品質基盤

### Task 14: 既存問題にprompt_version: "legacy"一括タグ付け (jlpt-app-scripts)

**Files:**
- Create: `jlpt-app-scripts/bin/99_tag_legacy.rs`
- Modify: `jlpt-app-scripts/Cargo.toml`（[[bin]]追加）

**Step 1: 一括タグ付けスクリプトを作成**

`jlpt-app-scripts/bin/99_tag_legacy.rs`:

```rust
use firestore::*;
use log::info;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    env_logger::init();

    let project_id = std::env::var("PROJECT_ID").expect("PROJECT_ID must be set");
    let db = FirestoreDb::new(&project_id).await?;

    // 全レベルの問題を取得
    let levels = ["n1", "n2", "n3", "n4", "n5"];
    let mut updated = 0;

    for level in &levels {
        let collection = format!("{}_questions", level);
        info!("Processing collection: {}", collection);

        let docs: Vec<serde_json::Value> = db
            .fluent()
            .select()
            .from(&collection)
            .obj()
            .query()
            .await?;

        for doc in &docs {
            if doc.get("prompt_version").is_none() {
                let mut updated_doc = doc.clone();
                updated_doc["prompt_version"] = serde_json::json!("legacy");

                let id = doc["id"].as_str().unwrap_or_default();
                if !id.is_empty() {
                    db.fluent()
                        .update()
                        .in_col(&collection)
                        .document_id(id)
                        .object(&updated_doc)
                        .execute()
                        .await?;
                    updated += 1;
                }
            }
        }
        info!("{}: updated {} documents", collection, updated);
    }

    info!("Total updated: {}", updated);
    Ok(())
}
```

**Step 2: Cargo.tomlにbin追加**

```toml
[[bin]]
name = "99_tag_legacy"
path = "bin/99_tag_legacy.rs"
```

**Step 3: ビルド確認**

Run: `cd jlpt-app-scripts && cargo check --bin 99_tag_legacy`
Expected: PASS

**Step 4: コミット**

```bash
git add jlpt-app-scripts/bin/99_tag_legacy.rs jlpt-app-scripts/Cargo.toml
git commit -m "feat(scripts): 既存問題にprompt_version=legacy一括タグ付けスクリプト"
```

---

### Task 15: 問題報告エンドポイント追加 (jlpt-app-backend)

**Files:**
- Create: `jlpt-app-backend/src/models/report.rs`
- Create: `jlpt-app-backend/src/api/report.rs`
- Modify: `jlpt-app-backend/src/models/mod.rs`
- Modify: `jlpt-app-backend/src/api/mod.rs`
- Modify: `jlpt-app-backend/src/main.rs`

**Step 1: QuestionReportモデルを作成**

`jlpt-app-backend/src/models/report.rs`:

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct QuestionReport {
    pub question_id: String,
    pub user_id: String,
    pub reported_at: i64,
}

impl QuestionReport {
    pub fn new(question_id: String, user_id: String) -> Self {
        Self {
            question_id,
            user_id,
            reported_at: chrono::Utc::now().timestamp(),
        }
    }

    pub fn doc_id(question_id: &str, user_id: &str) -> String {
        format!("{}_{}", question_id, user_id)
    }
}
```

**Step 2: 報告APIハンドラを作成**

`jlpt-app-backend/src/api/report.rs`:

```rust
use std::sync::Arc;

use axum::{extract::{Path, State}, response::IntoResponse};
use http::StatusCode;
use serde_json::json;

use crate::api::utils::response_handler;
use crate::common::database::Database;
use crate::models::claim::{AdminClaims, Claims};
use crate::models::report::QuestionReport;

pub async fn report_question(
    State(db): State<Arc<Database>>,
    claims: Claims,
    Path(question_id): Path<String>,
) -> impl IntoResponse {
    let doc_id = QuestionReport::doc_id(&question_id, &claims.user_id);

    // 重複チェック
    if let Ok(Some(_)) = db.read::<QuestionReport>("reports", &doc_id).await {
        return response_handler(
            StatusCode::CONFLICT,
            "error".to_string(),
            None,
            Some("既に報告済みです".to_string()),
        );
    }

    let report = QuestionReport::new(question_id, claims.user_id);
    match db.create::<QuestionReport>("reports", &doc_id, report).await {
        Ok(_) => response_handler(
            StatusCode::OK,
            "success".to_string(),
            Some(json!({"reported": true})),
            None,
        ),
        Err(e) => {
            log::error!("報告保存失敗: {:?}", e);
            response_handler(
                StatusCode::INTERNAL_SERVER_ERROR,
                "error".to_string(),
                None,
                Some("報告の保存に失敗しました".to_string()),
            )
        }
    }
}

pub async fn list_reports(
    State(db): State<Arc<Database>>,
    _claims: AdminClaims,
) -> impl IntoResponse {
    use firestore::*;

    match db
        .client
        .fluent()
        .select()
        .from("reports")
        .order_by([(path!(QuestionReport::reported_at), FirestoreQueryDirection::Descending)])
        .limit(100)
        .obj::<QuestionReport>()
        .query()
        .await
    {
        Ok(reports) => {
            // question_idごとの報告数を集計
            let mut counts: std::collections::HashMap<String, usize> = std::collections::HashMap::new();
            for r in &reports {
                *counts.entry(r.question_id.clone()).or_insert(0) += 1;
            }
            let mut sorted: Vec<_> = counts.into_iter().collect();
            sorted.sort_by(|a, b| b.1.cmp(&a.1));

            response_handler(
                StatusCode::OK,
                "success".to_string(),
                Some(json!(sorted.iter().map(|(id, count)| json!({"question_id": id, "report_count": count})).collect::<Vec<_>>())),
                None,
            )
        }
        Err(e) => {
            log::error!("報告一覧取得失敗: {:?}", e);
            response_handler(
                StatusCode::INTERNAL_SERVER_ERROR,
                "error".to_string(),
                None,
                Some("報告一覧の取得に失敗しました".to_string()),
            )
        }
    }
}
```

**Step 3: mod.rsにエクスポート追加**

`models/mod.rs`:
```rust
pub mod report;
```

`api/mod.rs`:
```rust
pub mod report;
```

**Step 4: main.rsにルート追加**

```rust
    .route("/api/questions/:id/report", post(api::report::report_question))
    .route("/api/admin/reports", get(api::report::list_reports))
```

**Step 5: ビルド確認**

Run: `cd jlpt-app-backend && cargo check`
Expected: PASS

**Step 6: コミット**

```bash
git add jlpt-app-backend/src/models/report.rs jlpt-app-backend/src/api/report.rs jlpt-app-backend/src/models/mod.rs jlpt-app-backend/src/api/mod.rs jlpt-app-backend/src/main.rs
git commit -m "feat(backend): 問題報告エンドポイント（POST /api/questions/:id/report, GET /api/admin/reports）"
```

---

### Task 16: フロントエンドに報告ボタン追加 (jlpt-app-frontend)

**Files:**
- Modify: `jlpt-app-frontend/src/lib/api.ts`
- Modify: `jlpt-app-frontend/src/app/[level]/quiz/page.tsx`

**Step 1: api.tsに報告API関数を追加**

```typescript
export async function reportQuestion(questionId: string): Promise<boolean> {
  try {
    const res = await fetchWithRefresh(`${API_BASE}/api/questions/${questionId}/report`, {
      method: "POST",
      credentials: "include",
    });
    return res.ok || res.status === 409; // 409=既に報告済み → 成功扱い
  } catch {
    return false;
  }
}
```

**Step 2: quiz/page.tsxに報告ボタンを追加**

結果表示セクション（good/badボタンの近く）に追加:

```tsx
import { reportQuestion } from "@/lib/api";

// state追加
const [reported, setReported] = useState(false);

// ハンドラ追加
const handleReport = async () => {
  if (!currentQuestion) return;
  const ok = await reportQuestion(currentQuestion.id);
  if (ok) setReported(true);
};

// 問題切り替え時にreportedをリセット
// 既存のnextQuestion関数内に追加:
setReported(false);

// UI: good/badボタンの下に追加
{!reported ? (
  <button
    onClick={handleReport}
    className="text-xs text-gray-400 hover:text-red-500 transition mt-2"
    title="この問題を報告"
  >
    ⚑ 問題を報告
  </button>
) : (
  <span className="text-xs text-gray-400 mt-2">報告しました</span>
)}
```

**Step 3: ビルド確認**

Run: `cd jlpt-app-frontend && npm run build`
Expected: PASS

**Step 4: コミット**

```bash
git add jlpt-app-frontend/src/lib/api.ts jlpt-app-frontend/src/app/\\[level\\]/quiz/page.tsx
git commit -m "feat(frontend): 問題報告ボタン追加（最小限、1問1回、ログイン済みユーザーのみ）"
```

---

### Task 17: ドキュメント最終更新

**Files:**
- Modify: `docs/gcp-environment.md`
- Modify: `jlpt-app-backend/docs/api-spec.md`

**Step 1: gcp-environment.mdに全変更を反映**

- リフレッシュトークン機構
- Firestoreコレクション追加（refresh_tokens, reports）
- 新エンドポイント一覧
- prompt_versionフィールド

**Step 2: api-spec.mdに新エンドポイントを追加**

- `POST /api/auth/refresh`
- `POST /api/questions/:id/report`
- `GET /api/admin/reports`

**Step 3: コミット**

```bash
git add docs/ jlpt-app-backend/docs/
git commit -m "docs: Phase 1-3全変更をドキュメントに反映"
```

---

## 実装順序まとめ

| Task | Phase | 内容 | リポジトリ |
|------|-------|------|-----------|
| 1 | 1 | RefreshTokenモデル | backend |
| 2 | 1 | Claims jti + 30分 | backend |
| 3 | 1 | トークンCookieユーティリティ | backend |
| 4 | 1 | signin改修 | backend |
| 5 | 1 | /api/auth/refresh | backend |
| 6 | 1 | logout改修 | backend |
| 7 | 1 | 透過的リフレッシュ | frontend |
| 8 | 1 | セッション切れモーダル | frontend |
| 9 | 1 | Firestoreインデックス + docs | backend + docs |
| 10 | 2 | Structured Output | scripts |
| 11 | 2 | SYSTEM_INSTRUCTION強化 | scripts |
| 12 | 2 | 歩留まりレポート | scripts |
| 13 | 2 | scriptsドキュメント更新 | scripts |
| 14 | 3 | legacy一括タグ付け | scripts |
| 15 | 3 | 問題報告エンドポイント | backend |
| 16 | 3 | 報告ボタン | frontend |
| 17 | 3 | ドキュメント最終更新 | docs |

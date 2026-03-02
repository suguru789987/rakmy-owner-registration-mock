# ラクミー タイムレコード — オーナー登録・店舗管理・権限管理 要件書

> **バージョン:** v1.1
> **作成日:** 2026-03-02
> **モック:** https://suguru789987.github.io/rakmy-owner-registration-mock/
> **モックソース:** https://github.com/suguru789987/rakmy-owner-registration-mock/blob/main/index.html
> **対象リポジトリ:** rakmy/rakmy_server (Rails 7.2 / MySQL / Sidekiq / Redis)

---

## 1. 概要

### 1.1 目的
現在開発者がCLIで手動実行しているオーナー登録を、セルフサービスUI + 招待フローとして自動化する。
加えて、オーナー → 店長 → 従業員の権限階層に基づく招待・管理機能を実装する。

### 1.2 権限階層（案3 ハイブリッド）

```
オーナー（全権管理者）
  ├── 全店舗・全機能にアクセス
  ├── 店舗の追加・編集・削除
  ├── 店長の招待・任命
  ├── 従業員の直接登録（全店舗）
  ├── 他オーナーの招待
  └── 給与・経営管理・外部連携

店長（店舗管理者）※任意配置
  ├── 担当店舗のみ管理
  ├── 従業員の招待・管理
  ├── 勤怠承認
  ├── 給与・経営管理（担当店舗）
  ├── 外部連携設定
  └── 担当店舗従業員のPW再発行

従業員
  ├── 打刻操作
  └── 自分の勤怠確認
```

### 1.3 権限マトリクス

| 操作 | オーナー | 店長 | 従業員 |
|------|---------|------|--------|
| 店舗の追加・編集・削除 | ✅ | ❌ | ❌ |
| 店舗情報の編集（担当店舗） | ✅ 全店舗 | ✅ 担当のみ | ❌ |
| オーナー招待 | ✅ | ❌ | ❌ |
| 店長の招待・任命 | ✅ | ❌ | ❌ |
| 従業員の招待・登録 | ✅ 全店舗 | ✅ 担当店舗のみ | ❌ |
| 従業員情報の閲覧・編集 | ✅ 全店舗 | ✅ 担当店舗のみ | ❌ |
| 勤怠データ閲覧 | ✅ 全店舗 | ✅ 担当店舗のみ | ✅ 自分のみ |
| 自分のPW確認・変更 | ✅ | ✅ | ❌ |
| PW再設定（パスワードを忘れた） | ✅ | ✅ | ✅ |
| 部下のPW再発行 | ✅ 全ユーザー | ✅ 担当店舗従業員のみ | ❌ |
| PW再発行依頼 | ❌ | ❌ | ✅ 管理者へ依頼 |
| 招待メールの再送信 | ✅ | ✅ 担当店舗従業員のみ | ❌ |
| 打刻データの修正 | ✅ 全店舗 | ✅ 担当店舗のみ | ❌ |
| 打刻操作 | ❌ | ❌ | ✅ |
| 給与・経営管理 | ✅ 全店舗 | ✅ 担当店舗のみ | ❌ |
| 外部連携設定 | ✅ | ✅ | ❌ |

---

## 2. 機能一覧とモック画面対応

| ID | 機能 | モック画面 | 優先度 |
|----|------|-----------|--------|
| A | 新規会社オーナー登録（セルフサインアップ） | A-1〜A-4 | P0 |
| B | オーナー/管理者招待 | B-1〜B-4 | P0 |
| C | 店舗管理 + 店長招待 | C-1〜C-4 | P0 |
| D | 従業員管理（オーナー/店長視点） | D-1〜D-4 | P1 |
| E | パスワード再設定（セルフサービス） | E-1〜E-3 | P0 |
| F | 管理者によるPW再発行 | B-1, D-1, D-2 | P0 |
| G | 招待メール再送信 | B-1, C-1, D-1, D-2 | P0 |
| H | PW確認・変更（オーナー/店長） | F-1 | P0 |
| I | PW再発行依頼（従業員） | F-2 | P0 |
| J | ログイン画面改修 | login | P1 |

---

## 3. DB設計

### 3.1 新規テーブル

#### `invitations`（招待管理）
```sql
CREATE TABLE invitations (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  company_id BIGINT UNSIGNED NOT NULL,
  invited_by_id BIGINT UNSIGNED,          -- 招待したユーザーID（NULLはセルフサインアップ）
  email VARCHAR(255) NOT NULL,
  name VARCHAR(255) NOT NULL,
  role ENUM('owner', 'manager', 'employee') NOT NULL,
  token VARCHAR(64) NOT NULL UNIQUE,
  status ENUM('pending', 'accepted', 'expired', 'cancelled') NOT NULL DEFAULT 'pending',
  store_id BIGINT UNSIGNED,               -- 店長/従業員の場合の所属店舗
  expires_at DATETIME NOT NULL,
  accepted_at DATETIME,
  created_at DATETIME NOT NULL,
  updated_at DATETIME NOT NULL,

  INDEX idx_invitations_token (token),
  INDEX idx_invitations_email (email),
  INDEX idx_invitations_company_status (company_id, status),
  FOREIGN KEY (company_id) REFERENCES companies(id),
  FOREIGN KEY (invited_by_id) REFERENCES users(id),
  FOREIGN KEY (store_id) REFERENCES stores(id)
);
```

#### `email_verifications`（メール認証コード）
```sql
CREATE TABLE email_verifications (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  email VARCHAR(255) NOT NULL,
  code VARCHAR(6) NOT NULL,
  attempts INT NOT NULL DEFAULT 0,
  max_attempts INT NOT NULL DEFAULT 5,
  verified BOOLEAN NOT NULL DEFAULT FALSE,
  token VARCHAR(64) NOT NULL UNIQUE,      -- 認証後に発行するセッショントークン
  expires_at DATETIME NOT NULL,
  created_at DATETIME NOT NULL,
  updated_at DATETIME NOT NULL,

  INDEX idx_email_verifications_email (email),
  INDEX idx_email_verifications_token (token)
);
```

#### `password_resets`（パスワード再設定）
```sql
CREATE TABLE password_resets (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  user_id BIGINT UNSIGNED,                    -- NULLの場合はセルフサービス（メールで検索）
  email VARCHAR(255) NOT NULL,
  code VARCHAR(6) NOT NULL,
  token VARCHAR(64) NOT NULL UNIQUE,          -- 認証後のセッショントークン
  attempts INT NOT NULL DEFAULT 0,
  max_attempts INT NOT NULL DEFAULT 5,
  verified BOOLEAN NOT NULL DEFAULT FALSE,
  reset_by_id BIGINT UNSIGNED,               -- 管理者によるリセットの場合、リセットした管理者のID
  expires_at DATETIME NOT NULL,
  created_at DATETIME NOT NULL,
  updated_at DATETIME NOT NULL,

  INDEX idx_password_resets_email (email),
  INDEX idx_password_resets_token (token),
  FOREIGN KEY (user_id) REFERENCES users(id),
  FOREIGN KEY (reset_by_id) REFERENCES users(id)
);
```

### 3.2 既存テーブル変更

#### `users` テーブルへのカラム追加
```sql
ALTER TABLE users
  ADD COLUMN role ENUM('owner', 'manager', 'employee') NOT NULL DEFAULT 'employee',
  ADD COLUMN invitation_id BIGINT UNSIGNED,
  ADD COLUMN password_changed_at DATETIME,
  ADD COLUMN force_password_change BOOLEAN NOT NULL DEFAULT FALSE;
```

#### `stores` テーブルへのカラム追加
```sql
ALTER TABLE stores
  ADD COLUMN manager_id BIGINT UNSIGNED,
  ADD FOREIGN KEY (manager_id) REFERENCES users(id);
```

> **注意:** 既存の `users` / `stores` / `companies` テーブル構造は実際のスキーマを確認のうえ調整すること。

---

## 4. API設計

### 4.1 フローA: 新規会社オーナー登録

#### `POST /api/v1/signup`
新規会社 + オーナーの仮登録。メール認証コードを送信。

**Request:**
```json
{
  "company_name": "株式会社サンプル",
  "owner_name": "山田 太郎",
  "email": "yamada@sample-company.jp"
}
```

**Response:** `201 Created`
```json
{
  "message": "確認コードをメールに送信しました",
  "verification_token": "temp_abc123..."
}
```

**バリデーション:**
- `company_name`: 必須、1〜100文字
- `owner_name`: 必須、1〜50文字
- `email`: 必須、メール形式、既存ユーザーと重複不可

**処理内容:**
1. バリデーション
2. `email_verifications` レコード作成（6桁コード生成、有効期限30分）
3. 確認コードメール送信（Sidekiqジョブ）

**レート制限:** 同一IP 3回/時間

---

#### `POST /api/v1/signup/verify`
メール認証コードの検証。

**Request:**
```json
{
  "verification_token": "temp_abc123...",
  "code": "482915"
}
```

**Response:** `200 OK`
```json
{
  "message": "メールアドレスが確認されました",
  "setup_token": "setup_xyz789..."
}
```

**エラーレスポンス:**
- `401` コード不一致（残り試行回数を返す）
- `410` 有効期限切れ
- `429` 試行回数超過（5回）

---

#### `POST /api/v1/signup/password`
パスワード設定 + 会社・オーナーアカウント本登録。

**Request:**
```json
{
  "setup_token": "setup_xyz789...",
  "password": "securepass123",
  "password_confirmation": "securepass123"
}
```

**Response:** `201 Created`
```json
{
  "message": "登録が完了しました",
  "user": {
    "id": 1,
    "name": "山田 太郎",
    "email": "yamada@sample-company.jp",
    "role": "owner",
    "company": {
      "id": 1,
      "name": "株式会社サンプル"
    }
  },
  "auth_token": "jwt_or_session_token..."
}
```

**処理内容（トランザクション）:**
1. `setup_token` の検証
2. `companies` レコード作成
3. `users` レコード作成（role: owner、パスワードbcryptハッシュ化）
4. セッション/JWT発行
5. ウェルカムメール送信（Sidekiqジョブ）

---

### 4.2 フローB: オーナー/管理者招待

#### `GET /api/v1/administrators`
管理者一覧取得。

**認証:** 必須（オーナーのみ）

**Response:** `200 OK`
```json
{
  "administrators": [
    {
      "id": 1,
      "name": "山田 太郎",
      "email": "yamada@sample-company.jp",
      "role": "owner",
      "stores": "全店舗",
      "status": "active",
      "last_login_at": "2026-03-02T09:15:00+09:00",
      "is_self": true
    },
    {
      "id": null,
      "name": "佐藤 一郎",
      "email": "sato@example.jp",
      "role": "owner",
      "stores": "全店舗",
      "status": "pending",
      "invitation_id": 5,
      "invited_at": "2026-03-01T10:00:00+09:00",
      "expires_at": "2026-03-04T10:00:00+09:00"
    }
  ]
}
```

---

#### `POST /api/v1/administrators/invite`
管理者（オーナー/店長）を招待。

**認証:** 必須（オーナーのみ）

**Request:**
```json
{
  "email": "sato@example.jp",
  "name": "佐藤 一郎",
  "role": "owner",
  "store_id": null
}
```
店長の場合:
```json
{
  "email": "ito@example.jp",
  "name": "伊藤 四郎",
  "role": "manager",
  "store_id": 3
}
```

**Response:** `201 Created`
```json
{
  "message": "招待メールを送信しました",
  "invitation": {
    "id": 5,
    "email": "sato@example.jp",
    "role": "owner",
    "status": "pending",
    "expires_at": "2026-03-05T09:15:00+09:00"
  }
}
```

**バリデーション:**
- `email`: 必須、メール形式、同一会社内で未使用
- `name`: 必須
- `role`: `owner` or `manager`
- `store_id`: `manager` の場合は必須、`owner` の場合は NULL
- 招待上限: 10回/会社/日

**処理内容:**
1. 権限チェック（オーナーのみ）
2. 重複チェック（同一メールへの有効な招待が存在しないこと）
3. `invitations` レコード作成（ランダム64文字トークン、有効期限72時間）
4. 招待メール送信（Sidekiqジョブ）

---

#### `DELETE /api/v1/administrators/invitations/:id`
招待のキャンセル。

**認証:** 必須（オーナーのみ）

**Response:** `200 OK`

---

#### `POST /api/v1/administrators/invitations/:id/resend`
招待メールの再送信。期限切れの場合は新しいトークンで再発行。

**認証:** 必須（オーナーのみ）

**Response:** `200 OK`

---

#### `GET /api/v1/invitations/:token`
招待トークンの検証（招待受諾画面の表示用）。

**認証:** 不要

**Response:** `200 OK`
```json
{
  "invitation": {
    "company_name": "株式会社サンプル",
    "name": "佐藤 一郎",
    "email": "sato@example.jp",
    "role": "owner",
    "store_name": null,
    "invited_by": "山田 太郎"
  }
}
```

**エラー:**
- `404` トークン無効
- `410` 有効期限切れ

---

#### `POST /api/v1/invitations/:token/accept`
招待受諾 + パスワード設定 + アカウント作成。

**Request:**
```json
{
  "password": "strongpass1",
  "password_confirmation": "strongpass1"
}
```

**Response:** `201 Created`
```json
{
  "message": "アカウントが有効化されました",
  "user": {
    "id": 5,
    "name": "佐藤 一郎",
    "email": "sato@example.jp",
    "role": "owner",
    "company": { "id": 1, "name": "株式会社サンプル" }
  },
  "auth_token": "jwt_or_session_token..."
}
```

**処理内容（トランザクション）:**
1. トークン検証（有効期限・ステータス）
2. `users` レコード作成
3. `invitations.status` を `accepted` に更新
4. `manager` の場合、`stores.manager_id` を更新
5. セッション/JWT発行

---

### 4.3 フローC: 店舗管理

#### `GET /api/v1/stores`
店舗一覧取得（既存API拡張）。

**認証:** 必須

**Response拡張:** 既存レスポンスに `manager` フィールドを追加
```json
{
  "stores": [
    {
      "id": 1,
      "name": "ラクミー本社",
      "brand": "飲食店向けBI_SaaS",
      "postal_code": "150-0042",
      "address": "東京都渋谷区宇田川町37-13 下田ビル3階",
      "manager": {
        "id": 2,
        "name": "鈴木 花子",
        "status": "active"
      }
    },
    {
      "id": 3,
      "name": "TEST店舗222",
      "brand": "飲食店向けBI_SaaS",
      "postal_code": "111-1111",
      "address": "沖縄",
      "manager": null
    }
  ]
}
```

---

#### `POST /api/v1/stores`
店舗追加（既存API拡張）。店長同時招待オプション追加。

**認証:** 必須（オーナーのみ）

**Request拡張:**
```json
{
  "store": {
    "name": "横浜店",
    "brand_id": 1,
    "postal_code": "220-0011",
    "address": "神奈川県横浜市西区高島2丁目"
  },
  "invite_manager": {
    "enabled": true,
    "name": "高橋 三郎",
    "email": "takahashi@example.jp"
  }
}
```

**処理内容（トランザクション）:**
1. `stores` レコード作成（既存処理）
2. `invite_manager.enabled` が `true` の場合:
   - `invitations` レコード作成（role: manager, store_id: 新店舗ID）
   - 招待メール送信（Sidekiqジョブ）

---

#### `POST /api/v1/stores/:id/invite_manager`
既存店舗への店長招待。

**認証:** 必須（オーナーのみ）

**Request:**
```json
{
  "name": "伊藤 四郎",
  "email": "ito@example.jp"
}
```

**Response:** `201 Created`

**バリデーション:**
- 対象店舗に既に店長が設定されていないこと
- 同一メールへの有効な招待が存在しないこと

---

#### `DELETE /api/v1/stores/:id/manager`
店長の解除（オーナーのみ）。

---

### 4.4 フローD: 従業員管理

#### `GET /api/v1/employees`
従業員一覧取得（既存API拡張）。

**認証:** 必須

**クエリパラメータ拡張:**
```
GET /api/v1/employees?store_id=3&employment_type=all&status=all&page=1&q=検索文字列
```

- `store_id`: 店舗フィルター（オーナーのみ `all` 可、店長は担当店舗に自動固定）

**レスポンス拡張:** 既存フィールドに `store` を追加（オーナー向け）
```json
{
  "employees": [
    {
      "id": 1,
      "name": "test",
      "employment_type": "業務委託",
      "transportation_unit": "日額",
      "transportation_cost": 1,
      "email": "nishikata+test2@wshoulder.com",
      "phone": "090-1234-1234",
      "status": "active",
      "store": { "id": 1, "name": "渋谷店" }
    }
  ],
  "meta": {
    "total_count": 25,
    "page": 1,
    "per_page": 10,
    "store_counts": {
      "all": 25,
      "1": 12,
      "2": 8,
      "3": 5
    }
  }
}
```

`store_counts` はオーナー向けのフィルタータブ用。

---

#### `POST /api/v1/employees`
従業員登録（既存API拡張）。

**認証:** 必須（オーナー or 担当店舗の店長）

**Request拡張:**
```json
{
  "employee": {
    "name": "吉田 七美",
    "phone": "090-1234-5678",
    "email": "yoshida@example.jp",
    "employee_code": "",
    "employment_type_id": 2,
    "transportation_unit": "daily",
    "transportation_cost": 500,
    "store_id": 3
  }
}
```

**権限チェック:**
- オーナー: 任意の `store_id` を指定可能
- 店長: 自分の担当店舗の `store_id` のみ指定可能（それ以外は `403`）

**処理拡張:**
1. 既存の従業員登録処理
2. 招待メール送信（Sidekiqジョブ） — パスワード設定リンク付き
3. `invitations` レコード作成（role: employee）

---

### 4.5 フローE: パスワード再設定（セルフサービス）

#### `POST /api/v1/password/reset`
パスワード再設定リクエスト。確認コードをメールに送信。

**認証:** 不要

**Request:**
```json
{
  "email": "yamada@sample-company.jp"
}
```

**Response:** `200 OK`
```json
{
  "message": "確認コードをメールに送信しました",
  "reset_token": "reset_abc123..."
}
```

**処理内容:**
1. メールアドレスでユーザーを検索
2. ユーザーが存在する場合: `password_resets` レコード作成（6桁コード、有効期限30分）
3. 確認コードメール送信（Sidekiqジョブ）
4. ユーザーが存在しない場合も同じレスポンスを返す（メール列挙攻撃防止）

**レート制限:** 同一IP 5回/時間、同一メール 3回/時間

---

#### `POST /api/v1/password/reset/verify`
確認コードの検証。

**Request:**
```json
{
  "reset_token": "reset_abc123...",
  "code": "739204"
}
```

**Response:** `200 OK`
```json
{
  "message": "確認されました",
  "setup_token": "pwsetup_xyz789..."
}
```

**エラーレスポンス:**
- `401` コード不一致（残り試行回数を返す）
- `410` 有効期限切れ
- `429` 試行回数超過（5回）

---

#### `POST /api/v1/password/reset/new`
新しいパスワードの設定。

**Request:**
```json
{
  "setup_token": "pwsetup_xyz789...",
  "password": "newsecurepass1",
  "password_confirmation": "newsecurepass1"
}
```

**Response:** `200 OK`
```json
{
  "message": "パスワードが再設定されました",
  "auth_token": "jwt_or_session_token..."
}
```

**処理内容（トランザクション）:**
1. `setup_token` の検証
2. パスワードのbcryptハッシュ化・更新
3. `users.password_changed_at` を更新
4. 既存セッションの無効化
5. 新しいセッション/JWT発行
6. パスワード変更通知メール送信（Sidekiqジョブ）

---

### 4.6 フローF: 管理者によるPW再発行

#### `POST /api/v1/users/:id/reset_password`
管理者が部下のパスワードをリセットし、一時パスワードをメール通知。

**認証:** 必須（オーナー: 全ユーザー / 店長: 担当店舗の従業員のみ）

**Response:** `200 OK`
```json
{
  "message": "一時パスワードをメールで通知しました",
  "user": {
    "id": 3,
    "name": "佐々木",
    "email": "sasaki@example.jp"
  }
}
```

**処理内容:**
1. 権限チェック（オーナー: 同一company_id / 店長: 担当store_idの従業員のみ）
2. 一時パスワード生成（12文字ランダム）
3. パスワードのbcryptハッシュ化・更新
4. `users.password_changed_at` を更新
5. 既存セッションの無効化
6. 一時パスワード通知メール送信（Sidekiqジョブ）
7. 次回ログイン時にパスワード変更を強制するフラグ設定

**権限マトリクス:**

| 実行者 | 対象 | 可否 |
|--------|------|------|
| オーナー | 店長 | ✅ |
| オーナー | 全店舗の従業員 | ✅ |
| 店長 | 担当店舗の従業員 | ✅ |
| 店長 | 他店舗の従業員 | ❌ |
| 店長 | オーナー | ❌ |
| 従業員 | 誰でも | ❌ |

---

### 4.7 招待メール再送信（詳細）

#### `POST /api/v1/administrators/invitations/:id/resend`
管理者招待メールの再送信。期限切れの場合は新しいトークンで再発行。

**認証:** 必須（オーナーのみ）

**Response:** `200 OK`
```json
{
  "message": "招待メールを再送信しました",
  "invitation": {
    "id": 5,
    "expires_at": "2026-03-05T09:15:00+09:00"
  }
}
```

**処理内容:**
1. 招待ステータスが `pending` または `expired` であること
2. 既に `accepted` の場合は `409 Conflict`
3. 期限切れの場合: 新しいトークン生成 + 期限延長（72時間）
4. まだ有効な場合: 同じトークンでメール再送信
5. 再送信間隔: 60秒以上

**レート制限:** 同一招待への再送信 5回/日

---

#### `POST /api/v1/stores/:id/manager_invitation/resend`
店長招待メールの再送信。

**認証:** 必須（オーナーのみ）

**処理内容:** `administrators/invitations/:id/resend` と同様

---

#### `POST /api/v1/employees/:id/invitation/resend`
従業員招待メールの再送信。

**認証:** 必須（オーナー: 全従業員 / 店長: 担当店舗の従業員のみ）

**処理内容:** `administrators/invitations/:id/resend` と同様

---

### 4.8 フローH: PW確認・変更（オーナー/店長）

#### `GET /api/v1/account/password`
現在のパスワード情報を取得（マスク済み）。

**認証:** 必須（オーナー or 店長）

**Response:** `200 OK`
```json
{
  "password_last_changed_at": "2026-02-15T10:30:00+09:00",
  "password_length": 13
}
```

---

#### `PUT /api/v1/account/password`
自分のパスワードを変更。

**認証:** 必須（オーナー or 店長）

**Request:**
```json
{
  "current_password": "securepass123",
  "new_password": "newsecurepass1",
  "new_password_confirmation": "newsecurepass1"
}
```

**Response:** `200 OK`
```json
{
  "message": "パスワードが変更されました"
}
```

**処理内容:**
1. 現在のパスワードの照合
2. 新しいパスワードのbcryptハッシュ化・更新
3. `users.password_changed_at` を更新
4. 他デバイスのセッション無効化
5. パスワード変更通知メール送信（Sidekiqジョブ）

**バリデーション:**
- `current_password`: 必須、現在のパスワードと一致
- `new_password`: 必須、8文字以上、現在のパスワードと異なること

---

### 4.9 フローI: PW再発行依頼（従業員）

#### `POST /api/v1/password/request_reset`
従業員が管理者（店長 or オーナー）にPW再発行を依頼。

**認証:** 不要

**Request:**
```json
{
  "email": "yoshida@example.jp",
  "reason": "パスワードを忘れた"
}
```

**Response:** `200 OK`
```json
{
  "message": "管理者にパスワード再発行を依頼しました"
}
```

**処理内容:**
1. メールアドレスでユーザーを検索（従業員のみ）
2. 該当店舗の店長に通知メール送信（店長がいない場合はオーナーに通知）
3. ユーザーが存在しない場合も同じレスポンスを返す（メール列挙攻撃防止）

**レート制限:** 同一メール 3回/日

---

## 5. メール設計

### 5.1 メールテンプレート一覧

| ID | メール名 | トリガー | 送信先 |
|----|---------|---------|--------|
| M-1 | サインアップ確認コード | POST /signup | 新規オーナー |
| M-2 | オーナー招待 | POST /administrators/invite (role=owner) | 被招待者 |
| M-3 | 店長招待 | POST /administrators/invite (role=manager) or POST /stores/:id/invite_manager | 被招待者 |
| M-4 | 従業員招待 | POST /employees（新規） | 被招待者 |
| M-5 | ウェルカムメール | サインアップ完了時 | 新規オーナー |
| M-6 | パスワード再設定確認コード | POST /password/reset | パスワード再設定要求者 |
| M-7 | パスワード変更完了通知 | POST /password/reset/new | パスワード変更対象者 |
| M-8 | 一時パスワード通知（管理者リセット） | POST /users/:id/reset_password | リセット対象者 |
| M-9 | PW再発行依頼通知 | POST /password/request_reset | 店長 or オーナー |

### 5.2 メール送信仕様

- **送信方法:** ActionMailer + Sidekiq（非同期）
- **送信元:** `noreply@timerecord.rakmy.jp`
- **確認コード（M-1）:**
  - 6桁数字
  - 有効期限: 30分
  - 再送信間隔: 60秒
- **招待リンク（M-2〜M-4）:**
  - URL: `https://timerecord.rakmy.jp/invite/accept?token={64文字トークン}`
  - 有効期限: 72時間
  - 1回限り使用
- **パスワード再設定コード（M-6）:**
  - 6桁数字
  - 有効期限: 30分
  - 再送信間隔: 60秒
- **一時パスワード通知（M-8）:**
  - 12文字ランダム一時パスワード
  - 次回ログイン時にパスワード変更を強制

---

## 6. セキュリティ要件

### 6.1 認証・パスワード

| 項目 | 仕様 |
|------|------|
| パスワード最小長 | 8文字 |
| パスワードハッシュ | bcrypt（has_secure_password） |
| セッション管理 | 既存の仕組みを踏襲 |

### 6.2 トークン

| 項目 | 仕様 |
|------|------|
| 招待トークン | `SecureRandom.urlsafe_base64(48)` — 64文字 |
| メール確認コード | `SecureRandom.random_number(10**6).to_s.rjust(6, '0')` |
| トークン保存 | DB保存（ハッシュ化推奨） |

### 6.3 レート制限

| エンドポイント | 制限 |
|---------------|------|
| POST /signup | 3回/IP/時間 |
| POST /signup/verify | 5回/トークン（試行回数） |
| POST /administrators/invite | 10回/会社/日 |
| POST /employees | 50回/会社/日 |
| 確認コード再送信 | 60秒間隔 |
| POST /password/reset | 5回/IP/時間、3回/メール/時間 |
| POST /users/:id/reset_password | 10回/管理者/日 |
| 招待メール再送信 | 5回/招待/日、60秒間隔 |

### 6.4 権限チェック

すべてのAPIで以下を検証:
1. ユーザーの `role` が操作に対して十分か
2. 店長の場合、操作対象の `store_id` が担当店舗か
3. 対象リソースが同一 `company_id` に属するか

---

## 7. バックエンド実装構成

### 7.1 新規ファイル構成（Rails標準）

```
app/
├── controllers/
│   └── api/
│       └── v1/
│           ├── signup_controller.rb          # A: サインアップ
│           ├── administrators_controller.rb   # B: 管理者一覧・招待
│           ├── invitations_controller.rb      # 招待受諾（共通）
│           └── password_resets_controller.rb  # E: PW再設定 + F: 管理者PW再発行
├── models/
│   ├── invitation.rb
│   ├── email_verification.rb
│   └── password_reset.rb
├── mailers/
│   ├── signup_mailer.rb                      # M-1, M-5
│   ├── invitation_mailer.rb                  # M-2, M-3, M-4
│   └── password_mailer.rb                    # M-6, M-7, M-8
├── jobs/
│   ├── send_verification_code_job.rb
│   ├── send_invitation_job.rb
│   ├── send_password_reset_code_job.rb
│   ├── send_temp_password_job.rb             # 管理者PW再発行通知
│   └── expire_invitations_job.rb             # 定期実行: 期限切れ処理
├── policies/                                 # Pundit等
│   ├── administrator_policy.rb
│   ├── store_policy.rb
│   └── employee_policy.rb
└── views/
    └── mailers/                              # メールテンプレート(Haml)
        ├── signup_mailer/
        │   ├── verification_code.html.haml
        │   └── welcome.html.haml
        ├── invitation_mailer/
        │   ├── owner_invitation.html.haml
        │   ├── manager_invitation.html.haml
        │   └── employee_invitation.html.haml
        └── password_mailer/
            ├── reset_code.html.haml
            ├── password_changed.html.haml
            └── temp_password.html.haml

db/
└── migrate/
    ├── YYYYMMDDHHMMSS_create_invitations.rb
    ├── YYYYMMDDHHMMSS_create_email_verifications.rb
    ├── YYYYMMDDHHMMSS_create_password_resets.rb
    ├── YYYYMMDDHHMMSS_add_role_to_users.rb
    └── YYYYMMDDHHMMSS_add_manager_id_to_stores.rb
```

### 7.2 既存ファイル変更

```
app/
├── controllers/
│   └── api/
│       └── v1/
│           ├── stores_controller.rb          # 拡張: manager情報返却、invite_managerアクション追加
│           └── employees_controller.rb       # 拡張: store_idフィルター、store_counts返却
├── models/
│   ├── user.rb                              # 拡張: role追加、has_many :invitations
│   ├── store.rb                             # 拡張: belongs_to :manager（optional）
│   └── company.rb                           # 拡張: has_many :invitations
└── views/
    └── (フロントエンド変更は別途)
```

### 7.3 定期ジョブ

| ジョブ | スケジュール | 内容 |
|--------|------------|------|
| `ExpireInvitationsJob` | 毎時 | 期限切れ招待の status を `expired` に更新 |

---

## 8. フロントエンド変更一覧

### 8.1 新規画面

| 画面 | URL | テンプレート |
|------|-----|-------------|
| サインアップ（入力） | /signup | signup/new.html.haml |
| サインアップ（メール認証） | /signup/verify | signup/verify.html.haml |
| サインアップ（PW設定） | /signup/password | signup/password.html.haml |
| サインアップ（完了） | /signup/complete | signup/complete.html.haml |
| 招待受諾（PW設定） | /invite/accept | invitations/accept.html.haml |
| PW再設定（メール入力） | /password/reset | password_resets/new.html.haml |
| PW再設定（確認コード） | /password/reset/verify | password_resets/verify.html.haml |
| PW再設定（新PW設定） | /password/reset/new | password_resets/edit.html.haml |
| PW確認・変更 | /dashboard/account/password | account/password.html.haml |
| PW再発行依頼 | /dashboard/account/password_request | account/password_request.html.haml |

### 8.2 既存画面の変更

| 画面 | 変更内容 |
|------|---------|
| ログイン (`/login`) | 「新規登録はこちら」リンク追加、「パスワードを忘れた」→ PW再設定フローへ遷移 |
| 店舗情報 (`/dashboard/stores`) | テーブルに「店長」列追加、各行に「店長を招待」ボタン、招待中店長に「再送信」ボタン |
| 店舗登録モーダル | 下部に「店長を同時に招待」セクション追加 |
| 従業員一覧 (`/dashboard/employees`) | オーナー向け: 店舗フィルタータブ + 所属店舗列追加 + 招待中行に「再送信」ボタン + アクティブ行に「PW再発行」ボタン |
| 従業員登録モーダル | オーナー向け: 上部に「所属店舗」セレクトボックス追加 |
| 管理者一覧 (`/dashboard/administrators`) | 一覧テーブル + 「管理者を招待」ボタン + 招待モーダル + 招待中行に「再送信」ボタン + アクティブ行に「PW再発行」ボタン |

---

## 9. 実装フェーズ

### Phase 1（P0 — 最優先）
1. DBマイグレーション（invitations, email_verifications, password_resets, users.role/force_password_change, stores.manager_id）
2. 招待モデル + メール送信基盤
3. フローB: 管理者一覧 + 招待 + 受諾（既存オーナーからの手動登録を自動化）
4. フローA: セルフサインアップ（会社+オーナー登録）
5. フローE: パスワード再設定（セルフサービス）
6. フローF: 管理者によるPW再発行
7. フローG: 招待メール再送信

### Phase 2（P1）
8. フローC: 店舗テーブルへの店長列追加 + 店長招待
9. 店舗登録モーダルへの店長同時招待機能
10. フローD: 従業員一覧の店舗フィルター + 所属店舗列
11. 従業員登録モーダルへの所属店舗セレクト追加
12. ログイン画面改修

### Phase 3（P2 — 将来）
13. オーナー権限の移譲
14. 店長の交代
15. 招待の一括送信
16. CSVインポートによる従業員一括登録

---

## 10. テスト要件

### 10.1 ユニットテスト（RSpec）

- `Invitation` モデル: バリデーション、トークン生成、期限切れ判定
- `EmailVerification` モデル: コード生成、試行回数、期限切れ
- `PasswordReset` モデル: コード生成、試行回数、期限切れ、一時パスワード生成
- `User` モデル: role判定メソッド、権限チェック、force_password_change
- 各Policyクラス: 権限マトリクスの全パターン

### 10.2 リクエストテスト

- 全APIエンドポイントの正常系・異常系
- 権限チェック（オーナー以外がオーナー専用APIを叩いた場合の403）
- 店長のスコープ制限（担当外店舗へのアクセスが403）
- レート制限の動作確認
- トークン有効期限の動作確認

### 10.3 統合テスト

- サインアップ完全フロー（入力 → メール認証 → PW設定 → ログイン）
- 招待完全フロー（招待送信 → メール受信 → PW設定 → ログイン）
- パスワード再設定フロー（メール入力 → 確認コード → 新PW設定 → ログイン）
- 管理者PW再発行フロー（PW再発行 → メール受信 → ログイン → 強制PW変更）
- 招待メール再送信フロー（再送信 → メール受信 → PW設定）
- 店舗追加 + 店長同時招待フロー
- 従業員登録フロー（オーナーから / 店長から）

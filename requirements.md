# ラクミー タイムレコード — オーナー登録・店舗管理・権限管理 要件書

> **バージョン:** v1.5
> **作成日:** 2026-03-02
> **最終更新:** 2026-04-08
> **モック:** https://suguru789987.github.io/rakmy-owner-registration-mock/
> **モックソース:** https://github.com/suguru789987/rakmy-owner-registration-mock/blob/main/index.html
> **対象リポジトリ:** rakmy/rakmy_server (Rails 7.2 / MySQL / Sidekiq / Redis)

---

## 1. 概要

### 1.1 目的
現在開発者がCLIで手動実行しているオーナー登録を、セルフサービスUI + 直接登録フローとして自動化する。
加えて、オーナー → 管理者 → 従業員の権限階層に基づく管理機能を実装する。

### 1.2 権限階層

```
オーナー / 本部管理者（全権管理者）
  ├── 全店舗・全機能にアクセス
  ├── 店舗の追加・編集・削除
  ├── 管理者の直接登録（初期PW設定→口頭共有）
  ├── 従業員の直接登録（全店舗、招待メール自動送信）
  └── 給与・経営管理・外部連携

管理者（店舗管理者）※任意配置、オーナーが直接登録
  ├── 担当店舗のみ管理
  ├── 従業員の登録・管理（担当店舗）
  ├── 勤怠承認
  ├── 給与・経営管理（担当店舗）
  ├── 外部連携設定
  └── 担当店舗従業員のPW再発行

従業員
  ├── 打刻操作
  └── 自分の勤怠確認
```

> **名称ルール:** 本仕様書・モックでは「管理者」で統一する。「店長」は使用しない。

### 1.3 権限マトリクス

| 操作 | オーナー / 本部管理者 | 管理者 | 従業員 |
|------|---------------------|--------|--------|
| 店舗の追加・編集・削除 | ✅ | ❌ | ❌ |
| 店舗情報の編集（担当店舗） | ✅ 全店舗 | ✅ 担当のみ | ❌ |
| オーナー招待 | ✅ | ❌ | ❌ |
| 管理者の登録 | ✅ | ❌ | ❌ |
| 従業員の招待・登録 | ✅ 全店舗 | ✅ 担当店舗のみ | ❌ |
| 従業員情報の閲覧・編集 | ✅ 全店舗 | ✅ 担当店舗のみ | ❌ |
| 勤怠データ閲覧 | ✅ 全店舗 | ✅ 担当店舗のみ | ✅ 自分のみ |
| 自分のPW確認・変更 | ✅ | ✅ | ❌ |
| PW再設定（パスワードを忘れた） | ✅ | ✅ | ✅ |
| 従業員のPW再発行 | ✅ 全店舗の従業員 | ✅ 担当店舗従業員のみ | ❌ |
| 招待メールの再送信 | ✅ | ✅ 担当店舗従業員のみ | ❌ |
| 打刻データの修正 | ✅ 全店舗 | ✅ 担当店舗のみ | ❌ |
| 打刻操作 | ❌ | ❌ | ✅ |
| 給与・経営管理 | ✅ 全店舗 | ✅ 担当店舗のみ | ❌ |
| データインポート | ✅ 全店舗 | ✅ 担当店舗のみ | ❌ |
| データエクスポート | ✅ 全店舗 | ✅ 担当店舗のみ | ❌ |
| 外部連携設定 | ✅ | ✅ | ❌ |

---

## 2. 機能一覧とモック画面対応

| ID | 機能 | モック画面 | 優先度 |
|----|------|-----------|--------|
| A | 新規会社オーナー登録（セルフサインアップ） | A-1〜A-4 | P0 |
| B | 管理者の直接登録 | D-1, D-1E, D-2 | P0 |
| C | 店舗管理 | B-1, B-2 | P1 |
| D | 従業員管理（オーナー/管理者視点） | C-1, C-1E, C-2, C-2E, C-3, C-3M, C-4, C-5 | P0/P1 |
| E | PW再設定（パスワードを忘れた） | login, E-1〜E-3 | P0 |
| F | 管理者による従業員PW再発行 | C-1, C-2 | P0 |
| G | 従業員招待メール再送信 | C-1, C-2 | P0 |
| H | ログイン画面改修 | login | P1 |

**既存画面の変更:**

| 画面ID | 既存画面 | 変更内容 |
|--------|---------|---------|
| F-1 | アカウント設定 ー パスワード | 全ユーザーがサイドバー「アカウント設定」からPW確認・変更。既存画面の仕様参照用 |

---

## 2.5 従業員ステータス定義

従業員一覧に表示されるステータスは全3種類。招待の状態とユーザーアカウントの状態を1つのバッジとして表示する。

### 2.5.1 各ステータスの定義

#### 招待中

| 項目 | 内容 |
|---|---|
| いつなるか | オーナー/管理者が従業員を登録した直後（招待メールが自動送信される） |
| DBの状態 | `invitations.status = pending`、`users` レコードは未作成 |
| 招待リンク | 有効（72時間以内） |
| 次の遷移先 | → 有効（PW設定完了）/ → 期限切れ（72時間経過） |

#### 有効

| 項目 | 内容 |
|---|---|
| いつなるか | 従業員が招待メールのリンクからPW設定を完了し、アカウントが作成された |
| DBの状態 | `invitations.status = accepted`、`users` レコードあり（ログイン可能） |
| 次の遷移先 | なし（削除のみ。削除した場合は一覧から消える） |

#### 期限切れ

| 項目 | 内容 |
|---|---|
| いつなるか | 招待メール送信後、72時間以内にPW設定が行われなかった（`ExpireInvitationsJob` が毎時更新） |
| DBの状態 | `invitations.status = expired`、`users` レコードは未作成 |
| 招待リンク | 無効（エラー画面「招待リンクが無効です」が表示される） |
| 次の遷移先 | → 招待中（管理者が「再送」で再招待 → 新トークン発行） |

### 2.5.2 状態遷移図

```
従業員登録
    │
    ▼
 [招待中] ──── 72時間経過 ───→ [期限切れ]
    │                              │
    │ PW設定完了            再送   │
    ▼                     ←────────┘
 [有効]

 ※ 招待中・期限切れの「削除」→ 招待レコード削除（一覧から消える）
 ※ 有効の「削除」→ アカウント削除（一覧から消える）
```

### 2.5.3 ステータス別バッジ表示

| ステータス | バッジ色 | カラーコード |
|---|---|---|
| 招待中 | オレンジ | 背景 `#fff8e1` / 文字 `#f0960a` |
| 有効 | 緑 | 背景 `#e8f8f5` / 文字 `#2ec4a5` |
| 期限切れ | 赤 | 背景 `#fce4ec` / 文字 `#c62828` |

### 2.5.4 ステータス別の操作列（PW操作・編集・削除）

| ステータス | PW操作列 | 編集ボタン | 削除ボタン |
|---|---|---|---|
| 招待中 | 「再送」ボタン（アウトラインスタイル） | 編集モーダル（メール変更可、変更時に招待再送） | 確認「○○への招待を取り消しますか？」→ 招待レコード削除 |
| 有効 | 「PW再発行」ボタン | 編集モーダル（メール変更不可） | 確認「○○を削除しますか？この操作は取り消せません。」→ アカウント削除 |
| 期限切れ | 「再送」ボタン（アウトラインスタイル） | 編集モーダル（メール変更可、変更時に招待再送） | 確認「○○への招待を取り消しますか？」→ 招待レコード削除 |

> **注意:** 上記の操作列はオーナー視点・管理者視点の両方で共通。ただし管理者は担当店舗の従業員のみ操作可能。

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
  status ENUM('pending', 'accepted', 'expired') NOT NULL DEFAULT 'pending',
  store_id BIGINT UNSIGNED,               -- 従業員の場合の所属店舗
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

> **廃止:** `email_verifications` テーブル、`password_resets` テーブルは不要。メール認証・パスワード管理はClerkが担当する。

### 3.2 既存テーブル変更

#### `users` テーブルへのカラム追加
```sql
ALTER TABLE users
  ADD COLUMN clerk_user_id VARCHAR(255) UNIQUE,  -- ClerkのユーザーID（user_xxxxx）
  ADD COLUMN role ENUM('owner', 'manager', 'employee') NOT NULL DEFAULT 'employee',
  ADD COLUMN invitation_id BIGINT UNSIGNED;
```

> **従業員と店舗の紐付け:** 既存テーブル `user_store_assignments` で管理。従業員の所属店舗変更（異動）はこのテーブルのレコードを更新する。将来的にヘルプ勤務識別（所属店舗 ≠ 打刻店舗）が必要になった場合、このテーブルを基盤として親システム（ラクミー本体）で対応予定。

#### `stores` テーブル
変更なし（管理者と店舗の紐付けは中間テーブル `manager_stores` で管理）。

#### `manager_stores`（管理者-店舗紐付け）
```sql
CREATE TABLE manager_stores (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  user_id BIGINT UNSIGNED NOT NULL,
  store_id BIGINT UNSIGNED NOT NULL,
  created_at DATETIME NOT NULL,
  updated_at DATETIME NOT NULL,

  UNIQUE INDEX idx_manager_stores_user_store (user_id, store_id),
  FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
  FOREIGN KEY (store_id) REFERENCES stores(id) ON DELETE CASCADE
);
```

> **注意:** 既存の `users` / `stores` / `companies` テーブル構造は実際のスキーマを確認のうえ調整すること。

---

## 4. API設計

### 4.1 フローA: 新規会社オーナー登録

> Clerkのサインアップフローを利用。メール認証・パスワード設定はClerkが処理する。
> フロントエンドはClerk Components（`<SignUp />`）またはClerk API を使用。

**フロー:**
1. ユーザーがサインアップ画面で会社名・氏名・メール・パスワードを入力
2. Clerkがメール認証コードを送信（Clerk管理）
3. ユーザーが認証コードを入力 → Clerkがユーザーを作成
4. Clerkの `user.created` Webhookでバックエンドに通知
5. バックエンドが `companies` + `users` レコードを作成

#### `POST /api/v1/webhooks/clerk`（Clerk Webhook）
Clerkからの `user.created` イベントを受信し、会社・ユーザーレコードを作成。

**処理内容:**
1. Webhook署名の検証（`svix` ライブラリ）
2. イベント種別が `user.created` の場合:
   - Clerkユーザー情報からメール・名前を取得
   - `companies` レコード作成（会社名はClerkの `unsafe_metadata.company_name` から取得）
   - `users` レコード作成（`clerk_user_id` を紐付け、role: owner）
3. ウェルカムメール送信（Sidekiqジョブ）

> **会社名の受け渡し:** サインアップ時にClerkの `unsafeMetadata` に `{ company_name: "株式会社サンプル" }` をセットし、Webhook経由でバックエンドに渡す。

#### `POST /api/v1/signup/company`（代替案: Webhook不使用）
Clerkサインアップ完了後、フロントエンドからバックエンドに会社情報を登録。

**認証:** 必須（Clerkセッション）

**Request:**
```json
{
  "company_name": "株式会社サンプル"
}
```

**Response:** `201 Created`
```json
{
  "message": "登録が完了しました",
  "company": { "id": 1, "name": "株式会社サンプル" },
  "user": {
    "id": 1,
    "name": "山田 太郎",
    "email": "yamada@sample-company.jp",
    "role": "owner"
  }
}
```

**処理内容:**
1. Clerkセッションからユーザー情報を取得（`clerk_user_id`, email, name）
2. `companies` レコード作成
3. `users` レコード作成（`clerk_user_id` を紐付け、role: owner）

---

### 4.2 フローB: 管理者の直接登録

> 管理者（オーナー/管理者）は招待フローを使わず、オーナーが直接登録する。初期パスワードを設定し、口頭やチャット等で管理者に共有する。

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
      "id": 2,
      "name": "鈴木 花子",
      "email": "suzuki@sample.jp",
      "role": "manager",
      "stores": "渋谷店",
      "status": "active",
      "last_login_at": "2026-03-01T14:30:00+09:00",
      "is_self": false
    }
  ]
}
```

---

#### `POST /api/v1/administrators`
管理者を直接登録（オーナーが初期PW設定）。

**認証:** 必須（オーナーのみ）

**Request:**
```json
{
  "name": "鈴木 花子",
  "email": "suzuki@sample.jp",
  "role": "manager",
  "password": "initialPass123",
  "password_confirmation": "initialPass123",
  "store_ids": [1]
}
```

**Response:** `201 Created`
```json
{
  "message": "管理者を登録しました",
  "administrator": {
    "id": 2,
    "name": "鈴木 花子",
    "email": "suzuki@sample.jp",
    "role": "manager",
    "stores": ["渋谷店"]
  }
}
```

**バリデーション:**
- `email`: 必須、メール形式、同一会社内で未使用
- `name`: 必須
- `role`: `owner` or `manager`
- `password`: 必須、8文字以上
- `store_ids`: `manager` の場合は必須（1つ以上）、`owner` の場合は不要（全店舗アクセス）

**処理内容:**
1. 権限チェック（オーナーのみ）
2. 重複チェック（同一メールが未使用であること）
3. **Clerk Backend API** でユーザーを作成（`POST /v1/users`、email + password）
4. `users` レコード作成（`clerk_user_id` を紐付け、role）
5. `manager` の場合、担当店舗との紐付けを作成

> **パスワード共有:** オーナーが初期パスワードを口頭・チャット等で管理者に伝える。管理者は初回ログイン後、Clerkのパスワード変更機能で変更可能。パスワードを忘れた場合は、ログイン画面の「パスワードを忘れた」からClerkのPW再設定フローを利用する。

---

#### `PUT /api/v1/administrators/:id`
管理者情報の編集。

**認証:** 必須（オーナーのみ）

**Request:**
```json
{
  "name": "鈴木 花子",
  "email": "suzuki-new@sample.jp",
  "store_ids": [1, 2]
}
```

**Response:** `200 OK`

---

#### `DELETE /api/v1/administrators/:id`
管理者の削除。

**認証:** 必須（オーナーのみ）

**処理内容:**
1. 権限チェック（オーナーのみ、自分自身は削除不可）
2. **Clerk Backend API** でユーザーを削除（`DELETE /v1/users/{clerk_user_id}`）
3. `users` レコード削除
4. 担当店舗との紐付けを削除
5. 一覧から消える

**確認ダイアログ:** 「○○を削除しますか？この操作は取り消せません。」

**Response:** `200 OK`

---

### 4.3 フローC: 店舗管理

#### `GET /api/v1/stores`
店舗一覧取得（既存API）。

**認証:** 必須

**Response:** 既存レスポンスと同様
```json
{
  "stores": [
    {
      "id": 1,
      "name": "ラクミー本社",
      "brand": "飲食店向けBI_SaaS",
      "postal_code": "150-0042",
      "address": "東京都渋谷区宇田川町37-13 下田ビル3階"
    },
    {
      "id": 3,
      "name": "TEST店舗222",
      "brand": "飲食店向けBI_SaaS",
      "postal_code": "111-1111",
      "address": "沖縄"
    }
  ]
}
```

---

#### `POST /api/v1/stores`
店舗追加（既存API）。

**認証:** 必須（オーナーのみ）

**Request:**
```json
{
  "store": {
    "name": "横浜店",
    "brand_id": 1,
    "postal_code": "220-0011",
    "address": "神奈川県横浜市西区高島2丁目"
  }
}
```

**処理内容（トランザクション）:**
1. `stores` レコード作成（既存処理）

> **注意:** 管理者の店舗割り当ては管理者一覧（フローB）の登録・編集で行う。

---

### 4.4 フローD: 従業員管理

#### `GET /api/v1/employees`
従業員一覧取得（既存API拡張）。

**認証:** 必須

**クエリパラメータ拡張:**
```
GET /api/v1/employees?store_id=3&employment_type=all&status=all&page=1&q=検索文字列
```

- `store_id`: 店舗フィルター（オーナーのみ `all` 可、管理者は担当店舗に自動固定）

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

**認証:** 必須（オーナー or 担当店舗の管理者）

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
- 管理者: 自分の担当店舗の `store_id` のみ指定可能（それ以外は `403`）

**処理拡張:**
1. 既存の従業員登録処理
2. 招待メール送信（Sidekiqジョブ） — パスワード設定リンク付き
3. `invitations` レコード作成（role: employee）

---

#### 従業員の編集・削除のステータス別動作

##### 編集

| ステータス | 動作 | 備考 |
|---|---|---|
| 招待中 | 名前・メール等を更新可能。**メールアドレス変更時は旧トークンを無効化し、新しい招待メールを自動再送** | 招待レコードのemail・tokenを更新 |
| 有効 | 既存の従業員編集処理（情報の上書き更新）。**メールアドレスは変更不可** | ログイン情報のため |
| 期限切れ | 招待中と同じ（編集＋自動再送） | 新トークン・新期限で再発行 |

##### 所属店舗の変更（異動）

従業員編集モーダルで所属店舗を変更することで、店舗間の異動が可能。

| 項目 | 仕様 |
|------|------|
| オーナー | 全店舗間で異動可能 |
| 管理者 | 担当店舗間のみ異動可能（担当外への異動は不可） |
| 交通費アラート | 所属店舗を変更すると「交通費（日額）を新しい店舗に合わせて更新してください」の警告を表示 |
| DB更新 | `user_store_assignments` のレコードを更新（旧店舗 → 新店舗） |
| 給与計算 | 打刻データベースで計算。異動前の打刻は旧店舗、異動後の打刻は新店舗に計上（按分不要） |
| ヘルプ勤務（将来構想） | 所属店舗以外での勤務を「ヘルプ勤務」として識別し、ヘルプ先店舗の勤務時間・時給で計上する機能。`user_store_assignments`（所属店舗）と打刻店舗の差分で識別する想定。タイムレコードには現在未実装、将来的に検討 |

##### 削除

| ステータス | 動作 | 確認ダイアログ |
|---|---|---|
| 招待中 | 招待レコード削除（一覧から消える） | 「○○への招待を取り消しますか？」 |
| 有効 | アカウント削除（既存の従業員削除処理、一覧から消える） | 「○○を削除しますか？この操作は取り消せません。」 |
| 期限切れ | 招待レコード削除（一覧から消える） | 「○○への招待を取り消しますか？」 |

---

#### `PUT /api/v1/employees/:id`
従業員情報の編集（既存API拡張）。

**認証:** 必須（オーナー or 担当店舗の管理者）

**Request:**
```json
{
  "employee": {
    "name": "佐々木",
    "phone": "090-0000-0000",
    "email": "sasaki-new@example.jp",
    "employee_code": "E001",
    "employment_type_id": 2,
    "transportation_unit": "daily",
    "transportation_cost": 500,
    "store_id": 3
  }
}
```

**処理内容（ステータス別）:**

**招待中・期限切れの場合:**
1. 権限チェック
2. `invitations` レコードの情報を更新（name, email 等）
3. メールアドレスが変更された場合:
   - 旧トークンを無効化
   - 新しいトークン（64文字）を生成、有効期限を72時間で再設定
   - 新しいメールアドレスに招待メールを自動送信（Sidekiqジョブ）
4. メールアドレスが変更されていない場合:
   - 情報のみ更新（招待メールは再送しない）

**承諾済み（有効）の場合:**
1. 権限チェック
2. 既存の従業員編集処理（`users` レコード更新）
3. **メールアドレスの変更は不可**（リクエストに含まれていても無視 or `422` エラー）

> **運用ルール:** 有効な従業員のメールアドレス変更が必要な場合（結婚等）は、該当従業員を削除して新しいメールアドレスで再登録する。

**エラー:**
- `403` 権限不足（管理者が他店舗の従業員を編集しようとした場合）
- `422` 有効ステータスのユーザーのメールアドレス変更

**Response:** `200 OK`
```json
{
  "message": "従業員情報を更新しました",
  "employee": {
    "id": 4,
    "name": "佐々木",
    "email": "sasaki-new@example.jp",
    "status": "pending",
    "invitation_resent": true
  }
}
```

---

#### `DELETE /api/v1/employees/:id`
従業員の削除（既存API拡張）。

**認証:** 必須（オーナー or 担当店舗の管理者）

**処理内容（ステータス別）:**

**招待中・期限切れの場合:**
1. 権限チェック
2. `invitations` レコードを物理削除
3. 一覧から消える（再度登録が必要）

**有効の場合:**
1. 権限チェック
2. 既存の従業員削除処理（`users` レコード削除）
3. 関連する `invitations` レコードも削除
4. 一覧から消える（再度登録が必要）

**エラー:**
- `403` 権限不足

**Response:** `200 OK`
```json
{
  "message": "従業員を削除しました"
}
```

---

### 4.5 フローE: PW再設定（パスワードを忘れた）

> **Clerkが全処理を担当。** バックエンドAPIは不要。
> フロントエンドはClerk Components（`<SignIn />` の forgotPassword）またはClerk API を使用。

**フロー:**
1. ログイン画面の「パスワードを忘れた」リンクをクリック
2. メールアドレスを入力 → Clerkが確認コードをメールに送信
3. 確認コードを入力 → Clerk側で検証
4. 新しいパスワードを設定 → Clerk側で更新
5. ログイン画面にリダイレクト

> **バックエンドの関与なし。** メール送信・コード検証・パスワード更新・セッション管理はすべてClerkが処理する。

---

### 4.6 フローF: 管理者による従業員PW再発行

> **注意:** 管理者（オーナー/管理者）自身のPW再設定は、ログイン画面の「パスワードを忘れた」（フローE）から行う。本フローは従業員のPW再発行のみを対象とする。

#### `POST /api/v1/users/:id/reset_password`
管理者が従業員のパスワードをリセットし、一時パスワードをメール通知。

**認証:** 必須（オーナー: 全店舗の従業員 / 管理者: 担当店舗の従業員のみ）

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
1. 権限チェック（オーナー: 同一company_idの従業員のみ / 管理者: 担当store_idの従業員のみ）
2. 一時パスワード生成（12文字ランダム）
3. **Clerk Backend API** でパスワードを更新（`PATCH /v1/users/{clerk_user_id}`、`password` パラメータ）
4. 一時パスワード通知メール送信（Sidekiqジョブ）

**権限マトリクス:**

| 実行者 | 対象 | 可否 |
|--------|------|------|
| オーナー | 全店舗の従業員 | ✅ |
| オーナー | 他のオーナー/管理者 | ❌（自分でPW再設定フローを利用） |
| 管理者 | 担当店舗の従業員 | ✅ |
| 管理者 | 他店舗の従業員 | ❌ |
| 従業員 | 誰でも | ❌ |

---

### 4.7 フローG: 従業員招待メール再送信

> 招待メールの再送信は従業員のみが対象。管理者は招待フローを使用しないため再送信の対象外。

#### `POST /api/v1/employees/:id/invitation/resend`
従業員招待メールの再送信。期限切れの場合は新しいトークンで再発行。

**認証:** 必須（オーナー: 全従業員 / 管理者: 担当店舗の従業員のみ）

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

### 4.8 既存画面: PW確認・変更（F-1 アカウント設定 ー パスワード）

> **Clerkが処理を担当。** フロントエンドはClerk Components（`<UserProfile />`）またはClerk API を使用。
> サイドバー「アカウント設定」からアクセスし、Clerkのパスワード変更UIを表示する。
> バックエンドAPIは不要。


---

## 5. メール設計

### 5.1 メールテンプレート一覧

| ID | メール名 | トリガー | 送信先 |
|----|---------|---------|--------|
| M-1 | サインアップ確認コード | Clerkが自動送信 | 新規オーナー |
| M-4 | 従業員招待 | POST /employees（新規） | 被招待者 |
| M-5 | ウェルカムメール | サインアップ完了時 | 新規オーナー |
| M-6 | パスワード再設定確認コード | Clerkが自動送信 | パスワード再設定要求者 |
| M-7 | パスワード変更完了通知 | Clerkが自動送信 | パスワード変更対象者 |
| M-8 | 一時パスワード通知（管理者リセット） | POST /users/:id/reset_password | リセット対象者 |

> **注意:** M-2（オーナー招待）、M-3（管理者招待）は廃止。管理者は直接登録のためメール招待なし。

### 5.2 メール送信仕様

- **送信方法:** ActionMailer + Sidekiq（非同期）
- **送信元:** `noreply@timerecord.rakmy.jp`
- **確認コード（M-1）:**
  - 6桁数字
  - 有効期限: 30分
  - 再送信間隔: 60秒
- **招待リンク（M-4 従業員招待のみ）:**
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

> **認証基盤: Clerk** — ユーザーの作成・認証・セッション管理・パスワード管理はすべてClerkが担当する。バックエンドは Clerk が発行するJWT（セッショントークン）を検証してユーザーを特定する。

### 6.1 認証・パスワード

| 項目 | 仕様 |
|------|------|
| 認証基盤 | Clerk |
| パスワード最小長 | 8文字（Clerk側で設定） |
| パスワードハッシュ | Clerk管理（バックエンドでは保持しない） |
| セッション管理 | Clerkセッション（JWT） |
| バックエンドの認証 | Clerk JWTを `Authorization: Bearer` ヘッダーで受け取り、Clerk SDK で検証 |
| ユーザー識別 | `clerk_user_id` でClerkユーザーとDBユーザーを紐付け |

### 6.2 トークン

| 項目 | 仕様 |
|------|------|
| 認証トークン | Clerk JWT（バックエンドでは生成しない） |
| 従業員招待トークン | `SecureRandom.urlsafe_base64(48)` — 64文字 |
| メール確認コード | Clerk管理（サインアップ・PW再設定） |
| トークン保存 | 招待トークンのみDB保存（ハッシュ化推奨） |

### 6.3 レート制限

| エンドポイント | 制限 | 管理 |
|---------------|------|------|
| サインアップ | Clerk側で制御 | Clerk |
| メール認証コード | Clerk側で制御 | Clerk |
| POST /api/v1/signup/company | 3回/IP/時間 | バックエンド |
| POST /api/v1/administrators | 10回/会社/日 | バックエンド |
| POST /api/v1/employees | 50回/会社/日 | バックエンド |
| PW再設定（パスワードを忘れた） | Clerk側で制御 | Clerk |
| POST /api/v1/users/:id/reset_password | 10回/管理者/日 | バックエンド |
| POST /api/v1/employees/:id/invitation/resend | 5回/招待/日、60秒間隔 | バックエンド |

### 6.4 権限チェック

すべてのAPIで以下を検証:
1. ユーザーの `role` が操作に対して十分か
2. 管理者の場合、操作対象の `store_id` が担当店舗か
3. 対象リソースが同一 `company_id` に属するか

---

## 7. バックエンド実装構成

### 7.1 新規ファイル構成（Rails標準）

```
app/
├── controllers/
│   └── api/
│       └── v1/
│           ├── webhooks/
│           │   └── clerk_controller.rb        # Clerk Webhook受信
│           ├── signup_controller.rb           # A: 会社登録（Clerk認証後）
│           ├── administrators_controller.rb    # B: 管理者一覧・直接登録
│           ├── invitations_controller.rb       # 従業員招待受諾
│           └── employees_controller.rb        # D: 従業員管理 + F: PW再発行
├── models/
│   └── invitation.rb
├── services/
│   └── clerk_service.rb                       # Clerk Backend API ラッパー
├── mailers/
│   ├── welcome_mailer.rb                      # M-5: ウェルカムメール
│   ├── invitation_mailer.rb                   # M-4: 従業員招待
│   └── password_mailer.rb                     # M-8: 一時PW通知
├── jobs/
│   ├── send_invitation_job.rb
│   ├── send_temp_password_job.rb              # 管理者PW再発行通知
│   └── expire_invitations_job.rb              # 定期実行: 期限切れ処理
├── policies/                                  # Pundit等
│   ├── administrator_policy.rb
│   ├── store_policy.rb
│   └── employee_policy.rb
├── middleware/
│   └── clerk_auth.rb                          # Clerk JWT検証ミドルウェア
└── views/
    └── mailers/                               # メールテンプレート(Haml)
        ├── welcome_mailer/
        │   └── welcome.html.haml
        ├── invitation_mailer/
        │   └── employee_invitation.html.haml
        └── password_mailer/
            └── temp_password.html.haml

db/
└── migrate/
    ├── YYYYMMDDHHMMSS_create_invitations.rb
    ├── YYYYMMDDHHMMSS_add_clerk_user_id_to_users.rb
    ├── YYYYMMDDHHMMSS_add_role_to_users.rb
    └── YYYYMMDDHHMMSS_create_manager_stores.rb
```

### 7.2 既存ファイル変更

```
app/
├── controllers/
│   └── api/
│       └── v1/
│           ├── stores_controller.rb          # 既存（変更少）
│           └── employees_controller.rb       # 拡張: store_idフィルター、store_counts返却
├── models/
│   ├── user.rb                              # 拡張: role追加、has_many :invitations
│   ├── store.rb                             # 既存（変更なし）
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
| サインアップ（アカウント作成） | /signup | Clerk Components（`<SignUp />`） |
| サインアップ（メール認証） | /signup/verify | Clerk Components（自動遷移） |
| サインアップ（会社登録） | /signup/company | signup/company.html.haml |
| サインアップ（完了） | /signup/complete | signup/complete.html.haml |
| 従業員招待受諾（PW設定） | /invite/accept | invitations/accept.html.haml |
| 無効な招待リンク（エラー） | /invite/accept?token=invalid | invitations/invalid.html.haml |
| PW再設定 | /password/reset | Clerk Components（`<SignIn />` forgotPassword） |
| PW確認・変更 | /dashboard/account/password | Clerk Components（`<UserProfile />`） |

### 8.2 既存画面の変更

| 画面 | 変更内容 |
|------|---------|
| ログイン (`/login`) | 「新規登録はこちら」リンク追加、「パスワードを忘れた」→ PW再設定フローへ遷移 |
| 店舗情報 (`/dashboard/stores`) | 既存画面の延長（変更少）。操作列に「QR」「編集」「削除」ボタン |
| 従業員一覧 オーナー視点 (`/dashboard/employees`) | 店舗フィルタータブ + 所属店舗列追加 + 全3ステータス（有効/招待中/期限切れ）の行を表示 + ステータス別PW操作・編集・削除ボタンの出し分け |
| 従業員一覧 管理者視点 (`/dashboard/employees`) | 担当店舗の従業員のみ表示 + 全3ステータスの行を表示 + オーナー視点と同じステータス別ボタン出し分け |
| 従業員編集モーダル（オーナー・有効） | メールアドレス変更不可（disabled）。名前・電話・雇用区分・交通費・所属店舗（全店舗）を編集可能 |
| 従業員編集モーダル（オーナー・招待中/期限切れ） | メールアドレス変更可能。変更時は旧招待リンク無効化＋新招待メール自動再送の注意書きを表示。所属店舗は全店舗から選択可能 |
| 従業員編集モーダル（管理者・有効） | オーナー版と同様。所属店舗は担当店舗のみのセレクトボックス（担当店舗間で異動可能） |
| 従業員編集モーダル（管理者・招待中/期限切れ） | オーナー版と同様。所属店舗は担当店舗のみのセレクトボックス |
| 従業員登録モーダル（オーナー） | 上部に「所属店舗」セレクトボックス追加（全店舗から選択可能） |
| 従業員登録モーダル（管理者） | 担当店舗が1つの場合は初期選択済み＋disabled、複数の場合は担当店舗のみに絞ったセレクトボックス |
| 管理者一覧 (`/dashboard/administrators`) | 一覧テーブル + 「管理者を登録」ボタン（直接登録、初期PW設定）+ 編集・削除ボタン（テキストボタン）。**PW操作列なし**（管理者は自分でPW再設定可能）。削除時は確認ダイアログ表示 |
| アカウント設定 ー パスワード (`/dashboard/account/password`) | F-1: 全ユーザーがサイドバー「アカウント設定」からPW確認・変更。モック参照 |

### 8.3 UIデザインルール

| 要素 | ルール |
|------|--------|
| 一覧テーブルの操作ボタン | テキストボタン（「編集」「削除」）で統一。アイコンボタンは使用しない。※ 店舗一覧は既存画面のためアイコンボタン（QR・編集・削除）を踏襲 |
| 削除ボタンの色 | 赤文字（`#c62828`） |
| 確認ダイアログ | 削除・PW再発行など不可逆操作は `confirm()` で確認 |
| 従業員一覧のステータス表示 | オーナー視点・管理者視点ともに全3ステータス（招待中/有効/期限切れ）の行を表示し、ボタンの出し分けを統一する |

---

## 9. 実装フェーズ

### Phase 1（P0 — 最優先）
1. DBマイグレーション（invitations, users.clerk_user_id/role, manager_stores）
2. フローB: 管理者一覧 + 直接登録 + 編集・削除
3. フローA: セルフサインアップ（会社+オーナー登録）
4. フローD: 従業員登録 + 招待メール送信基盤
5. フローE: PW再設定（パスワードを忘れた）
6. フローF: 管理者による従業員PW再発行
7. フローG: 従業員招待メール再送信

### Phase 2（P1）
8. フローC: 店舗管理
9. フローD: 従業員一覧の店舗フィルター + 所属店舗列
10. 従業員登録モーダルへの所属店舗セレクト追加
11. ログイン画面改修
12. Clerk Components のカスタマイズ（ブランディング）

### Phase 3（P2 — 将来）
13. オーナー権限の移譲
14. 管理者の交代
15. CSVインポートによる従業員一括登録

---

## 10. テスト要件

### 10.1 ユニットテスト（RSpec）

- `Invitation` モデル: バリデーション、トークン生成、期限切れ判定
- `ClerkService`: Clerk Backend API連携（ユーザー作成・削除・PW更新）
- `User` モデル: role判定メソッド、権限チェック、clerk_user_id紐付け
- 各Policyクラス: 権限マトリクスの全パターン

### 10.2 リクエストテスト

- 全APIエンドポイントの正常系・異常系
- 権限チェック（オーナー以外がオーナー専用APIを叩いた場合の403）
- 管理者のスコープ制限（担当外店舗へのアクセスが403）
- レート制限の動作確認
- トークン有効期限の動作確認

### 10.3 統合テスト

- サインアップ完全フロー（アカウント作成 → メール認証 → 会社登録 → ログイン）
- 管理者直接登録フロー（オーナーが登録 → 管理者が初期PWでログイン → 強制PW変更）
- 従業員招待フロー（登録 → 招待メール受信 → PW設定 → ログイン）
- パスワード再設定フロー（メール入力 → 確認コード → 新PW設定 → ログイン）
- 管理者による従業員PW再発行フロー（PW再発行 → メール受信 → ログイン → 強制PW変更）
- 従業員招待メール再送信フロー（再送信 → メール受信 → PW設定）
- 従業員登録フロー（オーナーから / 管理者から）

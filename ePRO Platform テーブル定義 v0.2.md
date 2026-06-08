# ePRO Platform テーブル定義 v0.2

---

# Schema: medical_auth

## organizations テーブル

| カラム名 | 型 | 制約 | 説明 |
|---|---|---|---|
| id | UUID | PK | 組織ID |
| organization_type | VARCHAR(50) | NOT NULL | 組織種別（hospital / university / pharma） |
| name | VARCHAR(255) | NOT NULL | 組織名 |
| active | BOOLEAN | NOT NULL | 有効フラグ |
| created_at | TIMESTAMP | NOT NULL | 作成日時 |
| updated_at | TIMESTAMP | NOT NULL | 更新日時 |

---

## users テーブル

| カラム名 | 型 | 制約 | 説明 |
|---|---|---|---|
| id | UUID | PK | 管理者ユーザーID |
| organization_id | UUID | FK(organizations.id), NOT NULL | 所属組織 |
| email | VARCHAR(255) | UNIQUE, NOT NULL | ログインメールアドレス |
| password_hash | TEXT | NOT NULL | ハッシュ化パスワード |
| mfa_enabled | BOOLEAN | NOT NULL | 二段階認証有効フラグ |
| active | BOOLEAN | NOT NULL | アカウント有効フラグ |
| created_at | TIMESTAMP | NOT NULL | 作成日時 |
| updated_at | TIMESTAMP | NOT NULL | 更新日時 |

---

## roles テーブル

| カラム名 | 型 | 制約 | 説明 |
|---|---|---|---|
| id | UUID | PK | ロールID |
| role_name | VARCHAR(100) | UNIQUE, NOT NULL | ロール名 |
| created_at | TIMESTAMP | NOT NULL | 作成日時 |

---

## user_roles テーブル

| カラム名 | 型 | 制約 | 説明 |
|---|---|---|---|
| id | UUID | PK | ユーザーロールID |
| user_id | UUID | FK(users.id), NOT NULL | ユーザーID |
| role_id | UUID | FK(roles.id), NOT NULL | ロールID |
| created_at | TIMESTAMP | NOT NULL | 作成日時 |

---

## audit_logs テーブル

| カラム名 | 型 | 制約 | 説明 |
|---|---|---|---|
| id | UUID | PK | 監査ログID |
| user_id | UUID | FK(users.id) | 操作実行ユーザー |
| action | VARCHAR(100) | NOT NULL | 実行操作 |
| target_type | VARCHAR(100) |  | 対象種別 |
| target_id | UUID |  | 対象ID |
| ip_address | VARCHAR(100) |  | IPアドレス |
| user_agent | TEXT |  | ブラウザ情報 |
| created_at | TIMESTAMP | NOT NULL | 実行日時 |

---

# Schema: medical_private

## patients テーブル

| カラム名 | 型 | 制約 | 説明 |
|---|---|---|---|
| id | UUID | PK | 患者内部ID |
| patient_uuid | UUID | UNIQUE, NOT NULL | システム共通患者UUID |
| facility_id | UUID | FK(organizations.id), NOT NULL | 医療機関ID |
| emr_patient_id | VARCHAR(255) |  | 電子カルテ患者ID |
| full_name | TEXT |  | 暗号化患者氏名 |
| birth_date | DATE |  | 暗号化生年月日 |
| gender | VARCHAR(20) |  | 性別 |
| created_at | TIMESTAMP | NOT NULL | 作成日時 |
| updated_at | TIMESTAMP | NOT NULL | 更新日時 |

---

## patient_contacts テーブル

| カラム名 | 型 | 制約 | 説明 |
|---|---|---|---|
| id | UUID | PK | 連絡先ID |
| patient_uuid | UUID | FK(patients.patient_uuid), NOT NULL | 患者UUID |
| email | TEXT |  | 暗号化メールアドレス |
| phone_number | TEXT |  | 暗号化電話番号 |
| email_verified_at | TIMESTAMP |  | メール確認日時 |
| created_at | TIMESTAMP | NOT NULL | 作成日時 |
| updated_at | TIMESTAMP | NOT NULL | 更新日時 |

---

## patient_magic_links テーブル

| カラム名 | 型 | 制約 | 説明 |
|---|---|---|---|
| id | UUID | PK | マジックリンクID |
| patient_uuid | UUID | FK(patients.patient_uuid), NOT NULL | 患者UUID |
| token_hash | TEXT | NOT NULL | ハッシュ化ログイントークン |
| expires_at | TIMESTAMP | NOT NULL | 有効期限 |
| used_at | TIMESTAMP |  | 使用日時 |
| ip_address | VARCHAR(100) |  | アクセス元IP |
| user_agent | TEXT |  | ブラウザ情報 |
| created_at | TIMESTAMP | NOT NULL | 作成日時 |

---

## patient_consents テーブル

| カラム名 | 型 | 制約 | 説明 |
|---|---|---|---|
| id | UUID | PK | 同意ID |
| patient_uuid | UUID | FK(patients.patient_uuid), NOT NULL | 患者UUID |
| consent_version | VARCHAR(50) | NOT NULL | 同意書バージョン |
| agreed_at | TIMESTAMP |  | 同意日時 |
| revoked_at | TIMESTAMP |  | 同意撤回日時 |
| ip_address | VARCHAR(100) |  | 同意時IP |
| created_at | TIMESTAMP | NOT NULL | 作成日時 |

---

# Schema: medical_research

## research_projects テーブル

| カラム名 | 型 | 制約 | 説明 |
|---|---|---|---|
| id | UUID | PK | 研究プロジェクトID |
| organization_id | UUID | FK(organizations.id), NOT NULL | 実施組織 |
| project_name | VARCHAR(255) | NOT NULL | プロジェクト名 |
| description | TEXT |  | 説明 |
| status | VARCHAR(50) | NOT NULL | 状態（draft / active / closed） |
| created_by | UUID | FK(users.id) | 作成ユーザー |
| created_at | TIMESTAMP | NOT NULL | 作成日時 |
| updated_at | TIMESTAMP | NOT NULL | 更新日時 |

---

## project_patients テーブル

| カラム名 | 型 | 制約 | 説明 |
|---|---|---|---|
| id | UUID | PK | 登録ID |
| project_id | UUID | FK(research_projects.id), NOT NULL | 研究プロジェクトID |
| patient_uuid | UUID | FK(patients.patient_uuid), NOT NULL | 患者UUID |
| enrolled_at | TIMESTAMP |  | 参加日時 |
| withdrawn_at | TIMESTAMP |  | 中止日時 |
| created_at | TIMESTAMP | NOT NULL | 作成日時 |

---

## surveys テーブル

| カラム名 | 型 | 制約 | 説明 |
|---|---|---|---|
| id | UUID | PK | アンケートID |
| project_id | UUID | FK(research_projects.id), NOT NULL | 研究プロジェクトID |
| formbricks_survey_id | VARCHAR(255) | NOT NULL | Formbricks Survey ID |
| survey_name | VARCHAR(255) | NOT NULL | アンケート名 |
| active | BOOLEAN | NOT NULL | 有効フラグ |
| created_at | TIMESTAMP | NOT NULL | 作成日時 |
| updated_at | TIMESTAMP | NOT NULL | 更新日時 |

---

## survey_deliveries テーブル

| カラム名 | 型 | 制約 | 説明 |
|---|---|---|---|
| id | UUID | PK | 配信ID |
| patient_uuid | UUID | FK(patients.patient_uuid), NOT NULL | 患者UUID |
| survey_id | UUID | FK(surveys.id), NOT NULL | アンケートID |
| delivery_type | VARCHAR(50) | NOT NULL | 配信方法（email / sms / line） |
| sent_at | TIMESTAMP |  | 配信日時 |
| opened_at | TIMESTAMP |  | 開封日時 |
| completed_at | TIMESTAMP |  | 回答完了日時 |
| status | VARCHAR(50) | NOT NULL | 状態 |
| created_at | TIMESTAMP | NOT NULL | 作成日時 |

---

## survey_responses テーブル

| カラム名 | 型 | 制約 | 説明 |
|---|---|---|---|
| id | UUID | PK | 回答ID |
| survey_delivery_id | UUID | FK(survey_deliveries.id), NOT NULL | 配信ID |
| formbricks_response_id | VARCHAR(255) | NOT NULL | Formbricks Response ID |
| responded_at | TIMESTAMP |  | 回答日時 |
| created_at | TIMESTAMP | NOT NULL | 作成日時 |

---

## notifications テーブル

| カラム名 | 型 | 制約 | 説明 |
|---|---|---|---|
| id | UUID | PK | 通知ID |
| patient_uuid | UUID | FK(patients.patient_uuid), NOT NULL | 患者UUID |
| type | VARCHAR(50) | NOT NULL | 通知種別 |
| subject | VARCHAR(255) | NOT NULL | 通知タイトル |
| sent_at | TIMESTAMP |  | 送信日時 |
| opened_at | TIMESTAMP |  | 開封日時 |
| status | VARCHAR(50) | NOT NULL | 状態 |
| created_at | TIMESTAMP | NOT NULL | 作成日時 |

---

## reward_logs テーブル

| カラム名 | 型 | 制約 | 説明 |
|---|---|---|---|
| id | UUID | PK | 謝礼ID |
| patient_uuid | UUID | FK(patients.patient_uuid), NOT NULL | 患者UUID |
| survey_delivery_id | UUID | FK(survey_deliveries.id), NOT NULL | 配信ID |
| reward_type | VARCHAR(50) | NOT NULL | 謝礼種別 |
| reward_amount | INTEGER | NOT NULL | 謝礼金額 |
| sent_at | TIMESTAMP |  | 送信日時 |
| status | VARCHAR(50) | NOT NULL | 状態 |
| created_at | TIMESTAMP | NOT NULL | 作成日時 |

---

# リレーション

- organizations.id ← users.organization_id (1:N)
- users.id ← user_roles.user_id (1:N)
- roles.id ← user_roles.role_id (1:N)
- users.id ← audit_logs.user_id (1:N)

- organizations.id ← patients.facility_id (1:N)

- patients.patient_uuid ← patient_contacts.patient_uuid (1:N)
- patients.patient_uuid ← patient_magic_links.patient_uuid (1:N)
- patients.patient_uuid ← patient_consents.patient_uuid (1:N)

- organizations.id ← research_projects.organization_id (1:N)

- research_projects.id ← project_patients.project_id (1:N)
- patients.patient_uuid ← project_patients.patient_uuid (1:N)

- research_projects.id ← surveys.project_id (1:N)

- patients.patient_uuid ← survey_deliveries.patient_uuid (1:N)
- surveys.id ← survey_deliveries.survey_id (1:N)

- survey_deliveries.id ← survey_responses.survey_delivery_id (1:1)

- patients.patient_uuid ← notifications.patient_uuid (1:N)

- patients.patient_uuid ← reward_logs.patient_uuid (1:N)
- survey_deliveries.id ← reward_logs.survey_delivery_id (1:N)

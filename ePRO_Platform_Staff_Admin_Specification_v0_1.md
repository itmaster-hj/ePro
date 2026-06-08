# ePRO Platform Staff Admin Specification v0.1

---

# 本ドキュメントについて

本ドキュメントは Claude Code / ChatGPT / Cursor 等へ直接投入可能な
**スタッフ向け管理画面（CRC/Study Admin）仕様書**である。

## 対象ユーザー

- CRC（Clinical Research Coordinator）
- Study Admin（研究管理者）
- 医師（Investigator）

## 対象機能

1. アンケート作成・管理
2. 患者リスト管理
3. 配信管理（患者ごと）
4. 回答状況確認（アンケートごと）

## 技術スタック

- Frontend: Next.js App Router
- Backend: Next.js Route Handlers
- Database: PostgreSQL + Prisma
- UI: TailwindCSS
- State: Zustand
- Form: React Hook Form
- Validation: Zod
- Export: csv (papaparse / xlsx)

---

# 1. システム概要

## 管理画面の目的

- CRC/Study Admin が研究内で実施するアンケートの作成・配信・管理
- 患者リストの一元管理
- アンケート回答状況のリアルタイム確認
- 配信管理（リマインド、再配信、配信停止）
- CSV/Excel出力による分析

## アーキテクチャ

```
┌─────────────────────────────────────────────┐
│   Staff Admin Dashboard (Next.js App Router) │
│                                             │
│  ├─ Survey Management                      │
│  ├─ Patient List                            │
│  ├─ Delivery Management                     │
│  └─ Response Dashboard                      │
├─────────────────────────────────────────────┤
│   API Layer (Next.js Route Handlers)        │
│                                             │
│  ├─ /api/v1/surveys/*                      │
│  ├─ /api/v1/survey-deliveries/*            │
│  ├─ /api/v1/patients/*                     │
│  └─ /api/v1/survey-responses/*             │
├─────────────────────────────────────────────┤
│   Database Layer (PostgreSQL + Prisma)      │
│                                             │
│  ├─ Survey                                 │
│  ├─ SurveyDelivery                         │
│  ├─ SurveyResponse                         │
│  ├─ Patient                                │
│  ├─ ProjectPatient                         │
│  └─ RewardLog                              │
└─────────────────────────────────────────────┘
```

---

# 2. 認可ポリシー

## ロール別アクセス権

| 機能 | super_admin | study_admin | crc | investigator |
|---|---|---|---|---|
| Survey作成 | ○ | ○ | ○ | × |
| Survey削除 | ○ | ○ | × | × |
| Survey配信 | ○ | ○ | ○ | × |
| 患者検索 | ○ | ○ | ○ | ○ |
| 患者登録 | ○ | ○ | ○ | × |
| 配信管理 | ○ | ○ | ○ | × |
| リマインド送信 | ○ | ○ | ○ | × |
| 回答確認 | ○ | ○ | ○ | ○ |
| CSV出力 | ○ | ○ | ○ | ○ |

---

# 3. スクリーン一覧

| Screen ID | Screen Name | URL | Role |
|---|---|---|---|
| A-001 | Dashboard | /admin/dashboard | CRC+ |
| A-002 | Survey List | /admin/surveys | CRC+ |
| A-003 | Survey Create | /admin/surveys/create | CRC+ |
| A-004 | Survey Edit | /admin/surveys/:id/edit | CRC+ |
| A-005 | Survey Detail | /admin/surveys/:id | CRC+ |
| A-006 | Survey Delivery | /admin/surveys/:id/delivery | CRC+ |
| A-007 | Patient List | /admin/patients | CRC+ |
| A-008 | Patient Create | /admin/patients/create | CRC+ |
| A-009 | Patient Detail | /admin/patients/:id | CRC+ |
| A-010 | Delivery Management | /admin/deliveries | CRC+ |
| A-011 | Delivery Detail | /admin/deliveries/:id | CRC+ |
| A-012 | Response Dashboard | /admin/responses | CRC+ |
| A-013 | Export | /admin/export | CRC+ |

---

# 4. 機能詳細仕様

# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# 機能1: アンケート作成・管理
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## A-002 Survey List

### 画面目的

プロジェクト内で作成・配信したアンケート一覧を表示。
フィルタリング、検索、並び替え機能を提供。

### 使用API

| Method | Endpoint |
|---|---|
| GET | /api/v1/surveys |

### UI Components

- Project Selector Dropdown
- Survey Search Input
- Status Filter (Draft / Active / Archived)
- Table: Survey Name, Status, Created Date, Delivery Count, Response Count, Actions
- Create Survey Button
- Export Button

### 表示項目

| Item | Type | Description |
|---|---|---|
| survey_id | UUID | サーベイID |
| survey_name | String | アンケート名 |
| project_id | UUID | 所属プロジェクト |
| status | Enum | ステータス |
| created_by | UUID | 作成者 |
| created_at | DateTime | 作成日時 |
| delivery_count | Integer | 配信数 |
| response_count | Integer | 回答数 |

### State

```typescript
interface SurveyListState {
  surveys: Survey[]
  projectId: string
  filters: {
    status?: string
    keyword?: string
    sortBy?: 'name' | 'created_at' | 'response_count'
    sortOrder?: 'asc' | 'desc'
  }
  pagination: {
    page: number
    limit: number
    total: number
  }
  loading: boolean
  error: string | null
}
```

### Validation

- Project ID 必須
- Status 値：draft / active / archived

### Error Cases

| Error | HTTP | Description |
|---|---|---|
| UNAUTHORIZED | 401 | ユーザーは権限なし |
| PROJECT_NOT_FOUND | 404 | プロジェクトが見つからない |

### UX Notes

- ページネーション対応（デフォルト：20件）
- ステータスバッジで視覚化
- 作成日時で逆順がデフォルト

---

## A-003 Survey Create

### 画面目的

新規アンケートを作成。

### 使用API

| Method | Endpoint | Description |
|---|---|---|
| POST | /api/v1/surveys | 新規作成 |
| POST | /api/v1/surveys/:id/formbricks-sync | Formbricks同期 |

### UI Components

- Project Selector
- Survey Name Input
- Survey Title Input
- Description TextArea
- Researcher/Sponsor Selector (医師 / 製薬企業)
- Purpose TextArea
- Target Condition TextArea（初期版は説明のみ）
- Estimated Duration Input (分単位)
- Reward Amount Input (金額)
- Deadline Date Picker
- Save Draft Button
- Preview Button
- Publish Button

### フォーム仕様

```typescript
interface SurveyCreateInput {
  projectId: string // 必須
  surveyName: string // 必須、255字以下
  surveyTitle: string // 必須、255字以下
  description: string // 任意
  purpose: string // 必須（調査目的）
  targetCondition: string // 任意（対象条件説明）
  estimatedDurationMinutes: number // 必須、1-60
  rewardAmount: number // 必須、0-1000000
  deadlineDate: DateTime // 必須
  status: 'draft' | 'active' // デフォルト: draft
}
```

### Validation

- projectId: UUID形式
- surveyName: 1-255字、必須
- surveyTitle: 1-255字、必須
- estimatedDurationMinutes: 1-60, 整数
- rewardAmount: 0以上, 整数
- deadlineDate: 現在時刻より未来, 必須

### Error Cases

| Error | HTTP | Description |
|---|---|---|
| INVALID_PROJECT | 400 | プロジェクトが無効 |
| DUPLICATE_NAME | 409 | 同名アンケートが存在 |
| FORBIDDEN | 403 | 権限なし |

### UX Notes

- ドラフト保存で途中保存が可能
- プレビューで患者視点の確認
- 公開前に最終確認ダイアログ

---

## A-004 Survey Edit

### 画面目的

既存アンケートを編集。
公開後は一部項目のみ編集可能。

### 使用API

| Method | Endpoint |
|---|---|
| GET | /api/v1/surveys/:id |
| PATCH | /api/v1/surveys/:id |

### 編集可能項目（Status別）

| Item | Draft | Active | Archived |
|---|---|---|---|
| surveyName | ○ | × | × |
| surveyTitle | ○ | ○ | × |
| description | ○ | ○ | × |
| purpose | ○ | ○ | × |
| deadlineDate | ○ | ○ | × |
| rewardAmount | ○ | × | × |

### UI Components

A-003 と同様だが、編集禁止項目はグレーアウト

### State

```typescript
interface SurveyEditState {
  survey: Survey
  isDraft: boolean
  isModified: boolean
  loading: boolean
  error: string | null
}
```

### Error Cases

| Error | HTTP | Description |
|---|---|---|
| SURVEY_NOT_FOUND | 404 | アンケートが見つからない |
| CANNOT_EDIT_PUBLISHED | 403 | 公開済みは編集不可 |
| CONFLICT | 409 | 他ユーザーが編集中 |

### UX Notes

- 変更検知で保存促示
- 確認ダイアログで誤操作防止

---

## A-005 Survey Detail

### 画面目的

アンケートの詳細情報及び配信・回答状況を表示。

### 使用API

| Method | Endpoint |
|---|---|
| GET | /api/v1/surveys/:id |
| GET | /api/v1/surveys/:id/statistics |
| GET | /api/v1/survey-deliveries?surveyId=:id |

### UI Components

#### 上部セクション

- Survey Title, Description
- Status Badge
- Created Date, Last Updated
- Created By

#### 統計セクション

- Card: 配信数
- Card: 回答数
- Card: 回答率 (%)
- Card: 平均回答時間

#### 配信・回答管理セクション

- Delivery Status Table（配信一覧）
- Response Summary

### 表示項目

| Item | Type |
|---|---|
| survey_id | UUID |
| survey_name | String |
| project_id | UUID |
| status | Enum |
| delivery_count | Integer |
| response_count | Integer |
| response_rate | Float |
| avg_response_time_minutes | Integer |

### State

```typescript
interface SurveyDetailState {
  survey: Survey
  statistics: {
    deliveryCount: number
    responseCount: number
    responseRate: number // 0-100
    avgResponseTimeMinutes: number
  }
  deliveries: SurveyDelivery[]
  loading: boolean
  error: string | null
}
```

### Error Cases

| Error | HTTP | Description |
|---|---|---|
| SURVEY_NOT_FOUND | 404 | アンケートが見つからない |
| FORBIDDEN | 403 | 権限なし |

### UX Notes

- リアルタイム統計表示（自動更新）
- 配信状況を視覚化（グラフ等）

---

## A-006 Survey Delivery

### 画面目的

アンケートを特定患者に配信する。
初期段階では CRC が Excel などで患者をスクリーニングし、
手動で対象患者を選択して配信。

### 使用API

| Method | Endpoint | Description |
|---|---|---|
| GET | /api/v1/patients | 患者一覧取得 |
| GET | /api/v1/project-patients/:projectId | プロジェクト登録患者 |
| POST | /api/v1/survey/send | Survey 配信 |

### UI Components

- Survey Info Card（読み取り専用）
- Patient Selection

  - Patient Search Input（氏名、メール、患者ID）
  - Project Patients Table（プロジェクト登録患者一覧）
  - Checkbox: 対象患者を複数選択
  - Selected Patient Count Badge

- Delivery Settings

  - Delivery Type: Email / SMS / LINE
  - Schedule Date/Time Picker
  - Immediate / Scheduled Toggle

- Preview

  - Email Template Preview

- Confirmation Dialog
  - 配信対象患者数
  - 配信方法
  - 配信予定日時

- Send Button

### フォーム仕様

```typescript
interface SurveyDeliveryInput {
  surveyId: string // 必須
  projectId: string // 必須
  patientUuids: string[] // 必須、1件以上
  deliveryType: 'email' | 'sms' | 'line' // 必須、初期版はemailのみ推奨
  scheduleDateTime: DateTime // 必須
  immediate: boolean // true の場合、すぐに配信
}
```

### 患者一覧検索仕様

| Field | SearchType | Example |
|---|---|---|
| 患者氏名 | Partial | 山田 |
| メールアドレス | Exact | patient@example.com |
| 患者ID（EMR） | Partial | A123 |
| 参加ステータス | Exact | enrolled / withdrawn |

### Validation

- surveyId: UUID形式, 必須
- projectId: UUID形式, 必須
- patientUuids: UUID配列, 1件以上, 必須
- deliveryType: enum値, 必須
- scheduleDateTime: 現在時刻より1分以上後, 必須

### Error Cases

| Error | HTTP | Description |
|---|---|---|
| SURVEY_NOT_FOUND | 404 | アンケートが見つからない |
| PATIENT_NOT_FOUND | 404 | 患者が見つからない |
| INVALID_EMAIL | 400 | メールアドレスが無効 |
| SURVEY_CLOSED | 403 | 回答期限切れ |
| DELIVERY_FAILED | 500 | 配信失敗 |

### UX Notes

- チェックボックスで複数選択
- 選択患者数をリアルタイム表示
- キャンセル時は確認ダイアログ
- 配信成功時は Toast 通知

---

# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# 機能2: 患者リスト管理
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## A-007 Patient List

### 画面目的

プロジェクト登録患者一覧を表示。
検索、フィルタリング、患者登録。

### 使用API

| Method | Endpoint |
|---|---|
| GET | /api/v1/patients |
| GET | /api/v1/project-patients/:projectId |

### UI Components

- Project Selector Dropdown
- Patient Search Input（氏名、メール、患者ID）
- Status Filter (enrolled / withdrawn)
- Table: Patient ID, Name, Email, Birth Date, Enrollment Date, Status, Actions
- Register Patient Button
- Bulk Import Button (将来版)
- Export Button

### 表示項目

| Item | Type | Encrypted |
|---|---|---|
| patient_uuid | UUID | - |
| hospital_patient_id | String | - |
| full_name | String | ○ |
| email | String | ○ |
| birth_date | Date | ○ |
| enrolled_at | DateTime | - |
| status | Enum | - |

### State

```typescript
interface PatientListState {
  patients: Patient[]
  projectId: string
  filters: {
    keyword?: string
    status?: 'enrolled' | 'withdrawn'
    sortBy?: 'name' | 'enrolled_at'
    sortOrder?: 'asc' | 'desc'
  }
  pagination: {
    page: number
    limit: number
    total: number
  }
  loading: boolean
  error: string | null
}
```

### Validation

- Project ID 必須
- Status 値: enrolled / withdrawn

### Error Cases

| Error | HTTP | Description |
|---|---|---|
| PROJECT_NOT_FOUND | 404 | プロジェクトが見つからない |
| FORBIDDEN | 403 | アクセス権限なし |

### UX Notes

- ページネーション対応（20件/ページ）
- 名前でバイロバル検索
- Enrolled患者を優先表示

---

## A-008 Patient Create

### 画面目的

新規患者を登録。
医療情報企画部から取得したデータ、または手入力。

### 使用API

| Method | Endpoint |
|---|---|
| POST | /api/v1/patients |
| POST | /api/v1/project-patients |

### UI Components

- Hospital/Organization Selector
- Full Name Input
- Birth Date Picker
- Gender Select (Male / Female / Other)
- Email Input
- Phone Number Input
- Hospital Patient ID Input
- Medical Information TextArea（疾患、内服薬等）
- Project Selector（複数選択可能）
- Save Button
- Cancel Button

### フォーム仕様

```typescript
interface PatientCreateInput {
  facilityId: string // 必須
  hospitalPatientId: string // 必須、一意制約
  fullName: string // 必須、255字以下
  birthDate: Date // 必須
  gender: 'male' | 'female' | 'other' // 必須
  email: string // 必須、メール形式
  phoneNumber?: string // 任意
  medicalInfo?: string // 任意（疾患、内服薬等）
  projectIds: string[] // 必須、1件以上
}
```

### Validation

- facilityId: UUID形式, 必須
- hospitalPatientId: 1-255字, 必須, 一意
- fullName: 1-255字, 必須
- birthDate: 過去の日付, 必須
- email: メール形式, 必須
- phoneNumber: 電話番号形式（如何なる形式でも受け入れ）
- projectIds: UUID配列, 1件以上, 必須

### Error Cases

| Error | HTTP | Description |
|---|---|---|
| DUPLICATE_PATIENT | 409 | 同じ hospital_patient_id が存在 |
| INVALID_EMAIL | 400 | メールアドレスが無効 |
| FACILITY_NOT_FOUND | 404 | 医療機関が見つからない |

### UX Notes

- 生年月日は日本語カレンダー
- 電話番号は任意
- プロジェクト複数選択可能
- 登録後は患者詳細画面へリダイレクト

---

## A-009 Patient Detail

### 画面目的

個別患者の詳細情報と参加プロジェクト、アンケート回答履歴を表示。

### 使用API

| Method | Endpoint |
|---|---|
| GET | /api/v1/patients/:id |
| GET | /api/v1/project-patients?patientUuid=:id |
| GET | /api/v1/survey-deliveries?patientUuid=:id |

### UI Components

#### 患者情報セクション

- Patient Info Card（read-only）
  - Patient ID
  - Name
  - Birth Date
  - Gender
  - Email
  - Phone Number
  - Medical Info
- Edit Button（権限あれば表示）

#### プロジェクト参加セクション

- Enrolled Projects Table
  - Project Name
  - Enrollment Date
  - Status (enrolled / withdrawn)
  - Actions (Withdraw)

#### アンケート回答履歴セクション

- Survey History Table
  - Survey Name
  - Delivery Date
  - Status (pending / opened / completed / expired)
  - Response Date（回答済みの場合）
  - Reward Status
  - Actions (View Response)

### 表示項目

| Item | Type |
|---|---|
| patient_uuid | UUID |
| hospital_patient_id | String |
| full_name | String |
| birth_date | Date |
| email | String |
| projects[] | Project[] |
| survey_deliveries[] | SurveyDelivery[] |

### State

```typescript
interface PatientDetailState {
  patient: Patient
  projects: ProjectPatient[]
  deliveries: SurveyDelivery[]
  loading: boolean
  error: string | null
}
```

### Error Cases

| Error | HTTP | Description |
|---|---|---|
| PATIENT_NOT_FOUND | 404 | 患者が見つからない |
| FORBIDDEN | 403 | アクセス権限なし |

### UX Notes

- 参加プロジェクトで中止可能
- 回答済みアンケートは詳細確認可能

---

# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# 機能3: 配信管理（患者ごと）
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## A-010 Delivery Management

### 画面目的

全アンケート配信の状況を一覧表示。
フィルタリング、検索、リマインド、再配信、停止。

### 使用API

| Method | Endpoint |
|---|---|
| GET | /api/v1/survey-deliveries |
| POST | /api/v1/survey/remind |
| POST | /api/v1/survey/resend |
| POST | /api/v1/survey/stop |

### UI Components

- Project Selector
- Survey Selector（プロジェクト内のみ）
- Status Filter (scheduled / sent / opened / completed / failed)
- Patient Name/Email Search
- Date Range Picker（配信日）
- Table: Patient Name, Survey Name, Delivery Date, Status, Opened At, Completed At, Actions
- Bulk Remind Button
- Bulk Resend Button
- Bulk Stop Button
- Export Button

### 表示項目

| Item | Type | Description |
|---|---|---|
| delivery_id | UUID | - |
| patient_uuid | UUID | - |
| survey_id | UUID | - |
| patient_name | String | 患者名 |
| survey_name | String | アンケート名 |
| delivery_type | Enum | email / sms / line |
| sent_at | DateTime | 配信日時 |
| opened_at | DateTime | 開封日時 |
| completed_at | DateTime | 完了日時 |
| status | Enum | scheduled/sent/opened/completed/failed |

### State

```typescript
interface DeliveryManagementState {
  deliveries: SurveyDelivery[]
  projectId: string
  surveyId?: string
  filters: {
    status?: string
    keyword?: string
    dateRange?: [Date, Date]
  }
  selectedDeliveries: string[] // チェックボックス選択
  pagination: {
    page: number
    limit: number
    total: number
  }
  loading: boolean
  error: string | null
}
```

### Validation

- Project ID 必須
- Status 値: scheduled / sent / opened / completed / failed

### Error Cases

| Error | HTTP | Description |
|---|---|---|
| DELIVERY_NOT_FOUND | 404 | 配信が見つからない |
| SURVEY_CLOSED | 403 | 回答期限切れ |
| DELIVERY_ALREADY_COMPLETED | 409 | 既に回答済み |

### UX Notes

- ステータスバッジで視覚化
- Pending配信を優先表示
- チェックボックスで複数選択
- 一括操作に確認ダイアログ

---

## A-011 Delivery Detail

### 画面目的

個別配信の詳細情報を表示。
リマインド、再配信、停止操作。

### 使用API

| Method | Endpoint |
|---|---|
| GET | /api/v1/survey-deliveries/:id |
| POST | /api/v1/survey/remind |
| POST | /api/v1/survey/resend |
| POST | /api/v1/survey/stop |

### UI Components

#### 配信情報セクション

- Delivery Info Card
  - Patient Name
  - Survey Name
  - Delivery Type
  - Sent Date
  - Opened Date（if available）
  - Completed Date（if available）
  - Status Badge

#### アクション セクション

- Remind Button（status: sent のみ）
- Resend Button（status: sent/failed のみ）
- Stop Button（status: scheduled/sent のみ）
- Delete Button（Draft Survey のみ）

#### 回答情報セクション（if completed）

- Response Summary
  - Response Date
  - Response Time
  - View Response Details Link

### State

```typescript
interface DeliveryDetailState {
  delivery: SurveyDelivery
  response?: SurveyResponse
  loading: boolean
  error: string | null
  actionInProgress?: string // 'remind' | 'resend' | 'stop'
}
```

### Error Cases

| Error | HTTP | Description |
|---|---|---|
| DELIVERY_NOT_FOUND | 404 | 配信が見つからない |
| CANNOT_REMIND_COMPLETED | 409 | 回答済みにリマインド不可 |
| SURVEY_CLOSED | 403 | 回答期限切れ |

### UX Notes

- 各アクション実行時は確認ダイアログ
- 成功時は Toast 通知
- 失敗時はエラーメッセージ表示

---

# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
# 機能4: 回答状況確認（アンケートごと）
# ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

## A-012 Response Dashboard

### 画面目的

アンケートごとの回答状況をダッシュボード形式で表示。
配信数、回答数、回答率、未回答者等を可視化。

### 使用API

| Method | Endpoint |
|---|---|
| GET | /api/v1/surveys |
| GET | /api/v1/surveys/:id/statistics |
| GET | /api/v1/survey-deliveries?surveyId=:id |
| GET | /api/v1/survey-responses |

### UI Components

#### 上部統計セクション

- KPI Cards
  - 配信数 (Total Deliveries)
  - 回答数 (Total Responses)
  - 回答率 (Response Rate %)
  - 未回答数 (Pending)
  - 平均回答時間 (Avg Response Time)

#### グラフセクション

- Status Distribution Pie Chart
  - Pending（未回答）
  - Opened（開封済み）
  - Completed（完了）
  - Failed（失敗）

- Timeline Chart
  - 日付ごとの累積回答数

#### テーブルセクション

- 回答者一覧 Table
  - Patient Name
  - Delivery Date
  - Response Date
  - Response Time
  - Reward Status
  - View Response Link

- 未回答者一覧 Table
  - Patient Name
  - Delivery Date
  - Days Since Delivery
  - Action (Remind / Stop)

### 表示項目

```typescript
interface ResponseStatistics {
  surveyId: string
  deliveryCount: number
  responseCount: number
  responseRate: number // 0-100
  pendingCount: number
  openedCount: number
  completedCount: number
  failedCount: number
  avgResponseTimeMinutes: number
  respondedPatients: {
    patientUuid: string
    patientName: string
    deliveryDate: DateTime
    responseDate: DateTime
    responseTime: number
  }[]
  pendingPatients: {
    patientUuid: string
    patientName: string
    deliveryDate: DateTime
    daysSinceDelivery: number
  }[]
}
```

### State

```typescript
interface ResponseDashboardState {
  projectId: string
  surveyId: string
  statistics: ResponseStatistics
  respondedPatients: Patient[]
  pendingPatients: Patient[]
  filters: {
    responseStatus?: string
  }
  loading: boolean
  error: string | null
}
```

### Error Cases

| Error | HTTP | Description |
|---|---|---|
| SURVEY_NOT_FOUND | 404 | アンケートが見つからない |
| FORBIDDEN | 403 | アクセス権限なし |

### UX Notes

- 統計情報のリアルタイム更新（5分ごと）
- グラフはResponsive対応
- 未回答者一覧でリマインド実行可能

---

## A-013 Export

### 画面目的

アンケート回答データ、患者情報、配信状況を CSV/Excel 形式で出力。

### 使用API

| Method | Endpoint |
|---|---|
| GET | /api/v1/survey-responses |
| GET | /api/v1/survey-deliveries |
| GET | /api/v1/patients |

### UI Components

- Project Selector
- Survey Selector（プロジェクト内のみ）
- Export Type Selector
  - Responses Only（回答データのみ）
  - Deliveries with Status（配信状況）
  - Patient List（患者情報）
  - All in One（統合）
- File Format Selector
  - CSV
  - Excel
- Export Button
- Download Link（生成後）

### 出力形式

#### Responses CSV

```csv
survey_id,survey_name,patient_uuid,patient_name,delivery_date,response_date,response_time_minutes,reward_amount
```

#### Deliveries CSV

```csv
delivery_id,survey_id,survey_name,patient_uuid,patient_name,delivery_date,opened_at,completed_at,status
```

#### Patient List CSV

```csv
patient_uuid,hospital_patient_id,full_name,email,birth_date,gender,enrolled_date
```

### State

```typescript
interface ExportState {
  projectId: string
  surveyId?: string
  exportType: 'responses' | 'deliveries' | 'patients' | 'all'
  fileFormat: 'csv' | 'excel'
  loading: boolean
  error: string | null
}
```

### Validation

- Project ID 必須
- Export Type 必須
- File Format 必須

### Error Cases

| Error | HTTP | Description |
|---|---|---|
| NO_DATA | 400 | 出力データなし |
| EXPORT_FAILED | 500 | 出力失敗 |

### UX Notes

- 非同期エクスポート（大規模データ対応）
- ダウンロード開始時は Toast 通知
- Filename に timestamp を含める

---

# 5. 推奨追加テーブル・API

## 追加テーブル

### SurveyDraft

途中保存機能用（将来実装）

```sql
CREATE TABLE medical_research.survey_drafts (
  id UUID PRIMARY KEY,
  survey_id UUID NOT NULL,
  created_by UUID NOT NULL,
  draft_content JSONB NOT NULL,
  created_at TIMESTAMP NOT NULL,
  updated_at TIMESTAMP NOT NULL
)
```

### SurveyTemplate

アンケートテンプレート管理（将来実装）

## 推奨追加API

| Endpoint | Method | Purpose |
|---|---|---|
| /api/v1/surveys/:id/clone | POST | アンケート複製 |
| /api/v1/surveys/:id/archive | POST | アンケート非公開化 |
| /api/v1/survey-deliveries/bulk-remind | POST | 一括リマインド |
| /api/v1/survey-deliveries/bulk-resend | POST | 一括再配信 |
| /api/v1/patients/bulk-import | POST | 患者一括登録 |
| /api/v1/analytics/cohort-analysis | GET | コホート分析（将来版） |

---

# 6. 推奨フロントエンド構成

## ファイル構造

```
src/
├── app/
│   └── admin/
│       ├── dashboard/
│       │   └── page.tsx
│       ├── surveys/
│       │   ├── page.tsx
│       │   ├── create/
│       │   │   └── page.tsx
│       │   ├── [id]/
│       │   │   ├── page.tsx
│       │   │   ├── edit/
│       │   │   │   └── page.tsx
│       │   │   └── delivery/
│       │   │       └── page.tsx
│       ├── patients/
│       │   ├── page.tsx
│       │   ├── create/
│       │   │   └── page.tsx
│       │   └── [id]/
│       │       └── page.tsx
│       ├── deliveries/
│       │   ├── page.tsx
│       │   └── [id]/
│       │       └── page.tsx
│       ├── responses/
│       │   └── page.tsx
│       └── export/
│           └── page.tsx
│
├── components/
│   ├── admin/
│   │   ├── SurveyForm/
│   │   ├── PatientTable/
│   │   ├── DeliveryTable/
│   │   ├── StatisticsCards/
│   │   └── ResponseChart/
│   ├── ui/
│   │   ├── Table/
│   │   ├── Modal/
│   │   ├── Form/
│   │   └── Badge/
│
├── hooks/
│   ├── useSurveys.ts
│   ├── usePatients.ts
│   ├── useDeliveries.ts
│   └── useResponses.ts
│
├── lib/
│   ├── api.ts
│   ├── validators.ts
│   └── export.ts
│
└── store/
    └── adminStore.ts (Zustand)
```

## 推奨パッケージ

| Package | Purpose |
|---|---|
| next | Framework |
| react-hook-form | Form Management |
| zod | Validation |
| zustand | State Management |
| @tanstack/react-query | Data Fetching |
| tailwindcss | Styling |
| recharts | Charting |
| papaparse | CSV Export |
| xlsx | Excel Export |

---

# 7. 推奨開発順序

```
1. API 実装（Survey, Patient, Delivery, Response）
   ↓
2. Database Migration（Prisma）
   ↓
3. 認可・RBAC Middleware
   ↓
4. A-007 Patient List
   ↓
5. A-008 Patient Create
   ↓
6. A-002 Survey List
   ↓
7. A-003 Survey Create
   ↓
8. A-006 Survey Delivery
   ↓
9. A-010 Delivery Management
   ↓
10. A-012 Response Dashboard
   ↓
11. A-013 Export
   ↓
12. Docker Build & AWS EC2 Deploy
```

---

# 8. セキュリティ要件

## 認可

- RBAC に基づく API Gate
- ユーザーの組織スコープの確認
- 他組織患者データへのアクセス禁止

## PII 保護

- 患者情報は medical_private schema に隔離
- 暗号化フィールド（full_name, email, phone_number）
- PII 出力時はアクセス制限

## 監査ログ

- 全 CRUD 操作を audit_logs に記録
- IP, User Agent を記録
- 重要操作（配信、削除）は明示的に

## 通信

- HTTPS 強制
- JWT Token 有効期限設定
- Refresh Token Rotation

---

# 9. エラーハンドリング

## クライアント側

```typescript
interface ApiResponse<T> {
  data?: T
  error?: {
    code: string
    message: string
  }
}

// Error Codes
enum ErrorCode {
  UNAUTHORIZED = 'UNAUTHORIZED',
  FORBIDDEN = 'FORBIDDEN',
  NOT_FOUND = 'NOT_FOUND',
  CONFLICT = 'CONFLICT',
  VALIDATION_ERROR = 'VALIDATION_ERROR',
  INTERNAL_ERROR = 'INTERNAL_ERROR'
}
```

## サーバー側

```typescript
// Next.js Route Handler Error Response
export async function handleApiError(error: unknown) {
  if (error instanceof PrismaClientKnownRequestError) {
    return NextResponse.json(
      { error: 'DATABASE_ERROR', message: '...' },
      { status: 500 }
    )
  }
  // ...
}
```

---

# 10. パフォーマンス最適化

## API

- ページネーション必須（デフォルト：20件）
- Select 句で不要カラム除外
- インデックス設定（status, created_at, patient_uuid）
- キャッシング戦略（TanStack Query）

## Frontend

- Lazy Loading（Image, Component）
- Code Splitting
- Server-side Rendering for SEO
- Suspense による段階的レンダリング

---

# 11. テスト戦略

## Unit Tests

- Zod Validators
- Utility Functions
- Hooks

## Integration Tests

- API エンドポイント
- Prisma Query

## E2E Tests

- 主要ユーザーフロー
  - Survey 作成 → 配信 → 回答確認

---

# 12. デプロイメント

## Docker

```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build
EXPOSE 3000
CMD ["npm", "start"]
```

## Environment Variables

```env
DATABASE_URL=postgresql://...
NEXT_PUBLIC_API_BASE_URL=https://...
JWT_SECRET=...
FORMBRICKS_API_KEY=...
```

## AWS EC2

- Docker Image をビルド
- ECR へ Push
- ECS / App Runner で実行
- RDS with PostgreSQL
- ALB で負荷分散

---

# 13. 今後の拡張予定

- [ ] 患者一括インポート（CSV/Excel）
- [ ] アンケートテンプレート管理
- [ ] 自動配信スケジュール
- [ ] LINE / SMS 配信サポート
- [ ] AIベース回答分析
- [ ] EMR統合
- [ ] GraphQL API
- [ ] Mobile App for CRC

---

# 付録A: API 仕様サマリー

## Survey Management APIs

```
POST /api/v1/surveys
GET /api/v1/surveys
GET /api/v1/surveys/:id
PATCH /api/v1/surveys/:id
DELETE /api/v1/surveys/:id
GET /api/v1/surveys/:id/statistics
```

## Survey Delivery APIs

```
POST /api/v1/survey/send
GET /api/v1/survey-deliveries
GET /api/v1/survey-deliveries/:id
POST /api/v1/survey/remind
POST /api/v1/survey/resend
POST /api/v1/survey/stop
```

## Patient APIs

```
POST /api/v1/patients
GET /api/v1/patients
GET /api/v1/patients/:id
PATCH /api/v1/patients/:id
GET /api/v1/project-patients/:projectId
```

## Response APIs

```
GET /api/v1/survey-responses
GET /api/v1/survey-responses/:id
GET /api/v1/surveys/:id/statistics
```

## Export APIs

```
GET /api/v1/export/responses
GET /api/v1/export/deliveries
GET /api/v1/export/patients
```

---

# 付録B: データベース設計

### テーブル関連図

```
medical_auth
├── organizations
├── users
├── user_roles
├── roles
└── audit_logs

medical_private
├── patients
├── patient_contacts
├── patient_magic_links
└── patient_consents

medical_research
├── research_projects
├── project_patients
├── surveys
├── survey_deliveries
├── survey_responses
├── notifications
└── reward_logs
```

---

# 付録C: 用語集

| Term | 説明 |
|---|---|
| Survey | アンケート |
| Delivery | 配信 |
| Response | 回答 |
| CRC | Clinical Research Coordinator（臨床試験コーディネーター） |
| Study Admin | 研究管理者 |
| Patient UUID | Formbricks へも送信される患者ID |
| PII | Personally Identifiable Information（個人識別情報） |
| RBAC | Role Based Access Control（ロールベースアクセス制御） |
| Magic Link | パスワード不要のメール認証 |

---

END OF DOCUMENT

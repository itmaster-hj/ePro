# ePRO Patient Screen Design v0.1
# ------------------------------------------------------------
# 本ドキュメントは Claude Code / ChatGPT / Cursor 等へ
# 直接投入可能な患者画面設計仕様書である。
#
# 目的：
# - 患者向けePRO画面設計の統一
# - React/Next.js実装の自動生成補助
# - API不足検知
# - DB不足検知
# - UX統一
#
# 対象：
# - Next.js App Router
# - React
# - TailwindCSS
# - Zustand
# - PostgreSQL
# - Formbricks
#
# ------------------------------------------------------------

# 1. システム概要

## システム名

ePRO Platform

---

## システム目的

患者がスマートフォンやPCブラウザから
PRO（Patient Reported Outcome）アンケートへ
回答するためのWebアプリケーション。

---

## 想定利用者

- 外来患者
- 高齢患者
- 慢性疾患患者
- 臨床研究参加患者

---

## 基本設計方針

- モバイルファースト
- 高齢者向けUI
- 1画面1質問
- ログイン負荷最小化
- Magic Link認証
- 電子同意対応
- 自動保存
- 回答継続率重視
- PII分離
- 医療監査対応

---

# 2. 技術構成

| Layer | Technology |
|---|---|
| Frontend | Next.js |
| Backend | Next.js Route Handlers |
| Database | PostgreSQL |
| ORM | Prisma |
| State | Zustand |
| Validation | Zod |
| Survey Engine | Formbricks |
| Authentication | Magic Link |
| Hosting | Docker + AWS EC2 |

---

# 3. 患者画面一覧

| Screen ID | Screen Name | URL |
|---|---|---|
| P-001 | Enrollment Entry | /entry |
| P-002 | Account Registration | /register |
| P-003 | Verify Email | /verify |
| P-004 | Patient Information | /patient-info |
| P-005 | Consent | /consent |
| P-006 | My Page | /mypage |
| P-007 | Survey List | /surveys |
| P-008 | Survey Detail | /surveys/:id |
| P-009 | Survey Answer | /surveys/:id/answer |
| P-010 | Survey Complete | /surveys/:id/complete |
| P-011 | Notifications | /notifications |
| P-012 | Profile | /profile |
| P-013 | Login | /login |
| P-014 | Error | /error |

---

# 4. 画面遷移図

```text
Enrollment Entry
 ↓
Account Registration
 ↓
Verify Email
 ↓
Patient Information
 ↓
Consent
 ↓
My Page
 ├─ Survey List
 │    └─ Survey Detail
 │           └─ Survey Answer
 │                  └─ Survey Complete
 │
 ├─ Notifications
 │
 └─ Profile
```

---

# 5. 共通UXルール

## 高齢者向けUI設計

- 最小フォントサイズ：18px
- ボタン高さ：48px以上
- タップ領域広め
- 日本語簡潔化
- 横スクロール禁止
- シングルカラムUI
- スマホ最適化

---

## Survey UXルール

- 1画面1質問
- Progress Bar表示
- 自動保存
- 回答途中再開
- 戻る可能
- 通信失敗時リトライ
- オフライン考慮

---

## アクセシビリティ

- WCAG考慮
- 色だけで状態表現しない
- 高コントラスト
- エラーメッセージ明示

---

# 6. 認証仕様

## 認証方式

Magic Link Authentication

---

## 認証ルール

- パスワードレス
- トークン1回のみ利用可能
- 有効期限あり
- JWT使用

---

## 関連テーブル

- medical_private.patient_magic_links
- medical_private.patient_contacts

---

## 関連API

| Method | Endpoint | Description |
|---|---|---|
| POST | /api/v1/patient-enrollment/request | 登録開始 |
| POST | /api/v1/patient-auth/request-link | Magic Link送信 |
| GET | /api/v1/patient-auth/verify | Token検証 |
| POST | /api/v1/patient-auth/logout | ログアウト |

---

# 7. 画面詳細

# ------------------------------------------------------------
# P-001 Enrollment Entry
# ------------------------------------------------------------

## 画面ID

P-001

---

## 画面名

Enrollment Entry

---

## URL

```text
/entry
```

---

## 画面目的

患者登録開始画面。

研究招待コードを入力する。

---

## UI Components

- Invite Code Input
- Continue Button
- Error Message

---

## 使用API

| Method | Endpoint |
|---|---|
| GET | /api/v1/patient-enrollment/project-info |

---

## State

| State | Type |
|---|---|
| inviteCode | string |
| loading | boolean |
| error | string |

---

## Validation

- 招待コード必須
- 招待コード形式確認
- 研究有効性確認

---

## Error Cases

| Error | Description |
|---|---|
| INVALID_INVITE_CODE | 招待コード無効 |
| PROJECT_CLOSED | 研究終了 |

---

## UX Notes

- 大きな入力欄
- 数字/英字入力補助
- エラー表示を明確化

---

# ------------------------------------------------------------
# P-002 Account Registration
# ------------------------------------------------------------

## 画面ID

P-002

---

## URL

```text
/register
```

---

## 画面目的

患者メールアドレス登録。

---

## UI Components

- Email Input
- Terms Checkbox
- Register Button

---

## 使用API

| Method | Endpoint |
|---|---|
| POST | /api/v1/patient-enrollment/request |

---

## State

| State | Type |
|---|---|
| email | string |
| termsAccepted | boolean |
| loading | boolean |
| error | string |

---

## Validation

- email形式確認
- 利用規約同意必須

---

## UX Notes

- ボタン大きめ
- 高齢者向け文字サイズ
- 説明簡潔化

---

# ------------------------------------------------------------
# P-003 Verify Email
# ------------------------------------------------------------

## 画面ID

P-003

---

## URL

```text
/verify
```

---

## 画面目的

Magic Link確認。

---

## UI Components

- Verification Message
- Loading Spinner
- Resend Link Button

---

## 使用API

| Method | Endpoint |
|---|---|
| GET | /api/v1/patient-auth/verify |

---

## Error Cases

| Error | Description |
|---|---|
| TOKEN_EXPIRED | Token期限切れ |
| INVALID_TOKEN | 無効Token |

---

## UX Notes

- Loading表示
- 失敗時再送導線

---

# ------------------------------------------------------------
# P-004 Patient Information
# ------------------------------------------------------------

## 画面ID

P-004

---

## URL

```text
/patient-info
```

---

## 画面目的

患者基本情報入力。

---

## UI Components

- Name Input
- Birth Date Picker
- Gender Select
- Hospital Patient ID Input
- Save Button

---

## 関連テーブル

- medical_private.patients
- medical_private.patient_contacts

---

## Validation

- 必須入力
- 生年月日妥当性

---

## UX Notes

- Date Picker簡易化
- 入力項目最小化

---

# ------------------------------------------------------------
# P-005 Consent
# ------------------------------------------------------------

## 画面ID

P-005

---

## URL

```text
/consent
```

---

## 画面目的

電子同意取得。

---

## UI Components

- Consent Document
- Agree Checkbox
- Electronic Signature
- Submit Button

---

## 使用API

| Method | Endpoint |
|---|---|
| POST | /api/v1/consents |

---

## 関連テーブル

- medical_private.patient_consents

---

## UX Notes

- PDF表示対応
- スクロール確認
- 同意version表示

---

## Error Cases

| Error | Description |
|---|---|
| CONSENT_FAILED | 同意登録失敗 |

---

# ------------------------------------------------------------
# P-006 My Page
# ------------------------------------------------------------

## 画面ID

P-006

---

## URL

```text
/mypage
```

---

## 画面目的

患者ホーム画面。

---

## UI Components

- Pending Surveys
- Completed Surveys
- Reward Summary
- Notifications
- Profile Link

---

## 関連テーブル

- survey_deliveries
- reward_logs
- notifications

---

## UX Notes

- 未回答を最上部表示
- 回答期限強調

---

# ------------------------------------------------------------
# P-007 Survey List
# ------------------------------------------------------------

## 画面ID

P-007

---

## URL

```text
/surveys
```

---

## 画面目的

アンケート一覧表示。

---

## UI Components

- Survey Card
- Status Badge
- Due Date
- Resume Button

---

## Status Types

- pending
- opened
- completed
- expired

---

## UX Notes

- 未回答優先表示
- 期限切れ強調

---

# ------------------------------------------------------------
# P-008 Survey Detail
# ------------------------------------------------------------

## 画面ID

P-008

---

## URL

```text
/surveys/:id
```

---

## 画面目的

アンケート説明画面。

---

## UI Components

- Survey Title
- Description
- Estimated Time
- Question Count
- Start Button

---

## UX Notes

- 所要時間表示
- 離脱率低減

---

# ------------------------------------------------------------
# P-009 Survey Answer
# ------------------------------------------------------------

## 画面ID

P-009

---

## URL

```text
/surveys/:id/answer
```

---

## 画面目的

アンケート回答。

---

## UI Components

- Progress Bar
- Question Card
- Radio Buttons
- Numeric Scale
- Text Area
- Next Button
- Previous Button
- Save Draft

---

## 使用API

| Method | Endpoint |
|---|---|
| GET | Formbricks Survey API |
| POST | Formbricks Response API |

---

## 関連テーブル

- survey_deliveries
- survey_responses

---

## State

| State | Type |
|---|---|
| currentQuestion | number |
| answers | object |
| progress | number |
| autosaveStatus | string |

---

## UX Rules

- 1画面1質問
- 自動保存
- 戻る可能
- スマホ最適化
- タップ領域大きめ

---

## Error Cases

| Error | Description |
|---|---|
| NETWORK_ERROR | 通信失敗 |
| TOKEN_EXPIRED | Token期限切れ |
| SURVEY_CLOSED | 回答期限切れ |

---

# ------------------------------------------------------------
# P-010 Survey Complete
# ------------------------------------------------------------

## 画面ID

P-010

---

## URL

```text
/surveys/:id/complete
```

---

## 画面目的

回答完了画面。

---

## UI Components

- Completion Message
- Reward Information
- Return Home Button

---

## UX Notes

- 達成感表示
- 謝礼案内

---

# ------------------------------------------------------------
# P-011 Notifications
# ------------------------------------------------------------

## 画面ID

P-011

---

## URL

```text
/notifications
```

---

## 画面目的

通知履歴表示。

---

## UI Components

- Notification List
- Read Status
- Open Detail

---

## 関連テーブル

- notifications

---

# ------------------------------------------------------------
# P-012 Profile
# ------------------------------------------------------------

## 画面ID

P-012

---

## URL

```text
/profile
```

---

## 画面目的

患者プロフィール管理。

---

## UI Components

- Email
- Phone Number
- Notification Settings
- Logout Button

---

# ------------------------------------------------------------
# P-013 Login
# ------------------------------------------------------------

## 画面ID

P-013

---

## URL

```text
/login
```

---

## 画面目的

既存患者ログイン。

---

## UI Components

- Email Input
- Send Magic Link Button

---

## 使用API

| Method | Endpoint |
|---|---|
| POST | /api/v1/patient-auth/request-link |

---

# ------------------------------------------------------------
# P-014 Error
# ------------------------------------------------------------

## 画面ID

P-014

---

## URL

```text
/error
```

---

## 画面目的

エラー表示。

---

## Error Types

- TOKEN_EXPIRED
- NETWORK_ERROR
- PROJECT_CLOSED
- SURVEY_CLOSED
- MAINTENANCE

---

# 8. グローバルState設計

| State | Description |
|---|---|
| authPatient | ログイン患者 |
| activeSurvey | 回答中アンケート |
| consentStatus | 同意状態 |
| notifications | 通知一覧 |
| rewardSummary | 謝礼情報 |

---

# 9. 推奨追加テーブル

## survey_drafts

途中保存用。

---

## consent_versions

同意書version管理。

---

# 10. 推奨追加API

| API | Purpose |
|---|---|
| /survey/draft | 下書き保存 |
| /notifications/read | 通知既読 |
| /patient/me | 自己情報取得 |
| /patient/me/update | 自己情報更新 |

---

# 11. 推奨Frontend構成

| Purpose | Technology |
|---|---|
| UI | TailwindCSS |
| State | Zustand |
| Form | React Hook Form |
| Validation | Zod |
| Data Fetching | TanStack Query |
| PWA | next-pwa |

---

# 12. 推奨開発順序

```text
Screen Design
↓
Wireframe
↓
Component Design
↓
Next.js App Router
↓
API Hooks
↓
State Design
↓
Formbricks Integration
↓
PWA
↓
Docker Build
↓
Production Deploy
```

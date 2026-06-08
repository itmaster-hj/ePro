# ePRO Platform API Specification v0.1

---

# 1. System Overview

## Architecture

- Frontend: Next.js
- Backend: Next.js Route Handlers
- Database: PostgreSQL
- Survey Engine: Formbricks
- Infrastructure: Docker + AWS EC2

---

## Database Schemas

| Schema | Purpose |
|---|---|
| medical_private | PII information |
| medical_research | Research and survey data |
| medical_auth | Authentication and authorization |

---

## Authentication Policy

### Staff Users

- Email + Password
- MFA required

### Patient Users

- Invite code
- Magic link authentication
- No password

---

## Design Policies

- PII must be separated from survey data
- Formbricks stores patient_uuid only
- Audit logs required
- Logical delete required
- Future EMR integration planned

---

# 2. Base API Rules

## Base URL

```text
/api/v1
```

---

## Authentication Header

```http
Authorization: Bearer <JWT_TOKEN>
```

---

## Success Response Format

```json
{
  "data": {}
}
```

---

## Error Response Format

```json
{
  "error": "ERROR_CODE",
  "message": "Human readable message"
}
```

---

# 3. RBAC Roles

| Role | Description |
|---|---|
| super_admin | System administrator |
| study_admin | Study administrator |
| crc | Clinical research coordinator |
| investigator | Physician |
| patient | Patient |

---

# 4. Staff Auth APIs

---

### POST /api/v1/auth/login

**説明：** スタッフログイン

**認可ロール：**
- Public

**リクエスト例：**
```json
{
  "email": "admin@example.com",
  "password": "password"
}
```

**レスポンス例 (200)：**
```json
{
  "access_token": "jwt_token",
  "refresh_token": "refresh_token",
  "user": {
    "user_id": "uuid-xxx",
    "name": "System Admin",
    "role": "super_admin"
  }
}
```

**エラーレスポンス例 (401)：**
```json
{
  "error": "INVALID_CREDENTIALS",
  "message": "メールアドレスまたはパスワードが正しくありません"
}
```

---

### POST /api/v1/auth/mfa/verify

**説明：** MFA認証確認

**認可ロール：**
- Public

**リクエスト例：**
```json
{
  "session_id": "uuid-xxx",
  "otp_code": "123456"
}
```

**レスポンス例 (200)：**
```json
{
  "access_token": "jwt_token"
}
```

**エラーレスポンス例 (401)：**
```json
{
  "error": "INVALID_OTP",
  "message": "OTPコードが正しくありません"
}
```

---

### GET /api/v1/auth/me

**説明：** ログインユーザー情報取得

**認可ロール：**
- super_admin
- study_admin
- crc
- investigator

**レスポンス例 (200)：**
```json
{
  "user_id": "uuid-xxx",
  "name": "Taro Yamada",
  "email": "user@example.com",
  "role": "crc"
}
```

---

### POST /api/v1/auth/logout

**説明：** ログアウト

**認可ロール：**
- Authenticated users

**レスポンス例 (200)：**
```json
{
  "status": "logged_out"
}
```

---

# 5. Patient Enrollment APIs

---

### POST /api/v1/patient-enrollment/request

**説明：** 招待コードを使用して患者自己登録を開始する

**認可ロール：**
- Public

**リクエスト例：**
```json
{
  "invite_code": "HF2026-X82KQ",
  "email": "patient@example.com"
}
```

**レスポンス例 (200)：**
```json
{
  "status": "magic_link_sent"
}
```

**エラーレスポンス例 (400)：**
```json
{
  "error": "INVALID_INVITE_CODE",
  "message": "招待コードが無効です"
}
```

**エラーレスポンス例 (403)：**
```json
{
  "error": "PROJECT_CLOSED",
  "message": "研究は終了しています"
}
```

---

### GET /api/v1/patient-enrollment/project-info

**説明：** 招待コードから研究情報を取得する

**認可ロール：**
- Public

**クエリ例：**
```text
/api/v1/patient-enrollment/project-info?code=HF2026-X82KQ
```

**レスポンス例 (200)：**
```json
{
  "project_name": "Heart Failure PRO Study",
  "description": "Heart failure patient survey study"
}
```

---

# 6. Patient Auth APIs

---

### POST /api/v1/patient-auth/request-link

**説明：** マジックリンクを送信する

**認可ロール：**
- Public

**リクエスト例：**
```json
{
  "email": "patient@example.com"
}
```

**レスポンス例 (200)：**
```json
{
  "status": "sent"
}
```

---

### GET /api/v1/patient-auth/verify

**説明：** マジックリンクを検証する

**認可ロール：**
- Public

**クエリ例：**
```text
/api/v1/patient-auth/verify?token=xxxxx
```

**レスポンス例 (200)：**
```json
{
  "access_token": "jwt_token",
  "patient_uuid": "uuid-xxx"
}
```

**エラーレスポンス例 (401)：**
```json
{
  "error": "TOKEN_EXPIRED",
  "message": "マジックリンクの有効期限が切れています"
}
```

---

### POST /api/v1/patient-auth/logout

**説明：** 患者ログアウト

**認可ロール：**
- patient

**レスポンス例 (200)：**
```json
{
  "status": "logged_out"
}
```

---

# 7. Patient APIs

---

### POST /api/v1/patients

**説明：** 患者を新規登録する

**認可ロール：**
- super_admin
- study_admin
- crc

**リクエスト例：**
```json
{
  "hospital_patient_id": "A12345",
  "name": "山田太郎",
  "birth_date": "1980-01-01",
  "sex": "male",
  "email": "patient@example.com",
  "phone": "09012345678"
}
```

**レスポンス例 (201)：**
```json
{
  "patient_uuid": "uuid-xxx",
  "status": "created",
  "created_at": "2026-05-24T10:00:00Z"
}
```

**エラーレスポンス例 (409)：**
```json
{
  "error": "DUPLICATE_PATIENT",
  "message": "同一患者が既に存在します"
}
```

---

### GET /api/v1/patients

**説明：** 患者検索

**認可ロール：**
- super_admin
- study_admin
- crc
- investigator

**クエリ例：**
```text
/api/v1/patients?keyword=山田&page=1&limit=20
```

**レスポンス例 (200)：**
```json
{
  "items": [
    {
      "patient_uuid": "uuid-xxx",
      "hospital_patient_id": "A12345",
      "name": "山田太郎"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 1
  }
}
```

---

### GET /api/v1/patients/:id

**説明：** 患者詳細取得

**認可ロール：**
- super_admin
- study_admin
- crc
- investigator

**レスポンス例 (200)：**
```json
{
  "patient_uuid": "uuid-xxx",
  "hospital_patient_id": "A12345",
  "name": "山田太郎",
  "birth_date": "1980-01-01",
  "projects": [
    {
      "project_id": "uuid-project",
      "project_name": "Heart Failure Study"
    }
  ]
}
```

---

### PATCH /api/v1/patients/:id

**説明：** 患者情報更新

**認可ロール：**
- super_admin
- study_admin
- crc

**リクエスト例：**
```json
{
  "email": "new@example.com",
  "phone": "08011112222"
}
```

**レスポンス例 (200)：**
```json
{
  "status": "updated"
}
```

---

### DELETE /api/v1/patients/:id

**説明：** 患者論理削除

**認可ロール：**
- super_admin

**レスポンス例 (200)：**
```json
{
  "status": "deleted"
}
```

---

# 8. Project APIs

---

### POST /api/v1/projects

**説明：** 研究プロジェクト作成

**認可ロール：**
- super_admin
- study_admin

**リクエスト例：**
```json
{
  "project_code": "HF-001",
  "project_name": "Heart Failure PRO Study",
  "description": "Heart failure patient survey study",
  "start_date": "2026-01-01",
  "end_date": "2026-12-31"
}
```

**レスポンス例 (201)：**
```json
{
  "project_id": "uuid-project",
  "status": "created"
}
```

---

### GET /api/v1/projects

**説明：** 研究一覧取得

**認可ロール：**
- Authenticated users

**レスポンス例 (200)：**
```json
{
  "items": [
    {
      "project_id": "uuid-project",
      "project_code": "HF-001",
      "project_name": "Heart Failure PRO Study"
    }
  ]
}
```

---

### GET /api/v1/projects/:id

**説明：** 研究詳細取得

**認可ロール：**
- Authenticated users

**レスポンス例 (200)：**
```json
{
  "project_id": "uuid-project",
  "project_code": "HF-001",
  "project_name": "Heart Failure PRO Study",
  "status": "active"
}
```

---

### POST /api/v1/projects/:id/publish

**説明：** 研究公開

**認可ロール：**
- super_admin
- study_admin

**レスポンス例 (200)：**
```json
{
  "status": "published"
}
```

---

### POST /api/v1/projects/:id/close

**説明：** 研究終了

**認可ロール：**
- super_admin
- study_admin

**レスポンス例 (200)：**
```json
{
  "status": "closed"
}
```

---

# 9. Survey APIs

---

### POST /api/v1/survey/send

**説明：** 患者へSurveyを送信する

**認可ロール：**
- study_admin
- crc

**リクエスト例：**
```json
{
  "project_id": "uuid-project",
  "patient_uuid": "uuid-patient",
  "survey_template_code": "PHQ9",
  "schedule_date": "2026-06-01T09:00:00Z"
}
```

**レスポンス例 (201)：**
```json
{
  "delivery_id": "uuid-delivery",
  "formbricks_survey_id": "survey-id",
  "status": "scheduled"
}
```

---

### GET /api/v1/survey/history

**説明：** Survey送信履歴取得

**認可ロール：**
- study_admin
- crc
- investigator

**レスポンス例 (200)：**
```json
{
  "items": [
    {
      "delivery_id": "uuid-delivery",
      "patient_uuid": "uuid-patient",
      "survey_template_code": "PHQ9",
      "status": "completed"
    }
  ]
}
```

---

### GET /api/v1/survey/responses/:id

**説明：** Survey回答取得

**認可ロール：**
- study_admin
- crc
- investigator

**レスポンス例 (200)：**
```json
{
  "response_id": "uuid-response",
  "submitted_at": "2026-06-01T12:00:00Z",
  "answers": [
    {
      "question_code": "Q1",
      "value": 3
    }
  ]
}
```

---

### POST /api/v1/survey/remind

**説明：** Surveyリマインド送信

**認可ロール：**
- study_admin
- crc

**リクエスト例：**
```json
{
  "delivery_id": "uuid-delivery"
}
```

**レスポンス例 (200)：**
```json
{
  "status": "resent"
}
```

---

### POST /api/v1/survey/webhook/formbricks

**説明：** Formbricks Webhook受信

**認可ロール：**
- Internal only

**リクエスト例：**
```json
{
  "event": "response.completed",
  "survey_id": "survey-id",
  "response_id": "response-id"
}
```

**レスポンス例 (200)：**
```json
{
  "status": "received"
}
```

---

# 10. Consent APIs

---

### POST /api/v1/consents

**説明：** 電子同意登録

**認可ロール：**
- crc
- investigator

**リクエスト例：**
```json
{
  "patient_uuid": "uuid-patient",
  "project_id": "uuid-project",
  "consent_version": "v1.0",
  "consented_at": "2026-06-01T10:00:00Z"
}
```

**レスポンス例 (201)：**
```json
{
  "consent_id": "uuid-consent",
  "status": "registered"
}
```

---

### GET /api/v1/consents/:id

**説明：** 同意情報取得

**認可ロール：**
- crc
- investigator
- study_admin

**レスポンス例 (200)：**
```json
{
  "consent_id": "uuid-consent",
  "patient_uuid": "uuid-patient",
  "project_id": "uuid-project",
  "consent_version": "v1.0"
}
```

---

# 11. Reward APIs

---

### GET /api/v1/rewards

**説明：** 謝礼一覧取得

**認可ロール：**
- study_admin
- crc

**レスポンス例 (200)：**
```json
{
  "items": [
    {
      "reward_id": "uuid-reward",
      "patient_uuid": "uuid-patient",
      "amount": 500
    }
  ]
}
```

---

### POST /api/v1/rewards/grant

**説明：** 謝礼付与

**認可ロール：**
- study_admin

**リクエスト例：**
```json
{
  "patient_uuid": "uuid-patient",
  "amount": 500,
  "reason": "Survey completion"
}
```

**レスポンス例 (201)：**
```json
{
  "reward_id": "uuid-reward",
  "status": "granted"
}
```

---

# 12. Audit APIs

---

### GET /api/v1/audit-logs

**説明：** 監査ログ検索

**認可ロール：**
- super_admin
- study_admin

**クエリ例：**
```text
/api/v1/audit-logs?action=PATIENT_CREATE
```

**レスポンス例 (200)：**
```json
{
  "items": [
    {
      "audit_log_id": "uuid-audit",
      "user_id": "uuid-user",
      "action": "PATIENT_CREATE",
      "timestamp": "2026-06-01T00:00:00Z"
    }
  ]
}
```

---

# 13. User Management APIs

---

### GET /api/v1/users

**説明：** ユーザー一覧取得

**認可ロール：**
- super_admin

**レスポンス例 (200)：**
```json
{
  "items": [
    {
      "user_id": "uuid-user",
      "name": "System Admin",
      "role": "super_admin"
    }
  ]
}
```

---

### POST /api/v1/users

**説明：** ユーザー作成

**認可ロール：**
- super_admin

**リクエスト例：**
```json
{
  "name": "New CRC",
  "email": "crc@example.com",
  "role": "crc"
}
```

**レスポンス例 (201)：**
```json
{
  "user_id": "uuid-user",
  "status": "created"
}
```

---

### PATCH /api/v1/users/:id/role

**説明：** ロール変更

**認可ロール：**
- super_admin

**リクエスト例：**
```json
{
  "role": "study_admin"
}
```

**レスポンス例 (200)：**
```json
{
  "status": "updated"
}
```

---

# 14. Security Requirements

## JWT

- Access token expiration required
- Refresh token rotation required

---

## Magic Link

- Token hash required
- One-time use only
- Expiration required

---

## Webhook Security

- Signature validation required
- IP restriction recommended

---

## Database

- Logical delete required
- Audit log required

---

# 15. Future Planned APIs

| API | Purpose |
|---|---|
| /schedules | Visit scheduling |
| /notifications | LINE/SMS/email |
| /files | PDF consent |
| /emr/link | EMR integration |
| /adverse-events | AE reporting |

---

# 16. Recommended Development Flow

```text
API Specification
↓
Prisma Schema
↓
Next.js App Router
↓
RBAC Middleware
↓
Formbricks Integration
↓
Frontend Screens
↓
Docker Deployment
↓
AWS EC2 Production
```
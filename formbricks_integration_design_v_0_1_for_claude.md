# ePRO Platform
# Formbricks Integration Design v0.1

---

# 1. ドキュメント概要

本ドキュメントは ePRO Platform における
Formbricks 統合設計を定義する。

本仕様書は Claude Code に直接投入可能な形式を前提とする。

対象範囲：

- Survey Rendering
- Survey Schema Mapping
- Response Flow
- Webhook Integration
- Patient App Integration
- Response Storage Strategy
- Versioning
- Audit
- Security

---

# 2. 設計方針（重要）

## 基本思想

Formbricks は「アンケート UI エンジン」として利用する。

ePRO システムの中心は自前実装とする。

---

## システム責務分離

| 機能 | 管理主体 |
|---|---|
| Patient Management | ePRO Core |
| Survey Management | ePRO Core |
| Delivery Management | ePRO Core |
| Reward Management | ePRO Core |
| Analytics | ePRO Core |
| Audit Log | ePRO Core |
| Survey Rendering | Formbricks |
| Question UI | Formbricks |
| Branch Logic | Formbricks |

---

## 非推奨構成

以下は禁止。

- Formbricks を Master DB にする
- Formbricks DB を直接参照する
- Formbricks schema を内部DBへそのまま保存する
- Formbricks のみへ回答保存する

---

## 推奨構成

```txt
Formbricks = Survey UI Engine
PostgreSQL = Source of Truth
```

---

# 3. システム構成

## Architecture Flow

```txt
Admin Console
   ↓
Internal Survey Schema
   ↓
Formbricks JSON Generator
   ↓
Formbricks Survey Render
   ↓
Patient App
   ↓
Patient Response
   ↓
Formbricks Webhook
   ↓
Next.js API
   ↓
PostgreSQL Save
```

---

# 4. Internal Survey Schema

## 4-1. 概要

ePRO 内部では Formbricks 独自 schema を直接利用しない。

内部標準 schema を定義し、
Formbricks JSON へ変換する。

目的：

- Formbricks upgrade耐性
- schema version固定
- audit対応
- 将来他エンジンへの移行容易化
- CDISC/FHIR対応

---

## 4-2. Internal Survey Interface

```ts
export interface InternalSurvey {
  id: string
  version: number
  title: string
  description?: string
  purpose?: string
  rewardAmount?: number
  expiresAt?: string
  status: 'draft' | 'active' | 'closed'
  questions: InternalQuestion[]
  createdAt: string
  updatedAt: string
}
```

---

## 4-3. Internal Question Interface

```ts
export interface InternalQuestion {
  id: string
  type: 'single' | 'multiple' | 'text'
  title: string
  description?: string
  required: boolean
  options?: string[]
  sortOrder: number
}
```

---

# 5. Formbricks Mapping

## 5-1. Question Type Mapping

| Internal Type | Formbricks Type |
|---|---|
| single | multipleChoiceSingle |
| multiple | multipleChoiceMulti |
| text | openText |

---

## 5-2. Mapping Rules

### Single Choice

```ts
single
→ multipleChoiceSingle
```

---

### Multiple Choice

```ts
multiple
→ multipleChoiceMulti
```

---

### Free Text

```ts
text
→ openText
```

---

# 6. Survey JSON Generator

## 6-1. 概要

InternalSurvey を Formbricks JSON に変換する。

生成処理は Server Side で実施する。

---

## 6-2. 推奨構成

```txt
lib/formbricks/
  ├ mapper.ts
  ├ generator.ts
  ├ validator.ts
  └ types.ts
```

---

## 6-3. Generator Example

```ts
export function generateFormbricksSurvey(
  survey: InternalSurvey
) {
  return {
    name: survey.title,
    questions: survey.questions.map(mapQuestion)
  }
}
```

---

# 7. Patient App Integration

## 7-1. 基本方針

Patient App は Next.js ベースで構築する。

Formbricks は Embed/SDK として利用する。

---

## 7-2. iframe禁止

以下は禁止。

```txt
iframe embedding
```

理由：

- 認証制御困難
- モバイルUX低下
- Session共有問題
- audit困難
- style崩れ

---

## 7-3. 推奨構成

```txt
Next.js Patient App
   ↓
Survey Page
   ↓
Formbricks Component
```

---

## 7-4. Page Structure

```txt
app/
 └ patient/
     └ surveys/
         └ [token]/
             └ page.tsx
```

---

# 8. Magic Link Integration

## 8-1. 基本方針

認証は ePRO Core 側で実施する。

Formbricks 側認証は利用しない。

---

## 8-2. Token Flow

```txt
Email Delivery
   ↓
Magic Link Click
   ↓
Next.js Validation
   ↓
Patient Session Create
   ↓
Survey Render
```

---

## 8-3. Token要件

| 項目 | 内容 |
|---|---|
| Token Type | UUID |
| Expire | configurable |
| One Time Use | 推奨 |
| Revocation | 必須 |
| Audit | 必須 |

---

# 9. Response Flow

## 9-1. 回答保存方針

回答データの Source of Truth は PostgreSQL とする。

Formbricks のみへの保存は禁止。

---

## 9-2. 推奨フロー

```txt
Patient Submit
   ↓
Formbricks
   ↓
Webhook
   ↓
Next.js API
   ↓
medical_research schema
```

---

## 9-3. 保存構造

### 推奨

```txt
survey_responses
survey_response_answers
```

---

### 非推奨

```txt
single large JSON only
```

理由：

- SQL集計困難
- BI困難
- analytics困難
- CDISC変換困難

---

# 10. Webhook Integration

## 10-1. 対応イベント

| Event | 用途 |
|---|---|
| response.created | 回答登録 |
| response.updated | 回答更新 |
| survey.completed | 完了処理 |

---

## 10-2. Webhook Endpoint

```txt
/api/webhooks/formbricks
```

---

## 10-3. セキュリティ

必須対応：

- Signature Validation
- Secret Token
- Replay Attack Prevention
- Audit Log

---

# 11. DB設計方針

## 11-1. 推奨テーブル

| テーブル | 用途 |
|---|---|
| surveys | アンケート |
| survey_versions | version管理 |
| survey_questions | 質問 |
| survey_deliveries | 配信 |
| survey_responses | 回答 |
| survey_response_answers | 回答詳細 |
| survey_audit_logs | 監査 |

---

## 11-2. survey_versions

回答時点の schema snapshot を保存する。

目的：

- 監査
- 再現性
- immutable保証

---

# 12. Versioning Design

## 12-1. 基本方針

公開済みアンケートは immutable とする。

---

## 12-2. 更新方式

```txt
v1 → active
修正
↓
v2 作成
```

既存回答は旧versionへ紐づけ維持。

---

## 12-3. Version Rules

| 項目 | 方針 |
|---|---|
| 質問変更 | 新version |
| 選択肢変更 | 新version |
| typo修正 | minor可 |
| 回答schema変更 | 新version |

---

# 13. Analytics Design

## 13-1. 集計方針

集計は ePRO Core 側で実施する。

Formbricks analytics は補助用途のみ。

---

## 13-2. 対応内容

| 内容 | 備考 |
|---|---|
| 回答率 | 必須 |
| 未回答数 | 必須 |
| 日次推移 | 推奨 |
| CSV Export | 必須 |
| Excel Export | 必須 |

---

# 14. Notification Design

## 14-1. 通知主体

通知管理は ePRO Core 側で実施する。

Formbricks の通知機能へ依存しない。

---

## 14-2. 対応通知

| 種類 | 内容 |
|---|---|
| Email | 必須 |
| Reminder | 必須 |
| SMS | 将来 |
| Push | 将来 |
| LINE | 将来 |

---

# 15. Security Requirements

## 15-1. 必須要件

| 項目 | 内容 |
|---|---|
| HTTPS | 必須 |
| MFA | 管理者必須 |
| RBAC | 必須 |
| Audit Log | 必須 |
| Token Expire | 必須 |
| Session Timeout | 必須 |

---

## 15-2. PII/PHI保護

以下はマスキング対応。

- email
- patient_name
- diagnosis
- medications

---

# 16. Audit Design

## 16-1. 記録対象

| 操作 | 記録 |
|---|---|
| Survey Create | 必須 |
| Survey Update | 必須 |
| Publish | 必須 |
| Delivery | 必須 |
| Response | 必須 |
| Export | 必須 |

---

## 16-2. Audit項目

| 項目 | 内容 |
|---|---|
| user_id | 操作者 |
| action | 操作 |
| target_id | 対象 |
| before_data | 変更前 |
| after_data | 変更後 |
| ip_address | IP |
| created_at | 時刻 |

---

# 17. Queue / Worker 推奨

## 17-1. Queue対象

| 処理 | Queue化 |
|---|---|
| Email送信 | 必須 |
| Reminder | 必須 |
| CSV Export | 推奨 |
| Analytics Cache | 推奨 |
| Webhook Retry | 必須 |

---

## 17-2. 推奨構成

```txt
Redis
BullMQ
Worker Container
```

---

# 18. Docker構成推奨

```txt
nginx
nextjs
postgres
redis
worker
formbricks
```

---

# 19. 将来拡張

将来的に以下対応を想定。

- branching logic
- multilingual survey
- PRO scoring
- AI summarization
- FHIR Questionnaire
- CDISC ODM export
- adaptive survey
- scheduled survey
- multi-study support
- multi-tenant support

---

# 20. 実装優先順位

## Phase 1

- Survey Render
- Magic Link
- Manual Delivery
- Response Save
- CSV Export

---

## Phase 2

- Auto Reminder
- Analytics Cache
- Dashboard
- Versioning強化

---

## Phase 3

- EMR/HIS連携
- FHIR
- AI分析
- 多施設対応

---

# 21. 備考

本設計では、
Formbricks を「Survey Engine」として限定利用し、
医療研究に必要な監査・再現性・分析性・拡張性は
ePRO Core 側で担保する。

この設計により、
Formbricks の upgrade や将来変更の影響を最小化する。


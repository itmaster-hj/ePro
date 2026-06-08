以下は ePRO Platform の正式 Prisma schema です。

制約:

* PostgreSQL
* Prisma v6
* multiSchema 使用
* Next.js App Router
* soft delete 必須
* audit log 必須
* Formbricks 連携予定

この schema を元に:

1. migration 作成
2. seed.ts 作成
3. repository layer 作成
4. zod schema 作成
5. RBAC middleware 作成

を実装してください。

# CLAUDE.md - Next.js 15 SaaS 项目指导手册

## 项目概述
这是一个使用 Next.js 15 (App Router) 构建的 SaaS 应用，后端数据库采用 SQLite（本地开发优先使用 better-sqlite3，生产/边缘环境优先使用 Turso/libSQL）。 
目标是构建可扩展、多租户、安全的 SaaS 产品，支持订阅计费、用户隔离数据，并保持高性能和低运维成本。

核心原则：
- **Server Components 优先**：尽可能在 Server Components / Server Actions 中处理数据和逻辑，减少客户端 bundle 大小和 hydration 开销。
- **边缘友好**：兼容 Vercel Edge / Cloudflare 等无状态环境（Turso 优势在此）。
- **强类型 + 显式错误处理**：TypeScript 严格模式，所有数据库操作必须有类型安全和错误边界。

## 技术栈（必须严格使用）
- **Framework**：Next.js 15+ App Router + React 19（Server Components / Server Actions 为主）
- **语言**：TypeScript（strict: true, noImplicitAny: true）
- **样式**：Tailwind CSS + shadcn/ui（或类似组件库）
- **数据库**：
 - 本地开发：better-sqlite3（同步、高性能，适合单进程）
 - 生产/多地域：Turso (libSQL) + @libsql/client（支持 HTTP + embedded replica）
 - ORM：**Drizzle ORM**（推荐，类型安全且与 Turso 原生兼容；不推荐 Prisma，因为 Drizzle 在 SQLite 上的迁移和查询性能更优）
- **认证**：NextAuth.js / Clerk / Lucia（根据已有实现选择，必须在 middleware 和 Server Actions 中双重校验）
- **其他**：Zod（验证）、React Hook Form + Zod Resolver、TanStack Query（客户端数据获取）

## 项目结构（推荐且必须遵循）
使用 src/ 目录分离配置与应用代码（Next.js 官方推荐，便于区分路由与业务逻辑）：

```
├── src/
│   ├── app/                    # Next.js App Router 页面
│   │   ├── (auth)/             # 认证相关路由（route groups）
│   │   ├── (dashboard)/       # 需要登录的页面
│   │   ├── (marketing)/       # 营销/落地页
│   │   ├── api/                # API 路由（REST）
│   │   ├── layout.tsx         # 根布局（providers 包裹）
│   │   └── error.tsx          # 全局错误边界
│   ├── components/
│   │   ├── ui/                 # shadcn/ui 基础组件（Button, Input...）
│   │   ├── forms/             # 表单组件（使用 React Hook Form + Zod）
│   │   └── features/          # 业务组件
│   ├── lib/
│   │   ├── db.ts              # Drizzle 客户端（单例）
│   │   ├── auth.ts            # 认证逻辑（getSession, getCurrentUser）
│   │   └── utils.ts           # 工具函数（formatDate, cn...）
│   ├── server/
│   │   ├── db/                # Drizzle 表定义 + 种子数据
│   │   │   ├── schema.ts      # 表结构（users, sessions, accounts...）
│   │   │   ├── relations.ts   # 表关系定义
│   │   │   └── seed.ts        # 开发环境种子数据
│   │   └── actions/           # Server Actions（替代传统 API）
│   │       ├── auth.actions.ts
│   │       └── product.actions.ts
│   └── types/                  # 全局 TypeScript 类型
│       ├── next.d.ts          # Next.js 类型扩展
│       └── db.d.ts            # Drizzle 生成的类型
├── public/                    # 静态资源
├── scripts/                   # 一次性脚本
│   ├── db/
│   │   ├── migrate.ts         # 迁移脚本（node tsx scripts/db/migrate.ts）
│   │   └── seed.ts            # 种子数据（node tsx scripts/db/seed.ts）
│   └── sync.ts                # 数据同步脚本
├── tests/                     # 测试文件（Vitest + Playwright）
├── .env.example               # 环境变量示例（不要提交 .env.local）
├── drizzle.config.ts          # Drizzle 配置
├── next.config.ts             # Next.js 配置
├── tailwind.config.ts         # Tailwind 配置
├── tsconfig.json              # TypeScript 配置
└── package.json
```

**注意**：
- `src/components/features/` 下的组件应该按业务模块组织（如 `auth/`, `dashboard/`, `products/`），而非按组件类型（presentation vs container）。
- 避免 `src/app/api/` 中的 REST API（除非需要与外部系统集成），优先使用 **Server Actions**。

## 数据库设计原则（必须遵循）

### 表命名：snake_case，复数形式
```typescript
// ✅ 正确
export const users = sqliteTable('users', {...});
export const orderItems = sqliteTable('order_items', {...});

// ❌ 错误
export const User = sqliteTable('User', {...});
```

### 列命名：snake_case，外键用 `{table}_id` 格式
```typescript
// ✅ 正确
createdAt: text('created_at').notNull(),
userId: integer('user_id').references(() => users.id),

// ❌ 错误
createdAt: text('createdAt'),
userId: integer('userId'),
```

### 必须使用自增整数 ID（SQLite 最佳实践）
```typescript
// ✅ 正确
id: integer('id').primaryKey(),

// ❌ 避免（UUID 额外开销大，除非有分布式需求）
id: text('id').primaryKey(),
```

### 外键必须（重要！）
```typescript
// ✅ 正确：显式外键约束
userId: integer('user_id').references(() => users.id).notNull(),

// ❌ 错误：没有外键，关系无法在数据库层面保证
userId: integer('user_id').notNull(),
```

### 索引：为高频查询创建
```typescript
// ✅ 正确
index(['userId', 'status']),  // 复合索引
index(['email']),             // 唯一约束

// ❌ 错误：所有列都加索引（影响写入性能）
```

### 迁移：始终可回滚（重要！）
```typescript
// ✅ 正确：Drizzle migrations 自动生成可回滚脚本
// 生成的迁移文件包含 .up.ts 和 .down.ts

// ❌ 禁止：手动写不可回滚的 DDL
```

## 命名规范（必须遵循）

### 文件命名
- **组件/页面**：PascalCase - `UserProfile.tsx`, `LoginPage.tsx`
- **Server Actions**：PascalCase - `CreateOrder.ts`, `UpdateUser.ts`
- **工具/帮助函数**：camelCase - `formatDate.ts`, `validateEmail.ts`
- **数据库表定义**：kebab-case - `user-auth.ts`, `subscription-plan.ts`
- **类型/接口**：PascalCase - `UserProfile.ts`, `OrderType.ts`

### 数据库命名
- **表名**：snake_case，复数 - `users`, `order_items`
- **列名**：snake_case - `created_at`, `is_active`
- **索引名**：idx_{table}_{columns} - `idx_users_email`
- **外键名**：fk_{table}_{ref_table} - `fk_orders_users`

### URL 路由
- **路径**：kebab-case - `/user-profile`, `/order-details`
- **动态参数**：camelCase - `/products/[productId]`
- **API 端点**：RESTful - `GET /api/v1/users`, `POST /api/v1/orders`

## 开发命令（必须使用）

```bash
# 安装依赖
npm install

# 开发模式（监听文件变化自动重启）
npm run dev

# 类型检查（部署前必须通过）
npm run typecheck

# 代码检查
npm run lint

# 数据库
npm run db:generate       # 生成 Drizzle 类型（每次修改 schema 后运行）
npm run db:push          # 推送 schema 到数据库（开发环境）
npm run db:migrate       # 运行迁移（生产环境）
npm run db:seed          # 种子数据
npm run db:studio        # 本地 GUI 查看数据（http://localhost:5323）

# 测试
npm run test             # 单元测试（Vitest）
npm run test:e2e         # E2E 测试（Playwright）

# 构建
npm run build            # 生产构建（部署前必须通过）
npm run start            # 生产环境运行
```

## 模式（必须遵循）

### 1. Server Actions 优先于 API 路由
```typescript
// ✅ 正确：使用 Server Action
// src/server/actions/user.actions.ts
'use server'
import { z } from 'zod'
import { db } from '@/lib/db'
import { users } from '@/server/db/schema'
import { revalidatePath } from 'next/cache'

const CreateUserSchema = z.object({
  email: z.string().email(),
  name: z.string().min(1).max(100),
})

export async function createUser(data: z.infer<typeof CreateUserSchema>) {
  const validated = CreateUserSchema.parse(data)
  await db.insert(users).values(validated)
  revalidatePath('/users')
}

// 调用方式（Client Component）
'use client'
import { createUser } from '@/server/actions/user.actions'
<form action={createUser}>...</form>

// ✅ 正确：或使用 useFormState (需要客户端状态时)
import { useFormState } from 'react-dom'
const [state, formAction] = useFormState(createUser, null)

// ❌ 避免：API 路由（仅用于外部系统集成或需要 CORS 时）
// /app/api/users/route.ts
```

### 2. 数据获取用 Server Components（默认）
```typescript
// ✅ 正确：Server Component 直接查询
// src/app/(dashboard)/users/page.tsx
import { db } from '@/lib/db'
import { users } from '@/server/db/schema'

export default async function UsersPage() {
  const allUsers = await db.select().from(users).all()
  return (
    <div>
      {allUsers.map(user => (
        <div key={user.id}>{user.name}</div>
      ))}
    </div>
  )
}

// ✅ 正确：带 loading 的流式渲染
import { Suspense } from 'react'
import { UsersList } from './_components/UsersList'

export default function Page() {
  return (
    <Suspense fallback={<UsersListSkeleton />}>
      <UsersList />
    </Suspense>
  )
}

// ❌ 避免：在 Client 组件中发起 fetch 请求（除非需要客户端实时更新）
// 'use client'
// const { data } = useSWR('/api/users', fetcher)
```

### 3. 表单验证用 Zod + React Hook Form
```typescript
// ✅ 正确
// src/components/forms/CreateUserForm.tsx
'use client'
import { useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { CreateUserSchema } from '@/server/actions/user.actions'

export function CreateUserForm() {
  const form = useForm<z.infer<typeof CreateUserSchema>>({
    resolver: zodResolver(CreateUserSchema),
  })

  return (
    <form onSubmit={form.handleSubmit(onSubmit)}>
      <input {...form.register('name')} />
      {form.formState.errors.name && <span>{form.formState.errors.name.message}</span>}
      <button type="submit">Submit</button>
    </form>
  )
}
```

### 4. 错误处理用 try/catch + 错误边界
```typescript
// ✅ 正确：Server Action 错误处理
export async function createUser(data: UserData) {
  try {
    await db.insert(users).values(data)
  } catch (error) {
    if (error instanceof z.ZodError) {
      return { error: error.errors }
    }
    throw error  // 重新抛出非验证错误
  }
}

// ✅ 正确：错误边界（app/error.tsx）
'use client'
export default function Error({
  error,
  reset,
}: {
  error: Error & { digest?: string }
  reset: () => void
}) {
  return (
    <div className="p-4 border border-red-500 rounded">
      <h2>出错了！</h2>
      <p>{error.message}</p>
      <button onClick={() => reset()}>重试</button>
    </div>
  )
}
```

### 5. 客户端状态用 Zustand（简单）或 TanStack Query（复杂）
```typescript
// ✅ 简单：Zustand（仅 UI 状态）
// src/stores/ui-store.ts
import { create } from 'zustand'

interface UIState {
  sidebarOpen: boolean
  toggleSidebar: () => void
}

export const useUIStore = create<UIState>((set) => ({
  sidebarOpen: true,
  toggleSidebar: () => set((state) => ({ sidebarOpen: !state.sidebarOpen })),
}))

// ✅ 复杂：TanStack Query（服务端状态缓存）
// src/lib/query-client.ts
import { QueryClient } from '@tanstack/react-query'

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 5 * 60 * 1000,
      retry: 1,
    },
  },
})
```

### 6. 使用事务处理多表操作
```typescript
// ✅ 正确：事务
await db.transaction(async (tx) => {
  const [order] = await tx.insert(orders).values({...}).returning()
  await tx.insert(orderItems).values(
    items.map(item => ({ ...item, orderId: order.id }))
  )
})

// ❌ 错误：分散的操作（无事务保护）
const [order] = await db.insert(orders).values({...}).returning()
await db.insert(orderItems).values([...]) // 如果这里失败，order 无法回滚
```

## Anti-Patterns（禁止）

### 1. 禁止直接拼接 SQL（SQL 注入风险）
```typescript
// ❌ 危险：SQL 注入
const query = `SELECT * FROM users WHERE email = '${email}'`

// ✅ 安全：Drizzle 参数化查询
const result = await db.select().from(users).where(eq(users.email, email))
```

### 2. 禁止在 Client Component 发起不必要的 fetch
```typescript
// ❌ 错误：可以在 Server Action 完成的工作
'use client'
async function submit() {
  const res = await fetch('/api/users', {...})
}

// ✅ 正确：使用 Server Action
'use client'
<form action={createUser}>...</form>
```

### 3. 禁止在 Server Component 顶层使用 await（阻塞渲染）
```typescript
// ❌ 错误：会阻塞整个页面渲染
export default async function Page() {
  const data = await expensiveOperation() // 整个页面等待
  return <div>{data}</div>
}

// ✅ 正确：使用 Suspense 流式渲染
export default function Page() {
  return (
    <Suspense fallback={<Skeleton />}>
      <DataSection />
    </Suspense>
  )
}
```

### 4. 禁止使用 `any` 类型
```typescript
// ❌ 错误
const result = db.query('SELECT * FROM users') as any

// ✅ 正确：Drizzle 自动生成类型
const result = await db.select().from(users) // 类型自动推断
```

### 5. 禁止在数据库操作中省略错误处理
```typescript
// ❌ 错误
await db.insert(users).values(data)

// ✅ 正确
try {
  await db.insert(users).values(data)
} catch (error) {
  console.error('Failed to create user:', error)
  throw new Error('Failed to create user')
}
```

### 6. 禁止提交 .env 文件到版本控制
```typescript
// ✅ 正确：.env.local 已配置
// .gitignore 包含
.env.local
.env*.local

// ❌ 错误：提交敏感信息
```

## 环境变量（必须配置）

创建 `.env.local`（不要提交到版本控制）：

```bash
# 数据库（本地 better-sqlite3）
DATABASE_URL=file:./local.db

# 或 Turso（生产/边缘）
DATABASE_URL=libsql://your-db.turso.io
TURSO_AUTH_TOKEN=your-auth-token

# 认证（NextAuth.js 为例）
NEXTAUTH_SECRET=your-secret-min-32-chars
NEXTAUTH_URL=http://localhost:3000

# 可选：第三方服务
ANTHROPIC_API_KEY=sk-ant-...
OPENAI_API_KEY=sk-...
```

## 测试策略

### 单元测试（Vitest）
- 测试业务逻辑（actions, utils）
- 测试数据库操作（mock 或使用 test database）
- 目标：核心逻辑 80%+ 覆盖

```bash
npm run test
```

### E2E 测试（Playwright）
- 测试关键用户流程（注册、登录、创建订单）
- 每个功能至少一个测试

```bash
npm run test:e2e
```

### 测试数据库
- 使用独立的 test database（memory SQLite 或 separate file）
- 每个测试前清理数据（fixture）

```typescript
// tests/helpers.ts
export async function createTestUser(db: DB) {
  return await db.insert(users).values({
    email: 'test@example.com',
    name: 'Test User',
  }).returning()
}
```

## 部署检查清单

- [ ] `npm run typecheck` 通过
- [ ] `npm run lint` 无警告
- [ ] `npm run build` 成功
- [ ] `npm run test` 通过
- [ ] 环境变量已配置（生产数据库）
- [ ] 数据库迁移已运行
- [ ] 错误边界已设置（`app/error.tsx`, `app/not-found.tsx`）
- [ ] 静态资源已优化（图片、字体）

## 快速开始

```bash
# 1. 创建项目
npx create-next-app@latest my-saas --typescript --tailwind --eslint --app --src-dir --import-alias "@/*"

# 2. 安装核心依赖
npm install drizzle-orm @libsql/isomorphic-fetch better-sqlite3
npm install -D drizzle-kit

# 3. 初始化 Drizzle
npx drizzle-kit init

# 4. 创建数据库 schema（参考上方 schema.ts）
# 5. 生成类型
npm run db:generate

# 6. 复制本文件到项目根目录

# 7. 运行
npm run dev
```
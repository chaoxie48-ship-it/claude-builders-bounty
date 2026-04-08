# CLAUDE.md — Next.js 15 + SQLite SaaS 项目模板

## 技术栈

- **框架**: Next.js 15 (App Router)
- **数据库**: SQLite (better-sqlite3 或 Turso)
- **ORM**: Prisma 或 Drizzle
- **样式**: Tailwind CSS
- **Node**: 20+

---

## 项目结构

```
src/
├── app/                    # Next.js App Router 页面
│   ├── (auth)/            # 认证相关路由 (用 route groups)
│   ├── (dashboard)/       # 需要登录的页面
│   ├── api/               # API 路由
│   └── layout.tsx         # 根布局
├── components/            # React 组件
│   ├── ui/                # 可复用 UI 组件
│   └── features/          # 业务组件
├── lib/                   # 工具函数
│   ├── db.ts              # 数据库客户端
│   └── utils.ts           # 通用工具
├── server/                # 服务端逻辑
│   └── actions.ts         # Server Actions (替代 API 路由)
└── types/                 # TypeScript 类型
```

---

## 命名规范

### 文件命名
- **组件**: `PascalCase` - `UserProfile.tsx`
- **工具函数**: `camelCase` - `formatDate.ts`
- **常量**: `UPPER_SNAKE_CASE` - `MAX_UPLOAD_SIZE`
- **类型**: `PascalCase` - `UserType.ts`

### 数据库表命名
- **表名**: `snake_case` 复数 - `users`, `order_items`
- **列名**: `snake_case` - `created_at`, `user_id`
- **外键**: `{table}_id` - `user_id`, `order_id`

### 路由命名
- **URL 路径**: `kebab-case` - `/user-profile`, `/order-details`
- **API 端点**: `/api/v1/{resource}` 格式

---

## 数据库迁移规则

### 1. 始终使用显式外键
```sql
-- ✅ 正确
CREATE TABLE orders (
  id INTEGER PRIMARY KEY,
  user_id INTEGER REFERENCES users(id) ON DELETE CASCADE
);

-- ❌ 错误: 没有外键约束
CREATE TABLE orders (
  id INTEGER PRIMARY KEY,
  user_id INTEGER
);
```

### 2. 迁移脚本必须可回滚
```sql
-- ✅ 正确: 添加列
ALTER TABLE users ADD COLUMN email TEXT;

-- 迁移文件应该包含 UP 和 DOWN
-- DOWN: ALTER TABLE users DROP COLUMN email;
```

### 3. 不要删除有数据的列
- 先将列标记为废弃，保留 30 天后再删除
- 使用 `ALTER TABLE ... RENAME TO` 而非直接删除

### 4. 使用自增 ID 而非 UUID
- SQLite 推荐使用 `INTEGER PRIMARY KEY AUTOINCREMENT`
- 性能更好，存储更小

### 5. 索引规则
```sql
-- 经常查询的列加索引
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_orders_status ON orders(status);

-- 复合索引按查询频率排序
CREATE INDEX idx_orders_user_status ON orders(user_id, status);
```

---

## 开发命令

```bash
# 安装依赖
npm install

# 运行开发服务器
npm run dev

# 数据库迁移
npx prisma migrate dev      # 开发环境
npx prisma migrate deploy   # 生产环境

# 生成 Prisma Client
npx prisma generate

# 代码检查
npm run lint

# 类型检查
npm run typecheck

# 运行测试
npm run test
```

---

## 必须遵循的模式

### 1. Server Actions 优先
```typescript
// ✅ 正确: 使用 Server Actions
'use server'
export async function createOrder(formData: FormData) {
  'use server'
  // 服务端逻辑
}

// ❌ 避免: API 路由
// /app/api/orders/route.ts
```

### 2. 数据获取用 Server Components
```typescript
// ✅ 正确: Server Component 直接查询
async function UserList() {
  const users = await db.query('SELECT * FROM users');
  return <ul>{users.map(...)}</ul>;
}

// ❌ 避免: Client Component 发起请求
// 在 Client 组件里调用 API
```

### 3. 使用事务处理多表操作
```typescript
// ✅ 正确
await db.transaction(async (tx) => {
  await tx.orders.create({...});
  await tx.orderItems.createMany([...]);
});

// ❌ 错误: 分散的事务
await db.orders.create({...});
await db.orderItems.createMany([...]); // 如果这里失败，前面无法回滚
```

### 4. 表单验证用 Zod
```typescript
import { z } from 'zod'

const CreateUserSchema = z.object({
  email: z.string().email(),
  name: z.string().min(1).max(100),
  age: z.number().min(0).optional(),
})
```

---

## 禁止的模式 (Anti-Patterns)

### 1. 不允许直接操作 SQL 字符串
```typescript
// ❌ 危险: SQL 注入风险
const query = `SELECT * FROM users WHERE email = '${email}'`;

// ✅ 安全: 使用参数化查询
const query = 'SELECT * FROM users WHERE email = ?';
db.query(query, [email]);
```

### 2. 不允许在 Client Component 发请求
```typescript
// ❌ 错误: 可以在 Server Action 完成的逻辑
'use client'
async function submit() {
  const res = await fetch('/api/users', {...});
}

// ✅ 正确: 使用 Server Action
'use client'
<form action={createUser}>...</form>
```

### 3. 不允许裸 await 在组件顶层
```typescript
// ❌ 错误
const user = await getUser(); // 会阻塞渲染

// ✅ 正确: 用 Suspense
async function UserPage() {
  return (
    <Suspense fallback={<Loading />}>
      <UserInfo />
    </Suspense>
  );
}
```

### 4. 不允许在数据库操作中使用 `any` 类型
```typescript
// ❌ 错误
const result = db.query('SELECT * FROM users') as any;

// ✅ 正确: 定义类型
interface User {
  id: number;
  email: string;
  created_at: string;
}
const result = db.query<User[]>('SELECT * FROM users');
```

---

## 错误处理

### 1. 使用 Result 类型
```typescript
type Result<T, E = Error> = 
  | { ok: true; value: T }
  | { ok: false; error: E };

async function getUser(id: number): Promise<Result<User>> {
  try {
    const user = await db.query('SELECT * FROM users WHERE id = ?', [id]);
    return { ok: true, value: user };
  } catch (error) {
    return { ok: false, error: error as Error };
  }
}
```

### 2. 错误边界
```typescript
// app/error.tsx
'use client'
 
export default function Error({
  error,
  reset,
}: {
  error: Error & { digest?: string }
  reset: () => void
}) {
  return (
    <div>
      <h2>Something went wrong!</h2>
      <button onClick={() => reset()}>Try again</button>
    </div>
  )
}
```

---

## 安全规则

### 1. 敏感数据绝不传前端
```typescript
// ❌ 错误: 泄露密码哈希
const users = await db.query('SELECT id, email, password_hash FROM users');

// ✅ 正确: 只传必要字段
const users = await db.query('SELECT id, email FROM users');
```

### 2. 密码必须加盐哈希
```typescript
import { hash, verify } from '@node-rs/argon2';

const hash = await hash(password);
const valid = await verify(hash, password);
```

### 3. CORS 严格限制
```typescript
// next.config.js
async headers() {
  return [
    {
      source: '/api/:path*',
      headers: [
        { key: 'Access-Control-Allow-Origin', value: 'https://yourdomain.com' },
      ],
    },
  ];
}
```

---

## 部署检查清单

- [ ] `npm run build` 成功
- [ ] `npm run lint` 无警告
- [ ] `npm run typecheck` 无错误
- [ ] 数据库迁移脚本测试通过
- [ ] 环境变量配置完成
- [ ] 错误边界已设置

---

## 快速开始

```bash
# 1. 创建项目
npx create-next-app@latest my-saas --typescript --tailwind --eslint

# 2. 安装数据库
npm install better-sqlite3 prisma @prisma/client
npx prisma init

# 3. 复制本文件到项目根目录

# 4. 运行
npm run dev
```

---

## 参考资料

- [Next.js 15 文档](https://nextjs.org/docs)
- [Prisma 指南](https://www.prisma.io/docs)
- [SQLite 最佳实践](https://www.sqlite.org/whentouse.html)

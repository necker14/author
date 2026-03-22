# Author 自建后端架构方案

## 背景

Author 当前是一个**纯本地应用**——数据存在本机文件系统 `data/` 目录，通过 Next.js API Routes 读写。这种模式只适合单机使用。要支持多设备同步、移动端、多用户访问，需要升级为**云端后端**。

由于目标用户主要在中国，Firebase/Google 服务不可用，因此选择**自建后端**。

## 你现在有什么

```
author-app/
├── app/api/                     ← 15 个 API 路由（已有后端！）
│   ├── storage/route.js         ← 核心：读写用户数据（JSON 文件）
│   ├── ai/route.js              ← AI 对话（转发到各 AI 供应商）
│   ├── ai/gemini/route.js       ← Gemini 相关
│   ├── ai/claude/route.js       ← Claude 相关
│   ├── ai/responses/route.js    ← AI 响应
│   ├── balance/route.js         ← 余额查询
│   ├── embed/route.js           ← 向量嵌入
│   ├── parse-file/route.js      ← 文件解析
│   └── ...
├── docker-compose.yml           ← Docker 部署配置
├── docker-compose.caddy.yml     ← Caddy 反向代理 + HTTPS
└── data/                        ← 用户数据（JSON 文件）
```

> [!IMPORTANT]
> 你的 Next.js API Routes **就是后端**。改造重点只在 `storage/route.js`——从读写文件改为读写数据库。其他 AI 路由基本不用动。

## 升级方案：加数据库

### 架构对比

```
现在（单机模式）:
  浏览器 → Next.js → 本地 data/ 文件夹
                      ❌ 只能本机访问

升级后（云端模式）:
  手机/电脑/网页 → Next.js → PostgreSQL 数据库
                              ✅ 任何设备访问
```

### 技术选型

| 组件 | 选择 | 理由 |
|------|------|------|
| **数据库** | PostgreSQL | 成熟稳定，JSON 支持好，免费 |
| **ORM** | Prisma | Node.js 生态最流行，类型安全 |
| **认证** | NextAuth.js | 与 Next.js 深度集成，支持多种登录方式 |
| **文件存储** | 腾讯云 COS 或 阿里云 OSS | 国内 CDN 快，便宜 |
| **部署** | Docker Compose on VPS | 你已经会了 |

---

## 具体要改什么

### 阶段 1：加数据库（核心改造）

#### 新增文件

| 文件 | 作用 |
|------|------|
| `prisma/schema.prisma` | 数据库模型定义 |
| `app/lib/db.js` | Prisma 客户端单例 |
| `docker-compose.prod.yml` | 生产环境 compose（含 PostgreSQL） |

#### 修改文件

| 文件 | 改动 |
|------|------|
| `app/api/storage/route.js` | 从 `fs.readFile/writeFile` → `prisma.userData.findUnique/upsert` |
| `package.json` | 加 `prisma`、`@prisma/client` 依赖 |

#### 数据库模型（简化版）

```prisma
model User {
  id        String   @id @default(cuid())
  email     String?  @unique
  name      String?
  createdAt DateTime @default(now())
  data      UserData[]
}

model UserData {
  id        String   @id @default(cuid())
  userId    String
  key       String        // "author-chapters-work-default"
  value     Json          // 整个 JSON 数据
  updatedAt DateTime      @updatedAt
  user      User          @relation(fields: [userId], references: [id])

  @@unique([userId, key])  // 一个用户一个 key 只有一条
}
```

#### storage/route.js 改造对比

```diff
// 读取
- const content = await fs.readFile(filePath, 'utf-8');
- return NextResponse.json({ data: JSON.parse(content) });
+ const record = await prisma.userData.findUnique({
+   where: { userId_key: { userId, key } }
+ });
+ return NextResponse.json({ data: record?.value ?? null });

// 写入
- await fs.writeFile(filePath, JSON.stringify(value), 'utf-8');
+ await prisma.userData.upsert({
+   where: { userId_key: { userId, key } },
+   update: { value },
+   create: { userId, key, value },
+ });
```

### 阶段 2：加用户认证

| 功能 | 实现 |
|------|------|
| 邮箱登录 | NextAuth.js + 邮箱验证码 |
| 微信登录 | NextAuth.js + 微信开放平台 |
| 手机号登录 | 接短信 API（阿里云/腾讯云） |

### 阶段 3：移动端

加了数据库和认证后，移动端（Flutter 或 PWA）直接调同一套 API。

---

## 部署架构

```yaml
# docker-compose.prod.yml
services:
  author-app:
    image: yuanshijiloong/author:latest
    environment:
      - DATABASE_URL=postgresql://author:password@db:5432/author
    depends_on:
      - db

  db:
    image: postgres:16-alpine
    volumes:
      - pgdata:/var/lib/postgresql/data
    environment:
      - POSTGRES_USER=author
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=author

  caddy:
    image: caddy:2-alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile

volumes:
  pgdata:
```

## 成本估算

| 项目 | 月费 | 说明 |
|------|------|------|
| VPS（腾讯云轻量） | ¥50-100 | 2核4G，50G SSD |
| 域名 | ¥5-10 | .com 约 ¥60/年 |
| 对象存储 | ¥0-10 | 按量付费，少量免费 |
| SSL 证书 | ¥0 | Caddy 自动签发 Let's Encrypt |
| **合计** | **¥55-120/月** | 可以撑 **数千用户** |

## 对比总结

| | 现在 | 升级后 |
|--|------|--------|
| 数据存储 | 本机文件 | PostgreSQL 数据库 |
| 用户系统 | Cookie ID | 邮箱/手机号登录 |
| 多设备 | ❌ | ✅ 手机+电脑+网页 |
| 数据安全 | 文件可能损坏 | 数据库事务保证 |
| 并发性能 | 文件竞争（已修过！） | 数据库天然支持 |
| 改动量 | — | 主要改 1 个文件 |

## 实施顺序

1. **安装 Prisma + 定义模型**（30分钟）
2. **改造 `storage/route.js`**（1小时）
3. **本地测试**：Docker Compose 跑 PostgreSQL
4. **加用户认证**（半天）
5. **部署到 VPS 测试**
6. **做移动端适配**

# User Authentication API

一個以 NestJS 實作的使用者註冊 / 登入 API，作為熟悉 NestJS + Prisma + JWT 的練習專案。

## 技術棧

| 分類 | 使用 |
|---|---|
| 後端框架 | NestJS 11 |
| 資料庫 | SQLite（開發用，透過 Prisma 6 存取） |
| ORM | Prisma |
| 認證 | JWT（`@nestjs/jwt`） |
| 密碼雜湊 | bcrypt |
| 環境設定 | `@nestjs/config` |
| 驗證 | class-validator + class-transformer |

## 專案結構

```
my-project/
├── prisma/
│   ├── schema.prisma         # 資料模型定義
│   └── migrations/           # 版本控制的 schema 變更
├── src/
│   ├── main.ts               # 應用進入點、全域 ValidationPipe
│   ├── app.module.ts         # 根模組（載入 ConfigModule）
│   ├── prisma/               # PrismaService（DB client 生命週期管理）
│   ├── auth/                 # 註冊 / 登入 / JWT Guard
│   │   ├── dto/              # 請求資料驗證定義
│   │   └── jwt-auth.guard.ts # 保護需登入的 route
│   └── users/                # 使用者資料存取 / 個人資訊 API
└── .env                      # 環境變數（未進版控）
```

## 環境設定

### 需求

- Node.js 20+
- npm

### 安裝

```bash
npm install
```

### 環境變數

在專案根目錄建立 `.env`：

```
DATABASE_URL="file:./dev.db"
JWT_SECRET="<請用 openssl rand -hex 32 產生一組亂數>"
```

`JWT_SECRET` 千萬不要 commit 到版控，`.gitignore` 已排除 `.env`。

### 建立資料庫

```bash
npx prisma migrate deploy
npx prisma generate
```

第一次會依照 `prisma/schema.prisma` 建立 `dev.db` 並跑 migration。

## 啟動

```bash
# 開發模式（檔案變更自動重啟）
npm run start:dev

# 一般啟動
npm run start

# 正式模式
npm run build
npm run start:prod
```

伺服器預設監聽 `http://localhost:3000`。

## 資料模型

```prisma
model User {
  id        Int      @id @default(autoincrement())
  email     String   @unique
  password  String   // bcrypt 雜湊後儲存，永遠不回傳給 client
  name      String?
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}
```

## API 說明

所有請求 / 回應皆為 JSON。

### `POST /auth/register` — 註冊

**Request**
```json
{
  "email": "user@example.com",
  "password": "at-least-8-chars",
  "name": "Optional Name"
}
```

**驗證規則**
- `email` 必須為合法 email 格式
- `password` 至少 8 個字元
- `name` 可省略

**Response 201**
```json
{
  "message": "Register successfully",
  "user": {
    "id": 1,
    "email": "user@example.com",
    "name": "Optional Name",
    "createdAt": "...",
    "updatedAt": "..."
  }
}
```

**錯誤**
- `400`：欄位驗證失敗（回傳訊息陣列）
- `409`：Email 已註冊

---

### `POST /auth/login` — 登入

**Request**
```json
{
  "email": "user@example.com",
  "password": "at-least-8-chars"
}
```

**Response 201**
```json
{
  "message": "Login successfully",
  "accessToken": "eyJhbGciOi...",
  "user": {
    "id": 1,
    "email": "user@example.com",
    "name": "Optional Name"
  }
}
```

**錯誤**
- `400`：欄位驗證失敗
- `401`：帳號或密碼錯誤

---

### `GET /users/me` — 取得目前登入者資訊（需帶 token）

**Header**
```
Authorization: Bearer <accessToken>
```

**Response 200**
```json
{
  "id": 1,
  "email": "user@example.com",
  "name": "Optional Name",
  "createdAt": "...",
  "updatedAt": "..."
}
```

**錯誤**
- `401`：Token 缺失、格式錯誤、簽章無效或已過期

## 認證流程

1. 使用者呼叫 `POST /auth/register`，密碼經 bcrypt（`saltRounds = 10`）雜湊後存入資料庫
2. 使用者呼叫 `POST /auth/login`，後端比對雜湊，成功後簽發 JWT
3. JWT payload：`{ sub: userId, email, iat, exp }`，預設有效期 7 天
4. Client 後續請求在 header 帶 `Authorization: Bearer <token>`
5. 受保護路由（如 `GET /users/me`）以 `JwtAuthGuard` 驗證簽章與過期時間，通過後把 payload 掛在 `req.user`

## 安全性設計選擇

- 密碼永遠不會出現在任何 API 回應（`UsersService.createUser` 用 `select` 限制欄位；`/users/me` 用解構移除）
- `ValidationPipe` 全域啟用，`whitelist + forbidNonWhitelisted` 擋掉 request body 中未定義的欄位，避免大量賦值（mass assignment）漏洞
- JWT secret 從環境變數讀取，開機時透過 `ConfigService.getOrThrow` 檢查是否存在，遺漏時直接失敗
- Register 檢查 email 是否重複，發生衝突回 `409`

## 已知限制與後續可延伸

這是練習專案，範圍刻意收斂。以下是尚未實作但意識到的事：

- **未實作 refresh token**：目前 access token 直接有效 7 天。合理的做法是短期 access token（15 分鐘）+ refresh token 換發
- **未實作登出後端邏輯**：JWT 為無狀態，若需即時作廢需引入 token 黑名單或改用 session
- **密碼強度規則簡單**：僅檢查長度 ≥ 8，未強制大小寫、數字、特殊符號組合
- **未做 rate limiting**：`/auth/login` 未防暴力破解，之後可導入 `@nestjs/throttler`
- **無 e2e 測試**：目前手動用 curl / Postman 驗證，未來可用 `supertest` 補上
- **開發用 SQLite**：正式環境應改用 PostgreSQL 或其他 RDBMS，schema 已可直接切換

## 測試（手動）

啟動服務後：

```bash
# 1. 註冊
curl -X POST http://localhost:3000/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","password":"password123","name":"Test"}'

# 2. 登入拿 token
TOKEN=$(curl -s -X POST http://localhost:3000/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","password":"password123"}' \
  | node -pe "JSON.parse(require('fs').readFileSync(0)).accessToken")

# 3. 用 token 打受保護 route
curl http://localhost:3000/users/me \
  -H "Authorization: Bearer $TOKEN"
```

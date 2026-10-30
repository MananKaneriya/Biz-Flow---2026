# Bizflow API — Claude Code Build Prompt

Run this from the root of the `bizflow` monorepo. Paste everything below the line into Claude Code.

---

# CONTEXT

You are working on **Bizflow**, a business management system for a ceramic tile manufacturer, deliberately built to generalize to other small manufacturing and trading businesses. It covers items, stock, customers, orders, and dispatch.

It's a monorepo: one backend serves three clients.

```
bizflow/
├── apps/
│   ├── api/        Node.js + Express + PostgreSQL (Prisma) — YOUR SCOPE
│   ├── web/        React (Vite) — not yet built
│   ├── desktop/    Electron wrapper around apps/web — not yet built
│   └── mobile/     React Native (Expo) — not yet built
└── packages/
    └── shared/     API client + Zod validation schemas reused by web and mobile
```

**Your scope in this task is `apps/api` only.** Do not create or modify anything in `apps/web`, `apps/desktop`, or `apps/mobile`. You also create the root workspace files and `packages/shared`, because nothing exists yet (see the scaffold section below).

**Stack — do not substitute anything here:**

| Concern | Technology |
|---|---|
| Runtime | Node.js, ES modules (`"type": "module"`) |
| Framework | Express 4 |
| Database | PostgreSQL |
| ORM | Prisma (the system of record for ALL structured data) |
| Auth | JWT — access + refresh tokens, httpOnly cookies, bcrypt hashing |
| Validation | Zod (schemas live in `packages/shared`) |
| File uploads | multer (local staging) → Cloudinary (hosting) |

Cloudinary is used **only** for item image files. It never stores business data. Every record — users, customers, items, stock, movements, orders, order items, dispatches — lives in PostgreSQL via Prisma. Cloudinary returns a URL, that URL gets written to `Item.imageUrl` through a normal Prisma call, and that's the entire extent of its involvement.

---

# BEFORE YOU WRITE ANY CODE

1. **Database.** I provide a PostgreSQL database, and its connection strings go in `apps/api/.env` as `DATABASE_URL` and `DIRECT_URL` (a local Postgres or a hosted dev database; never production). If `apps/api/.env` does not exist, create it from `.env.example` with placeholder values, **stop, and ask me to fill it in** before running any migration. Never print secrets in your replies, and never commit `.env`.
2. **Migrations are only ever run with `prisma migrate dev` against that development database.** Never run `prisma migrate reset`, `prisma db push --force-reset` or any `DROP` statement without asking me first.
3. **Cloudinary credentials** are also mine to supply. Until they're in `.env`, build the image flow anyway and skip only the live upload check.
4. Check `node --version` (20 or newer) and `git status`. Work on a branch named `feat/api-backend`, not on `master`, and commit after each execution step.

---

# TASK

This is a **fresh start**. Build the monorepo scaffold, the `packages/shared` package and the whole of `apps/api` from scratch, following the layout below, with every module fully implemented.

The layout is layer-based (`controllers/`, `routes/`, `services/`, `middlewares/`, `utils/`, `db/`) and follows the conventions of a reference Express project I chose: the `ApiError` / `ApiResponse` / `asyncHandler` utilities, JWT cookie auth, and multer plus Cloudinary for uploads. We add a `services/` layer on top of that reference so business rules stay out of controllers.

## Monorepo scaffold (you create all of this)

The folder starts with only `.git`, `docs/`, `CLAUDE.md` and `.gitignore`. Nothing else exists. **Do not overwrite or delete `CLAUDE.md`, `.gitignore` or `docs/`**; if `.gitignore` is missing an entry you need, append to it.

```
bizflow/
├── package.json                     # private, "type": "module", npm workspaces: apps/*, packages/*; scripts: dev:api, test
├── README.md                        # short: what Bizflow is, folder map, how to run the API
├── apps/
│   └── api/                         # layout below
└── packages/
    └── shared/
        ├── package.json             # name "@bizflow/shared", "type": "module", exports "./constants", "./validation", "./api"; depends on zod
        └── src/
            ├── constants/index.js   # ROLES, ORDER_STATUSES, MOVEMENT_REASONS, ORDER_STATUS_TRANSITIONS, ORDER_TRANSITION_ROLES
            ├── validation/          # one Zod schema file per domain (auth, item, customer, stock, order, dispatch, pagination) + index.js
            └── api/                 # fetch-based client, one file per domain (auth.api.js, items.api.js, ...) + index.js
```

- Plain JavaScript with JSDoc, no TypeScript. ES modules everywhere.
- `apps/api` imports from `@bizflow/shared/constants` and `@bizflow/shared/validation`. **`zod` is installed only in `packages/shared`**, and `apps/api` never imports `zod` directly (the validate middleware only calls `schema.safeParse` and reads `.error.issues`).
- `ORDER_STATUS_TRANSITIONS` is the legal-transition map from business rule 3. `ORDER_TRANSITION_ROLES` maps each target status to the roles allowed to move an order into it (ADMIN is allowed everywhere).
- The API client in `packages/shared/src/api` uses `fetch` with `credentials: "include"`, takes the base URL as a parameter, understands the `ApiResponse` / `ApiError` JSON shape, and throws a typed error carrying `statusCode`, `message` and `errors`. It has one function per endpoint in the endpoint list below. It must not import anything from `apps/api`.
- The developer machine runs **Windows 11**. Keep every npm script cross-platform: no `rm -rf`, no `export VAR=value`, no bash-only syntax.

## apps/api layout

```
apps/api/
├── src/
│   ├── app.js                       # express config + route mounting ONLY, no logic
│   ├── index.js                     # entry: load env → connectDB() → app.listen()
│   ├── constants.js                 # app name, cookie options, pagination defaults
│   ├── db/
│   │   └── index.js                 # Prisma client singleton + connectDB()
│   ├── controllers/
│   │   ├── auth.controller.js
│   │   ├── customer.controller.js
│   │   ├── item.controller.js
│   │   ├── stock.controller.js
│   │   ├── order.controller.js
│   │   └── dispatch.controller.js
│   ├── routes/
│   │   ├── auth.routes.js
│   │   ├── customer.routes.js
│   │   ├── item.routes.js
│   │   ├── stock.routes.js
│   │   ├── order.routes.js
│   │   └── dispatch.routes.js
│   ├── services/
│   │   ├── auth.service.js
│   │   ├── customer.service.js
│   │   ├── item.service.js
│   │   ├── stock.service.js
│   │   ├── order.service.js
│   │   └── dispatch.service.js
│   ├── middlewares/
│   │   ├── auth.middleware.js       # verifyJWT
│   │   ├── role.middleware.js       # requireRole(...roles)
│   │   ├── validate.middleware.js   # Zod schema runner
│   │   ├── multer.middleware.js     # disk staging for uploads
│   │   └── error.middleware.js      # centralized error handler, mounted last
│   └── utils/
│       ├── ApiError.js
│       ├── ApiResponse.js
│       ├── asyncHandler.js
│       └── cloudinary.js
├── prisma/
│   ├── schema.prisma                # written in the DATABASE SCHEMA section; do not deviate
│   ├── migrations/
│   └── seed.js                      # creates the first ADMIN user
├── public/
│   └── temp/.gitkeep                # multer staging dir, contents gitignored
├── .env.example
└── package.json
```

**Build in this exact layout; don't improvise extra layers or folders.** If a requirement seems to need a file that isn't in the tree, ask first.

**Layer responsibilities — enforce this strictly:**

- **routes** — URL definitions, middleware chaining. Zero logic.
- **controllers** — HTTP in/out. Read `req`, call one service function, wrap in `ApiResponse`. If a controller contains an `if` statement about business meaning, that `if` belongs in the service.
- **services** — all business rules and all Prisma queries. No `req`/`res` awareness whatsoever; a service must be callable from a script or a test.

---

# BACKEND REQUEST FLOW

Every request follows the same path. Build and review all code against it.

**Startup:** `index.js` loads env → validates that every required env var is present (fail loudly, don't fail mysteriously later) → `connectDB()` → only then `app.listen()`. On `SIGTERM`/`SIGINT` it stops accepting requests, lets in-flight ones finish, disconnects Prisma, and exits.

**Per request** (example: `POST /api/v1/dispatch`):

1. Global middleware in `app.js`: `helmet` → `compression` → `cors` → `express.json` (16kb limit) → `cookieParser`
2. Router matches `/api/v1/dispatch` → `dispatch.routes.js`
3. Route middleware chain, in this order: `verifyJWT` → `requireRole("ADMIN", "DISPATCH")` → `validate(schema, "body")`
4. `dispatch.controller.js` — wrapped in `asyncHandler`; reads `req.body` and `req.user`, calls one service function
5. `dispatch.service.js` — opens one `prisma.$transaction`, applies the business rules, writes, returns plain data
6. The controller responds with `res.status(201).json(new ApiResponse(201, data, "Order dispatched"))`
7. **Failure at any step:** a thrown `ApiError` (or any other error) goes straight to `error.middleware.js`, which returns `{ statusCode, data: null, message, success: false, errors }`

**Rules of the flow**

- Middleware order is fixed: authenticate → authorize → validate → controller. Never parse or touch the body of a request that isn't allowed to be there.
- A controller calls exactly one service function per request. If it needs two, a service function is missing.
- Services may call other services (`order.service` → `stock.service`). They never call controllers and never reference `req`/`res`.
- Only services talk to Prisma. The one exception is `verifyJWT`, which does a single primary-key lookup.
- Service functions that write inside a transaction accept an optional `tx` client (defaulting to the shared `prisma` client). This lets `dispatchOrder` reuse `stock.service` helpers inside its single transaction instead of duplicating the ledger logic.

---

# ARCHITECTURE DECISIONS (already made — implement, don't relitigate)

| Decision | Choice | Reason |
|---|---|---|
| Layering | routes → middlewares → controllers → services → Prisma | Business rules are testable without HTTP; controllers stay thin |
| Data store | PostgreSQL via Prisma for all business data | Stock and dispatch need relational integrity and transactions |
| Files | Cloudinary for item images only | Keeps binaries out of the database; returns a URL we store |
| Auth | JWT access + refresh tokens in httpOnly cookies; Bearer header also accepted | Cookies for web, header for mobile and desktop clients |
| Auth lookup | `verifyJWT` does one indexed primary-key lookup per request, minimal `select` | Deactivated users and role changes apply immediately; costs about 1 ms |
| Errors | `ApiError` thrown, one error middleware | One response shape for every client |
| Deletes | Soft delete (`isActive`) | Orders and ledger rows reference these records permanently |
| Stock | Append-only ledger (`StockMovement`) plus atomic counter on `Stock` | Auditable and safe under concurrency |
| Pagination | Offset for small tables, cursor for large append-only tables | See Performance section |
| Caching | None initially | Add only after a measured hotspot |
| Testing | Vitest + supertest for the dispatch transaction and stock ledger only; Postman collection for everything else | Highest-risk paths get automation; the rest is checked manually |
| Workspace | npm workspaces; `zod` lives only in `packages/shared` | One copy of each schema shared by API, web and mobile; no duplicate validation libraries |
| First user | Created by `prisma/seed.js` from env vars | Registration is ADMIN-only, so an empty database needs a bootstrap admin |
| Existing code | None; greenfield | Everything in this prompt is built new |

---

# DATABASE SCHEMA — WRITE THIS FIRST

There is no existing schema. Create `apps/api/prisma/schema.prisma` exactly as below, then generate the first migration. These 8 models, their relations, indexes and defaults are decided: if something here looks wrong or conflicts with a requirement elsewhere in this prompt, stop and ask me instead of changing it silently.

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider  = "postgresql"
  url       = env("DATABASE_URL")
  directUrl = env("DIRECT_URL") // same value as DATABASE_URL on a local database; the non-pooled URL on Supabase/Neon
}

enum Role {
  ADMIN
  SALES
  PRODUCTION
  DISPATCH
}

enum MovementReason {
  PRODUCTION
  ORDER_DISPATCH
  ADJUSTMENT
  RETURN
}

enum OrderStatus {
  DRAFT
  CONFIRMED
  IN_PRODUCTION
  READY_FOR_DISPATCH
  DISPATCHED
  CANCELLED
}

model User {
  id               String          @id @default(uuid())
  name             String
  email            String          @unique
  passwordHash     String
  role             Role
  isActive         Boolean         @default(true)
  refreshTokenHash String? // SHA-256 of the current refresh token, never the raw token
  createdAt        DateTime        @default(now())
  updatedAt        DateTime        @updatedAt
  ordersCreated    Order[]
  dispatches       Dispatch[]
  stockMovements   StockMovement[]
}

model Customer {
  id        String   @id @default(uuid())
  name      String
  phone     String?
  address   String?
  gstin     String?
  isActive  Boolean  @default(true)
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  orders    Order[]

  @@index([isActive])
}

model Item {
  id            String          @id @default(uuid())
  designCode    String          @unique
  name          String
  category      String?
  size          String?
  unitPrice     Decimal         @db.Decimal(12, 2)
  imageUrl      String?
  imagePublicId String? // Cloudinary public_id, so replaced or deleted images can be removed reliably
  isActive      Boolean         @default(true)
  createdAt     DateTime        @default(now())
  updatedAt     DateTime        @updatedAt
  stock         Stock?
  movements     StockMovement[]
  orderItems    OrderItem[]

  @@index([isActive, category])
}

model Stock {
  id        String   @id @default(uuid())
  itemId    String   @unique
  item      Item     @relation(fields: [itemId], references: [id])
  quantity  Int      @default(0)
  updatedAt DateTime @updatedAt
}

model StockMovement {
  id          String         @id @default(uuid())
  itemId      String
  item        Item           @relation(fields: [itemId], references: [id])
  changeQty   Int // signed: positive adds stock, negative removes it
  reason      MovementReason
  orderId     String? // the order that caused it, when there is one
  order       Order?         @relation(fields: [orderId], references: [id])
  note        String?
  createdById String
  createdBy   User           @relation(fields: [createdById], references: [id])
  createdAt   DateTime       @default(now())

  @@index([itemId, createdAt])
  @@index([orderId])
}

model Order {
  id          String          @id @default(uuid())
  orderNumber Int             @unique @default(autoincrement()) // human-friendly number shown in the UI
  customerId  String
  customer    Customer        @relation(fields: [customerId], references: [id])
  status      OrderStatus     @default(DRAFT)
  totalAmount Decimal         @db.Decimal(14, 2)
  notes       String?
  createdById String
  createdBy   User            @relation(fields: [createdById], references: [id])
  createdAt   DateTime        @default(now())
  updatedAt   DateTime        @updatedAt
  items       OrderItem[]
  dispatch    Dispatch?
  movements   StockMovement[]

  @@index([status, createdAt])
  @@index([customerId, createdAt])
}

model OrderItem {
  id        String  @id @default(uuid())
  orderId   String
  order     Order   @relation(fields: [orderId], references: [id])
  itemId    String
  item      Item    @relation(fields: [itemId], references: [id])
  quantity  Int
  unitPrice Decimal @db.Decimal(12, 2) // price snapshot at order time; later Item price changes must not alter old orders

  @@unique([orderId, itemId])
  @@index([itemId])
}

model Dispatch {
  id             String   @id @default(uuid())
  orderId        String   @unique
  order          Order    @relation(fields: [orderId], references: [id])
  vehicleNumber  String
  notes          String?
  dispatchedById String
  dispatchedBy   User     @relation(fields: [dispatchedById], references: [id])
  dispatchedAt   DateTime @default(now())

  @@index([dispatchedAt])
}
```

Relationships: Customer 1→many Order · Order 1→many OrderItem · OrderItem →1 Item · Item 1→1 Stock · Item 1→many StockMovement · Order 1→1 Dispatch · User creates Orders, StockMovements and Dispatches.

**Rules that follow from the schema**

- Creating an `Item` creates its `Stock` row (`quantity: 0`) in the same transaction, so every item always has exactly one stock row. Starting at zero needs no ledger entry; every later change goes through the ledger.
- `unitPrice` on `OrderItem` is copied from `Item.unitPrice` at order creation, and `Order.totalAmount` is the sum of `quantity × unitPrice` over its lines, computed with `Prisma.Decimal` (never JavaScript numbers). Reject orders that contain an inactive item.
- Quantities are whole numbers in the item's selling unit. `OrderItem.quantity` and every dispatched quantity must be greater than zero (enforce in the Zod schema).

**Migrations**

1. `npx prisma migrate dev --name init` creates the tables and the indexes declared above.
2. Then `npx prisma migrate dev --create-only --name extras`, and put this in the generated SQL file before applying it:

```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX "Item_name_trgm_idx" ON "Item" USING GIN (name gin_trgm_ops);
CREATE INDEX "Customer_name_trgm_idx" ON "Customer" USING GIN (name gin_trgm_ops);
ALTER TABLE "Stock" ADD CONSTRAINT "Stock_quantity_nonnegative" CHECK (quantity >= 0);
```

The `CHECK` constraint is a second line of defence: even if application code has a bug, PostgreSQL itself refuses to store negative stock.

**First admin user.** Registration is ADMIN-only, so nobody could ever log in on an empty database. Create `apps/api/prisma/seed.js`, wired through `"prisma": { "seed": "node prisma/seed.js" }` in `apps/api/package.json`. It reads `SEED_ADMIN_NAME`, `SEED_ADMIN_EMAIL` and `SEED_ADMIN_PASSWORD` from the environment, refuses to run if the password is shorter than 12 characters, upserts the user by email with role `ADMIN`, hashes the password with bcryptjs, and is safe to run repeatedly. It never prints the password.

---

# UTILITIES — implement exactly this shape

`src/utils/ApiError.js`
```js
class ApiError extends Error {
  constructor(statusCode, message = "Something went wrong", errors = [], stack = "") {
    super(message);
    this.statusCode = statusCode;
    this.data = null;
    this.message = message;
    this.success = false;
    this.errors = errors;
    if (stack) {
      this.stack = stack;
    } else {
      Error.captureStackTrace(this, this.constructor);
    }
  }
}
export { ApiError };
```

`src/utils/ApiResponse.js`
```js
class ApiResponse {
  constructor(statusCode, data, message = "Success") {
    this.statusCode = statusCode;
    this.data = data;
    this.message = message;
    this.success = statusCode < 400;
  }
}
export { ApiResponse };
```

`src/utils/asyncHandler.js`
```js
const asyncHandler = (requestHandler) => {
  return (req, res, next) => {
    Promise.resolve(requestHandler(req, res, next)).catch((err) => next(err));
  };
};
export { asyncHandler };
```

**Every** controller function is wrapped in `asyncHandler`, throws `ApiError` on failure, and returns `res.status(code).json(new ApiResponse(code, data, message))` on success. No bare `try/catch` blocks returning ad-hoc `res.json({ error })` anywhere in the codebase. One response shape, everywhere, so the shared API client on the frontend can parse it uniformly.

---

# MIDDLEWARE

`auth.middleware.js` — accepts the token from either the `accessToken` httpOnly cookie or an `Authorization: Bearer` header:
```js
export const verifyJWT = asyncHandler(async (req, _, next) => {
  const token =
    req.cookies?.accessToken ||
    req.header("Authorization")?.replace("Bearer ", "");
  if (!token) throw new ApiError(401, "Unauthorized request");

  try {
    const decoded = jwt.verify(token, process.env.ACCESS_TOKEN_SECRET);
    const user = await prisma.user.findUnique({
      where: { id: decoded?.id },
      select: { id: true, name: true, email: true, role: true, isActive: true },
    });
    if (!user || !user.isActive) throw new ApiError(401, "Invalid access token");
    req.user = user;
    next();
  } catch (error) {
    throw new ApiError(401, error?.message || "Invalid access token");
  }
});
```
Note the `select` — never load `passwordHash` into `req.user`.

`role.middleware.js` — `requireRole(...roles)` returns middleware that throws `ApiError(403, ...)` if `req.user.role` isn't in the list. Always chained after `verifyJWT`.

`validate.middleware.js` — takes a Zod schema and a target (`"body"` / `"query"` / `"params"`), parses, and on failure throws `ApiError(400, "Validation failed", zodIssues)` so the client gets field-level errors. Import the schemas from `@bizflow/shared/validation` — those are the single source of truth and the web/mobile forms use the identical schemas.

`error.middleware.js` — mounted last in `app.js`. Converts anything thrown into the standard JSON shape:
```js
{ statusCode, data: null, message, success: false, errors: [] }
```
Handle Prisma's known error codes specifically: `P2002` → 409 with the conflicting field named, `P2025` → 404, `P2003` → 400 for foreign key violations. Return the stack trace only when `NODE_ENV !== "production"`.

---

# ROUTE PROTECTION AND PERMISSIONS

| Area | Read | Write |
|---|---|---|
| Items | any authenticated | ADMIN, PRODUCTION |
| Stock | any authenticated | ADMIN, PRODUCTION |
| Customers | ADMIN, SALES | ADMIN, SALES |
| Orders | ADMIN, SALES, PRODUCTION | ADMIN, SALES |
| Order status transitions | — | depends on target status: PRODUCTION can move to `IN_PRODUCTION`/`READY_FOR_DISPATCH`; SALES can confirm/cancel; ADMIN anything |
| Dispatch | ADMIN, DISPATCH | ADMIN, DISPATCH |
| User registration | ADMIN only | ADMIN only |

---

# ENDPOINTS TO BUILD

All under `/api/v1`.

**Auth** (`/auth`)
- `POST /login` — email + password → sets `accessToken` and `refreshToken` httpOnly cookies, returns user (never the hash)
- `POST /logout` — clears cookies and sets `User.refreshTokenHash` to null
- `POST /refresh-token` — verifies the refresh token, compares its SHA-256 hash with `User.refreshTokenHash`, then issues a new access token **and a new refresh token** (rotation) and stores the new hash. A token that doesn't match is rejected, and an inactive user is rejected. `login` also rejects inactive users.
- `POST /register` — ADMIN only, creates staff users
- `GET /current-user` — returns `req.user`
- `POST /change-password`

**Items** (`/items`)
- `GET /` — paginated (`?page=1&limit=20`), searchable (`?q=`), filterable by category/size
- `GET /:itemId`
- `POST /` — ADMIN/PRODUCTION, optional image upload
- `PATCH /:itemId`
- `PATCH /:itemId/image` — multipart, single file field `image`
- `DELETE /:itemId` — **soft delete via an `isActive` flag. Never hard-delete a business record.**

**Stock** (`/stock`)
- `GET /` — current levels joined with item info, paginated
- `GET /:itemId/movements` — the audit trail for one item, paginated
- `POST /adjust` — body `{ itemId, changeQty, reason, note? }`

**Customers** (`/customers`)
- `GET /` (paginated + search), `GET /:customerId`, `POST /`, `PATCH /:customerId`, soft `DELETE /:customerId`

**Orders** (`/orders`)
- `GET /` — filterable by status, customer, date range; paginated
- `GET /:orderId` — full detail with line items and customer
- `POST /` — creates order + all its line items in one transaction
- `PATCH /:orderId/status` — status transitions only

**Dispatch** (`/dispatch`)
- `POST /` — body `{ orderId, vehicleNumber }`
- `GET /` — dispatch history, paginated
- `GET /:dispatchId`

---

# THE THREE BUSINESS RULES THAT MATTER MOST

These are the rules the whole schema exists to protect. Get these wrong and the system loses its users' trust permanently.

### 1. Stock is a ledger, never a directly-writable number

No code path may ever write `Stock.quantity` on its own. Every single change inserts a `StockMovement` row carrying its reason and a reference to what caused it, and updates `Stock.quantity` in the **same transaction**. This is what makes "why did stock drop last Tuesday" answerable six months from now, and what stops silent inventory drift under concurrent writes.

For the concurrency case specifically: use an atomic `increment`/`decrement` in the Prisma update rather than read-then-write, so two simultaneous adjustments can't lose one another. Guard against negative stock with the atomic check-and-decrement pattern from the Performance section (a conditional `updateMany`, never read-compare-write) and throw `ApiError(409, ...)` if an adjustment would push stock below zero.

### 2. Dispatch is one atomic unit

`dispatchOrder` runs entirely inside a single `prisma.$transaction`. In one unit it must: create the `Dispatch` record, insert a negative `StockMovement` per line item with reason `ORDER_DISPATCH`, decrement each `Stock`, and update the `Order` status to `DISPATCHED`. All four commit or none do.

Before any of that, validate inside the same transaction that the order is in `READY_FOR_DISPATCH`, isn't already dispatched, and that every line item has sufficient stock. A crash midway that decrements stock without recording a dispatch is exactly the class of bug that costs real inventory.

Sort line items by `itemId` before updating stock so concurrent dispatches lock rows in the same order and cannot deadlock. This is the most business-critical endpoint in the system. Write it carefully and add tests for it.

### 3. Order status transitions are validated server-side

Never trust the client to send only legal transitions. The legal transition map lives in `packages/shared/constants` (`ORDER_STATUS_TRANSITIONS`, plus `ORDER_TRANSITION_ROLES` for who may trigger each move). You create it in the scaffold step; import and enforce it in `order.service.js` and never redefine it inside `apps/api`. An illegal transition throws `ApiError(400, ...)` naming both the current and attempted status.

Valid path: `DRAFT → CONFIRMED → IN_PRODUCTION → READY_FOR_DISPATCH → DISPATCHED`, with `CANCELLED` reachable from any pre-dispatch state and nothing reachable from `DISPATCHED` or `CANCELLED`.

---

# CLOUDINARY — item images only

`src/utils/cloudinary.js`:
```js
import { v2 as cloudinary } from "cloudinary";
import fs from "fs/promises";

cloudinary.config({
  cloud_name: process.env.CLOUDINARY_CLOUD_NAME,
  api_key: process.env.CLOUDINARY_API_KEY,
  api_secret: process.env.CLOUDINARY_API_SECRET,
});

const uploadOnCloudinary = async (localFilePath) => {
  if (!localFilePath) return null;
  try {
    const response = await cloudinary.uploader.upload(localFilePath, {
      resource_type: "image",
      folder: "bizflow/items",
    });
    return response;
  } catch (error) {
    return null;
  } finally {
    // always clean up the staged file, success or failure
    await fs.unlink(localFilePath).catch(() => {});
  }
};

const deleteFromCloudinary = async (publicId) => {
  if (!publicId) return null;
  try {
    return await cloudinary.uploader.destroy(publicId);
  } catch {
    return null;
  }
};

export { uploadOnCloudinary, deleteFromCloudinary };
```

`multer.middleware.js` — disk storage into `./public/temp`. Add real limits: `fileSize` capped at 5MB and a `fileFilter` rejecting anything that isn't `image/jpeg`, `image/png`, or `image/webp`. Generate a unique filename (uuid + original extension) rather than using `file.originalname` directly — two users uploading `tile.jpg` at once must not collide, and unsanitized filenames are a path-traversal risk.

Flow in the item module: multer stages the file → controller calls `uploadOnCloudinary(req.file.path)` → if it returns `null`, throw `ApiError(500, "Image upload failed")` → service writes `secure_url` to `Item.imageUrl` via Prisma. When replacing an existing image, call `deleteFromCloudinary` on the old asset so orphans don't accumulate — use the `Item.imagePublicId` column for this rather than parsing the id back out of the URL.

`public/temp/*` goes in `.gitignore` with `.gitkeep` excepted.

---

# CODE QUALITY REQUIREMENTS

Hold to these throughout. I care more about this being correct and maintainable than about it being finished fast.

**Correctness and data integrity**
- Every multi-write operation is wrapped in `prisma.$transaction`. No exceptions.
- Use atomic `increment`/`decrement` over read-modify-write for any counter.
- Money uses Prisma `Decimal`, never JavaScript floats. Don't do arithmetic on prices in JS — compute totals in the database or with a decimal library.
- Validate at the boundary with Zod; assume nothing about request shape inside services.

**Performance**
- **Every list endpoint is paginated.** No endpoint returns an unbounded collection — "return all orders" is fast in month one and fatal in year two. Use offset pagination (`{ data, page, limit, total, totalPages }`) for small tables and cursor pagination (`{ data, nextCursor, hasMore }`) for orders, stock movements and dispatch history; see the Performance section.
- **No N+1 queries.** Use Prisma `include`/`select` to fetch relations in one query. If you find yourself calling Prisma inside a `.map()`, that's the bug.
- **`select` only the fields actually needed.** Never fetch a whole table's columns to return three of them, and never let `passwordHash` leave the database layer.
- Use `Promise.all` for genuinely independent queries — the count and the page in a paginated response, for instance.
- Confirm the indexes the schema declares actually cover your hot query paths. Filtering orders by status and date range needs to hit an index.

**Structure and readability**
- Small, single-purpose functions with descriptive names. `dispatchOrder`, not `handleDispatch`.
- No duplicated logic. If pagination parsing appears in three services, it belongs in a shared helper.
- Comments explain *why*, not *what*. `// atomic to prevent lost updates under concurrent dispatch` earns its place; `// get the item` does not.
- Consistent ES module imports with explicit `.js` extensions throughout.
- No dead code, no commented-out blocks, no leftover `console.log`. Use a real logger if you add logging.

**Security**
- Secrets only from `process.env`. Nothing hardcoded, ever. Fail loudly at startup if a required env var is missing rather than failing mysteriously at request time.
- bcrypt with a cost factor of at least 10. Hash in a service helper so it can't be forgotten at a call site.
- httpOnly + `sameSite: "strict"` + `secure` in production for auth cookies.
- CORS locked to `process.env.CORS_ORIGIN` with `credentials: true`.
- `express.json({ limit: "16kb" })` — don't accept unbounded bodies.
- Add `helmet` and a rate limiter on `/auth/login` specifically.
- Never return `passwordHash` or `refreshToken` in any response.

---

# PERFORMANCE AND DATA STRUCTURES

Goal: every endpoint responds in milliseconds and stays that way as data grows. In an API like this, in-memory algorithms are rarely the bottleneck — database access is. Optimise in this order: indexes → query shape → pagination → in-code data structures → caching.

**Response-time targets** (measured locally against the seeded dataset described at the end of this section, excluding network latency):

| Endpoint type | Target |
|---|---|
| Single-record read | 5–20 ms |
| Paginated list or search | 10–40 ms |
| Write or transaction (order create, stock adjust, dispatch) | 20–80 ms |
| `POST /auth/login` | 50–150 ms. bcrypt is deliberately slow; do NOT lower the cost factor to hit a target |
| Image upload | Bound by Cloudinary and file size; not subject to these targets |

### 1. Indexes (B-tree)

These are already declared in `schema.prisma` (see DATABASE SCHEMA). Confirm each one exists in the generated migration and that the hot queries use them:

- `User`: unique `email`
- `Item`: unique `designCode`, index `(isActive, category)`
- `Customer`: index `isActive`
- `Order`: index `(status, createdAt)`, index `(customerId, createdAt)`
- `OrderItem`: index `orderId`, index `itemId`
- `Stock`: unique `itemId`
- `StockMovement`: index `(itemId, createdAt)`
- `Dispatch`: unique `orderId`, index `createdAt`

PostgreSQL does not index foreign keys automatically, so every FK column used in a join or filter needs its own index.

### 2. Search (trigram index)

`ILIKE '%term%'` — which is what Prisma's `contains` with `mode: "insensitive"` generates — cannot use a B-tree index. For item name/design code and customer name search, the trigram indexes are created in the `extras` migration from the DATABASE SCHEMA section. Confirm they exist. Reference SQL:

```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX item_name_trgm_idx ON "Item" USING GIN (name gin_trgm_ops);
CREATE INDEX customer_name_trgm_idx ON "Customer" USING GIN (name gin_trgm_ops);
```

If you also search `designCode`, add a matching trigram index in the same migration.

### 3. Pagination

- **Offset** (`page`/`limit`, response `{ data, page, limit, total, totalPages }`) for small, slow-growing tables: items, customers, users.
- **Cursor/keyset** for tables that grow without bound: orders, stock movements, dispatch history. Use Prisma `cursor` + `take` + `skip: 1` with a stable, indexed `orderBy: [{ createdAt: "desc" }, { id: "desc" }]`. Response `{ data, nextCursor, hasMore }`. Do not run `COUNT(*)` on these tables; a full count over millions of rows is itself a slow query. If the UI needs a total, expose it as a separate optional endpoint.
- Cap `limit` at 100 in the Zod schema; default 20. Put the shared parsing in one helper.

### 4. Query shape

- One query per resource with `include`/`select`. No queries inside loops or `.map()`.
- `select` only the columns the response uses.
- Independent queries run together with `Promise.all`.
- Bulk writes in one call: create an order and its lines with a nested `items: { create: [...] }` (a single atomic call), not a loop of inserts.
- One shared Prisma client, with the pool size set explicitly through `DATABASE_URL` (`?connection_limit=10`).
- In development only, subscribe to Prisma's `query` event and log a warning for any query slower than 100 ms, so slow queries surface while building.

### 5. In-code data structures and algorithms

- **`Map` / `Set`** (hash tables, O(1)) for every lookup or dedupe. Never `array.find()` or `array.includes()` inside a loop, because that is O(n²). Example: build `const stockByItemId = new Map(...)` once, then validate each order line against it.
- **Transition table.** The order status rules are a plain object such as `{ DRAFT: ["CONFIRMED", "CANCELLED"], ... }` from `packages/shared`. Validating a transition is one O(1) lookup.
- **Deterministic lock ordering.** Inside the dispatch transaction, sort line items by `itemId` ascending before updating stock. Two concurrent dispatches then take row locks in the same order and cannot deadlock.
- **Atomic check-and-decrement.** Never read stock, compare in JS, then write. Use one statement:

  ```js
  const { count } = await tx.stock.updateMany({
    where: { itemId, quantity: { gte: qty } },
    data: { quantity: { decrement: qty } },
  });
  if (count === 0) throw new ApiError(409, `Insufficient stock for item ${itemId}`);
  ```

  The condition and the decrement happen atomically in the database: no race window, no extra round trip.
- **Short transactions.** No network calls (Cloudinary, email) and no heavy computation inside `$transaction`. Do slow work before or after it.

### 6. HTTP level

- `compression` middleware (gzip) on JSON responses.
- Keep payloads lean: don't return nested objects or columns the client didn't ask for.

### 7. Caching, only when measured

Do not add caching up front. If profiling later shows a hot, rarely-changing read (the item list used by every order form, for example), add a small in-process TTL cache (a `Map` with expiry, or an LRU) in the service layer and invalidate it on every write to that resource. Do not introduce Redis in this task. An in-process cache is fine until there are multiple API instances.

### Performance verification

Create `prisma/seed.perf.js` (not part of normal setup) that generates roughly 100k orders, 300k order items and 500k stock movements across about 2k items and 5k customers. Against that dataset, run `EXPLAIN (ANALYZE, BUFFERS)` on the list and search queries for orders, stock movements, items and customers. Report any `Seq Scan` on a table with more than a few thousand rows, and report measured times against the targets table. Fix and re-measure rather than only reporting.

---

# DSA COMPLIANCE — the chosen data structures and algorithms are mandatory

Every technique in the Performance section is a requirement, not a suggestion. In your final report, fill in this table with the actual file, function and proof for each row. `/review` checks the code against it.

| # | Technique | Where it must appear | Cost | Proof you must show |
|---|---|---|---|---|
| 1 | B-tree indexes | `schema.prisma`; every hot list and join query | O(log n) lookup instead of O(n) scan | `EXPLAIN` shows an Index Scan; no Seq Scan on a table with more than a few thousand rows |
| 2 | Trigram GIN index | item and customer `?q=` search | sublinear substring match | `EXPLAIN` on a search shows the trigram index in use |
| 3 | Keyset (cursor) pagination | lists of orders, stock movements, dispatch history | O(log n + limit) at any depth | A test fetches a deep page with no `skip` beyond the cursor's `skip: 1`, and there is no `COUNT(*)` on these tables |
| 4 | Offset pagination with a cap | lists of items, customers | bounded by `limit` | The Zod pagination schema caps `limit` at 100 |
| 5 | Hash lookups (`Map` / `Set`) | `order.service` (line validation, duplicate-item detection), `dispatch.service` (stock check) | O(1) per lookup, O(n) in total | No `.find`, `.includes`, `.indexOf` or `.some` inside a loop over request lines |
| 6 | Transition table | `order.service.updateOrderStatus` | O(1) | One lookup in `ORDER_STATUS_TRANSITIONS`; no if/else chain over statuses |
| 7 | Ordered lock acquisition | `dispatch.service.dispatchOrder` | O(k log k) sort, removes deadlock risk | A test runs two concurrent dispatches whose items overlap in opposite order; both finish |
| 8 | Atomic conditional update | `stock.service.applyStockChange(tx, ...)`, the **single function** every stock write goes through | O(1), one round trip | grep shows `stock.update` / `stock.updateMany` appear only inside that function |
| 9 | Batch writes | order creation (nested `create` of lines); dispatch (`stockMovement.createMany`) | 1 round trip instead of k | No `create` inside a loop |
| 10 | Batched reads | one `findMany({ where: { id: { in: ids } } })`, then build a `Map` | 1 query instead of k | No Prisma call inside `.map()` or a loop that reads data |
| 11 | Parallel independent queries | the count and the page of offset-paginated lists (`Promise.all`) | latency is the max, not the sum | Visible in `item.service` and `customer.service` |
| 12 | Projection | every query | smaller rows, less serialization | `select` is used everywhere; `passwordHash` and `refreshTokenHash` never leave the service layer |

**Interpretation rules**

- Here `k` is the number of lines in an order, normally under 50. Sorting `k` items is negligible; the point of rule 7 is correctness, not speed.
- A sequential `for...of` loop with `await` is **required** for the ordered stock updates inside the dispatch transaction. That is deliberate lock ordering, not an N+1 read. Reads must still be batched (rule 10).
- Filtering, sorting and pagination happen in SQL, never in JavaScript. Never load a whole table into memory and filter it there.
- A cache, `Redis`, or any structure not listed above is out of scope for this task.
- If you believe a rule above is wrong for a specific case, stop and tell me why instead of working around it.

---

# `app.js` AND `index.js` SHAPE

`app.js` does configuration and route mounting only — no business logic, no route handlers defined inline:
```js
import express from "express";
import cors from "cors";
import cookieParser from "cookie-parser";
import helmet from "helmet";
import compression from "compression";

const app = express();

app.use(helmet());
app.use(compression());
app.use(cors({ origin: process.env.CORS_ORIGIN, credentials: true }));
app.use(express.json({ limit: "16kb" }));
app.use(express.urlencoded({ extended: true, limit: "16kb" }));
app.use(cookieParser());

import authRouter from "./routes/auth.routes.js";
import customerRouter from "./routes/customer.routes.js";
import itemRouter from "./routes/item.routes.js";
import stockRouter from "./routes/stock.routes.js";
import orderRouter from "./routes/order.routes.js";
import dispatchRouter from "./routes/dispatch.routes.js";

app.use("/api/v1/auth", authRouter);
app.use("/api/v1/customers", customerRouter);
app.use("/api/v1/items", itemRouter);
app.use("/api/v1/stock", stockRouter);
app.use("/api/v1/orders", orderRouter);
app.use("/api/v1/dispatch", dispatchRouter);

import { errorHandler } from "./middlewares/error.middleware.js";
app.use(errorHandler); // must be last

export { app };
```

`index.js` loads dotenv, calls `connectDB()` from `db/index.js`, and calls `app.listen()` **only inside the resolved promise** — the server must never accept traffic before the database connection is confirmed. Log and `process.exit(1)` on connection failure. Add graceful shutdown on `SIGTERM`/`SIGINT` that disconnects Prisma before exiting.

`db/index.js` exports a single Prisma client instance — a true singleton, since multiple clients exhaust the connection pool — plus a `connectDB()` that wraps `prisma.$connect()`.

---

# PACKAGES TO INSTALL

Nothing is installed yet. Run these from the repo root, one workspace at a time, so npm records each dependency in the right `package.json`.

**Production dependencies**

```bash
npm install --workspace apps/api express@4 @prisma/client@5 dotenv bcryptjs jsonwebtoken cookie-parser cors helmet compression express-rate-limit multer@2 cloudinary uuid
npm install --workspace apps/api --save-dev prisma@5 nodemon
npm install --workspace packages/shared zod
```

| Package | Why it's here |
|---|---|
| `bcryptjs` | Password hashing. Pure JS, so no native build toolchain needed — avoids the `node-gyp` failures `bcrypt` causes on Windows and in slim Docker images. If you'd rather have the native speed, `bcrypt` works identically at the call site; just be aware of the build requirement. |
| `jsonwebtoken` | Signing and verifying the access + refresh tokens. |
| `cookie-parser` | Reads the httpOnly `accessToken` / `refreshToken` cookies in `verifyJWT`. |
| `cors` | Locked to `CORS_ORIGIN` with `credentials: true` so cookies survive cross-origin calls from the web app. |
| `helmet` | Baseline security headers. |
| `express-rate-limit` | Throttles `POST /auth/login` against credential stuffing. |
| `multer` | **Install 2.x, not 1.x.** Multer `<2.1.0` carries two high-severity DoS CVEs (CVE-2026-3304, CVE-2026-2359). If any tutorial or reference tells you to use `multer@^1.4.5-lts.1`, ignore it. |
| `cloudinary` | Item image hosting only. |
| `zod` | Request validation. Installed in `packages/shared` only (see the scaffold section); `apps/api` consumes the schemas and never imports `zod` itself. |
| `express`, `@prisma/client`, `prisma`, `dotenv`, `nodemon` | The core stack. Express 4 and Prisma 5 are pinned on purpose (see version constraints). `nodemon` and `prisma` are dev dependencies. |
| `uuid` | Collision-safe filenames for staged uploads. |
| `compression` | gzip for JSON responses, so payloads are smaller on slow mobile networks. |

**Dev dependencies**

```bash
npm install --workspace apps/api --save-dev vitest supertest
```

| Package | Why it's here |
|---|---|
| `vitest` | Test runner. Native ESM support, so it works with `"type": "module"` without transpiler config. |
| `supertest` | HTTP assertions against the Express app. Needed for the dispatch transaction tests specifically. |

**Testing scope.** Write automated tests only for the dispatch transaction (success, wrong status, insufficient stock, concurrent dispatch of the same stock, and two concurrent dispatches whose items overlap in opposite order, which must both finish without a deadlock) and the stock ledger (adjustment writes a movement and updates quantity together, negative stock is rejected). Everything else is checked manually through a Postman collection, which you should also produce at `apps/api/postman/bizflow.postman_collection.json` covering every endpoint.

**Optional, only if you want them — ask me before adding**

- `pino` + `pino-http` — structured request logging, if you'd rather have that than the bare minimum.
- `eslint` + `prettier` — worth it, but `packages/config` in this monorepo is meant to hold shared lint config, so set it up there rather than per-app.

**Version constraints to respect**

- **Stay on Express 4.** Express 5 changed error-handling and routing behaviour; the `asyncHandler` pattern and the error middleware below assume 4.x. Don't upgrade as a side effect of this task.
- **Don't change the Prisma major version in this task.** Install Prisma 5.x. Prisma is on 7.x now, and moving up is worth doing later, but it brings breaking changes (v7 made the client Rust-free by default) and deserves its own isolated change once the backend works. Don't install a newer major version in this task; flag it to me as a follow-up instead.
- **One copy of `zod`, in `packages/shared`.** Two versions of a validation library in one tree produce schema instances that fail each other's `instanceof` checks, and the failure mode is baffling.

After installing, run `npm audit` and report anything high or critical rather than silently ignoring it.

---

# ALSO UPDATE

- **`.env.example`** — `DATABASE_URL`, `DIRECT_URL`, `SEED_ADMIN_NAME`, `SEED_ADMIN_EMAIL`, `SEED_ADMIN_PASSWORD`, `PORT`, `NODE_ENV`, `CORS_ORIGIN`, `ACCESS_TOKEN_SECRET`, `ACCESS_TOKEN_EXPIRY`, `REFRESH_TOKEN_SECRET`, `REFRESH_TOKEN_EXPIRY`, `CLOUDINARY_CLOUD_NAME`, `CLOUDINARY_API_KEY`, `CLOUDINARY_API_SECRET`
- **`apps/api/package.json`** — `"type": "module"`, scripts `dev`, `start`, `test` (vitest), `db:migrate`, `db:seed`, and the `prisma.seed` entry.
- **`packages/shared`** — finish it: a client function for every endpoint in the endpoint list, a Zod schema for every request body and query, and the shared constants. The web and mobile apps will consume these; don't leave endpoints unrepresented.
- **`apps/api/README.md`** — endpoint table, setup steps, env var list.

---

# WHAT NOT TO DO

- Don't switch ORMs or databases. Prisma + PostgreSQL stays.
- Don't put business logic in controllers, or HTTP awareness in services.
- Don't write directly to `Stock.quantity` outside a ledger transaction.
- Don't hard-delete business records.
- Don't return unpaginated lists.
- Don't define validation schemas or the status-transition map inside `apps/api`; they live in `packages/shared`.
- Don't touch `apps/web`, `apps/desktop`, or `apps/mobile`.
- Don't change the Prisma schema from the one in this prompt without asking me.
- Don't overwrite `CLAUDE.md`, `.gitignore` or anything in `docs/`.
- Don't run `prisma migrate reset` or any destructive database command, and don't point anything at a production database.
- Don't add Redis or any caching layer in this task.
- Don't call Cloudinary or any other network service inside a `$transaction`.
- Don't lower the bcrypt cost factor to make login faster.
- Don't run `COUNT(*)` on orders, stock movements or dispatch history.

---

# WORKFLOW — gstack skills

gstack is already installed globally on this machine (`~/.claude/skills/gstack`). Use its skills at the points below. If a skill is unavailable or errors, say so in one line and carry on; **don't install, update or reconfigure gstack**, don't run `gstack-team-init`, and don't add gstack files to this repo.

| When | Skill | What to do |
|---|---|---|
| Before step 1 | `/plan-eng-review` | Review this document. Report contradictions, gaps and risky assumptions as a short list. **Do not edit `docs/backend-spec.md` yourself**; I decide what changes. |
| Whole task | `/careful` | Keep it on. It must also apply before every migration command. |
| After steps 2, 6, 9, 10, 11 | `/review` | Review the branch diff. Fix real bugs. Ask me about disagreements over design. The DSA COMPLIANCE table is part of the review criteria. |
| After steps 6 and 11 | `/qa` | There is no browser here: run the API locally against the development database, exercise the endpoints with the seeded admin, and write a regression test for every bug found. Never use anything but the development database. Skip live Cloudinary calls unless credentials are in `.env`. |
| Before the final report | `/cso` | Expected to be unavailable here (no C++ build tools). If it is, write "not assessed" and do the manual checklist instead: cookie flags, rate limit on login, helmet, no secrets in git (`git grep`), no `passwordHash` in any response. |
| A failure you can't explain after two attempts | `/investigate` | Find the root cause before trying another fix. |
| Final | `/ship` | **Do not run it.** There is no GitHub remote yet. I'll ask for it later. Finish with commits on `feat/api-backend`. |

**Do not use:** `/freeze` (it would block `packages/shared` and the root files), `/office-hours`, `/plan-ceo-review`, `/autoplan`, the design and devex skills, `/benchmark`, `/land-and-deploy`, `/canary`. The product and architecture are already decided. Don't set up Codex or any outside review.

---

# EXECUTION ORDER

Work in this sequence and commit after each step on the `feat/api-backend` branch. Steps marked **gate** end with the gstack skill named in the WORKFLOW section.

0. **Gate:** run `/plan-eng-review` on this document and report the findings. Wait for my answer before continuing if it finds a contradiction.
1. **Workspace scaffold.** Root `package.json` and `README.md`, `packages/shared` skeleton with `constants` and `validation` filled in, `apps/api/package.json`, `.env.example`. Install dependencies from the repo root.
2. **Database. Gate: `/review`.** Write `schema.prisma`, run the `init` and `extras` migrations, write and run `seed.js`. Needs `apps/api/.env`; if it's missing, stop and ask me (see BEFORE YOU WRITE ANY CODE).
3. Utils (`ApiError`, `ApiResponse`, `asyncHandler`, `cloudinary`) and `db/index.js`
4. All five middlewares
5. `app.js`, `index.js`, `constants.js` — confirm the server boots and connects to the database
6. **Auth module. Gate: `/review`, then `/qa`.** Everything else depends on `verifyJWT`; test login with the seeded admin.
7. Items module + Cloudinary image flow
8. Customers module
9. **Stock module. Gate: `/review`.** The ledger transaction and `applyStockChange`.
10. **Orders module. Gate: `/review`.** Creation transaction and status transition validation.
11. **Dispatch module. Gate: `/review`, then `/qa`.** The atomic transaction, the most critical step.
12. Finish `packages/shared` (client function per endpoint), the Postman collection, the Vitest tests, and `apps/api/README.md`
13. Performance pass: `seed.perf.js`, the `EXPLAIN ANALYZE` review, and the completed DSA COMPLIANCE table
14. `/cso` or the manual security checklist, then the final report

# VERIFICATION BEFORE YOU REPORT DONE

- `node --check` passes on every `.js` file
- `npx prisma validate` passes, and `npx prisma migrate status` reports the database is up to date
- `npx prisma db seed` creates the admin, and logging in as that admin works and returns the standard response shape
- `npm run dev` boots cleanly and logs a successful database connection
- A protected route rejects an unauthenticated request with 401 and the standard error shape
- Grep the codebase for `console.log`, hardcoded secrets, and `res.json({ error` — all should return nothing
- Report `EXPLAIN (ANALYZE, BUFFERS)` results for the list and search queries against the seeded dataset, with measured times against the response-time targets
- Show the completed DSA COMPLIANCE table with the file, function and proof for all 12 rows, and confirm no network calls inside any `$transaction`
- Show the gstack skills you ran, one line each with the outcome (or "unavailable")
- Show me the final `apps/api/src` file tree and the endpoint table

Then walk me through the dispatch transaction line by line so I can verify the atomicity myself. If any requirement above conflicts with another requirement, or with how a library actually behaves, stop and ask rather than guessing.

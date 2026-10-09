# DocsApp — A to Z Learning Plan

> Build a "My Documents" app (signup/login, upload to S3, search documents) with
> **React + Express + PostgreSQL + AWS S3**, then **Dockerize**, **test**, add **Swagger**,
> set up **CI/CD with GitHub Actions** and **deploy on AWS** — all on free tiers.

---

## Table of Contents

- [How to use this plan](#how-to-use-this-plan)
- [Progress checklist](#progress-checklist)
- [Final architecture](#final-architecture)
- [Cost rules (stay free)](#cost-rules-stay-free)
- [Phase 0 — Tools & accounts](#phase-0--tools--accounts)
- [Phase 1 — Project skeleton](#phase-1--project-skeleton)
- [Phase 2 — Docker for local development](#phase-2--docker-for-local-development)
- [Phase 3 — PostgreSQL & migrations](#phase-3--postgresql--migrations)
- [Phase 4 — Auth API + Swagger](#phase-4--auth-api--swagger)
- [Phase 5 — S3 setup & presigned uploads](#phase-5--s3-setup--presigned-uploads)
- [Phase 6 — Documents API (list, search, download, delete)](#phase-6--documents-api-list-search-download-delete)
- [Phase 7 — React frontend](#phase-7--react-frontend)
- [Phase 8 — Testing & code quality](#phase-8--testing--code-quality)
- [Phase 9 — Production Docker image](#phase-9--production-docker-image)
- [Phase 10 — CI with GitHub Actions](#phase-10--ci-with-github-actions)
- [Phase 11 — AWS infrastructure (manual, by hand)](#phase-11--aws-infrastructure-manual-by-hand)
- [Phase 12 — CD: automatic deployment](#phase-12--cd-automatic-deployment)
- [Phase 13 — Monitoring & operations](#phase-13--monitoring--operations)
- [Phase 14 — Bonus / level up](#phase-14--bonus--level-up)
- [Teardown checklist](#teardown-checklist)
- [Troubleshooting](#troubleshooting)
- [Glossary](#glossary)

---

## How to use this plan

1. Do **one phase at a time**. Every phase ends with something that works.
2. **Type the code yourself** — don't copy-paste blindly. Read every line and the comments.
3. After each phase: `git add . && git commit -m "phase X: ..." && git push`.
4. From Phase 10 onwards, work on a **branch per phase** and open a Pull Request so CI runs.
5. Region used in examples: `ap-south-1` (Mumbai). Pick the region nearest to you and use
   **the same region everywhere**.
6. Replace `yourname` in bucket names with something unique (bucket names are global).

Estimated time: **~4 weeks at 2–3 hours/day**.

---

## Progress checklist

- [ ] Phase 0 — Tools & accounts
- [ ] Phase 1 — Project skeleton
- [ ] Phase 2 — Docker for local development
- [ ] Phase 3 — PostgreSQL & migrations
- [ ] Phase 4 — Auth API + Swagger
- [ ] Phase 5 — S3 setup & presigned uploads
- [ ] Phase 6 — Documents API
- [ ] Phase 7 — React frontend
- [ ] Phase 8 — Testing & code quality
- [ ] Phase 9 — Production Docker image
- [ ] Phase 10 — CI with GitHub Actions
- [ ] Phase 11 — AWS infrastructure
- [ ] Phase 12 — CD: automatic deployment
- [ ] Phase 13 — Monitoring & operations
- [ ] Phase 14 — Bonus

---

## Final architecture

```
                        ┌───────────────────────── AWS ───────────────────────────┐
 Browser                │                                                          │
 https://xxxx.cloudfront.net                                                       │
     │                  │   ┌─────────────┐   /*     ┌───────────────────────┐    │
     └─────────────────────►│ CloudFront  │─────────►│ S3: frontend bucket   │    │
                        │   │ (free HTTPS)│          │ (React build output)  │    │
                        │   └──────┬──────┘          └───────────────────────┘    │
                        │          │ /api/*                                        │
                        │          ▼                                               │
                        │   ┌──────────────────┐  SQL  ┌──────────────────────┐   │
                        │   │ EC2 t3.micro     │──────►│ RDS PostgreSQL       │   │
                        │   │ Docker: Express  │       │ db.t4g.micro         │   │
                        │   └────────┬─────────┘       └──────────────────────┘   │
                        │            │ IAM role (no access keys)                   │
                        │            ▼                                             │
                        │   ┌───────────────────────┐  ┌───────────────────────┐  │
                        │   │ S3: documents bucket  │  │ ECR: Docker images    │  │
                        │   └───────────────────────┘  └───────────────────────┘  │
                        └──────────────────────────────────────────────────────────┘
                                         ▲
 GitHub ── push to main ── GitHub Actions: test → build image → push to ECR
                                          → deploy to EC2 (SSM) → upload frontend to S3
```

### Upload flow (the most important idea)

```
1. Browser  ── POST /api/documents/upload-url {fileName, size, type} ──► API
2. API      ── checks JWT, creates S3 key, returns presigned PUT URL (valid 5 min) ──► Browser
3. Browser  ── PUT file bytes directly ──► S3   (file never passes through your server)
4. Browser  ── POST /api/documents/confirm {key} ──► API
5. API      ── verifies object exists in S3, INSERTs metadata row ──► PostgreSQL
```

**Files live in S3. Information about files (name, size, owner, key) lives in PostgreSQL.**

### Security rule #1

Every documents query **must** contain `WHERE user_id = $1`, and `userId` always comes
from the **verified JWT**, never from the request body/query. That is what makes
"users see only their own documents" true.

---

## Cost rules (stay free)

New AWS accounts get the **Free plan with credits** (for the first months). This project fits in it.

| Service | Used for | Cost |
|---|---|---|
| EC2 t3.micro | Backend container | Free tier / credits |
| Public IPv4 (Elastic IP) | Stable address for EC2 | ~$3.6/month → covered by credits |
| RDS db.t4g.micro, 20 GB gp3, Single-AZ | Database | Free tier / credits |
| S3 (2 buckets) | Documents + frontend | Cents at most |
| CloudFront | HTTPS + CDN | Free tier (1 TB/month) |
| ECR | Docker images | Free under 500 MB (keep last 5 images) |
| SSM Parameter Store (standard) | Secrets | Free |
| CloudWatch Logs | Logs | Free tier (5 GB) |
| GitHub + Actions | Code + CI/CD | Free |

### ⚠️ NEVER create these (they burn money fast)

- **NAT Gateway** (~$32/month)
- **Application/Network Load Balancer** (~$16+/month)
- **ECS Fargate**, **EKS**
- RDS **Multi-AZ**, RDS Proxy, Aurora
- Any instance bigger than `micro`

### ✅ Do on day 1

- Budget alerts at **$1** and **$5**
- MFA on root user
- Stop/delete things you're not using (see [Teardown checklist](#teardown-checklist))

---

## Phase 0 — Tools & accounts

**Goal:** All tools installed, accounts created and secured.

### 0.1 Install on Windows

| Tool | Where | Check |
|---|---|---|
| Node.js 24 LTS | nodejs.org | `node -v` |
| Git | git-scm.com | `git --version` |
| Docker Desktop | docker.com (free for personal use; enables WSL2) | `docker -v` |
| AWS CLI v2 | aws.amazon.com/cli | `aws --version` |
| VS Code | code.visualstudio.com | — |
| pgAdmin (optional GUI) | pgadmin.org | — |

VS Code extensions: **ESLint**, **Prettier**, **Docker**, **GitHub Actions**, **YAML**.

> ⚠️ Do **not** install PostgreSQL itself on Windows — it will run in Docker.
> If you already installed it, stop its Windows service, otherwise port 5432 will conflict.

### 0.2 GitHub

1. Create an account on github.com.
2. Configure Git once:
   ```powershell
   git config --global user.name "Your Name"
   git config --global user.email "you@example.com"
   git config --global init.defaultBranch main
   ```

### 0.3 AWS account (security first)

1. Sign up at aws.amazon.com (needs a card; choose the **Free plan**).
2. **Root user → MFA**: Account menu → Security credentials → Assign MFA device.
3. **Billing → Budgets → Create budget** → "Zero spend" or monthly cost budget of **$1**, and another of **$5**, with email alerts.
4. **IAM → Users → Create user** `admin-yourname`
   - Attach policy `AdministratorAccess`
   - Enable console access + MFA
   - From now on **log in with this user, not root**.
5. Create an **access key** for `admin-yourname` (Security credentials → Create access key → CLI).
6. On your laptop:
   ```powershell
   aws configure
   # AWS Access Key ID:     <paste>
   # AWS Secret Access Key: <paste>
   # Default region name:   ap-south-1
   # Default output format: json
   aws sts get-caller-identity   # should print your account + user
   ```

### ✅ Done when
- `node -v`, `git --version`, `docker -v`, `aws --version` all work
- `aws sts get-caller-identity` shows your admin user
- Budget alerts exist

### 📚 You learned
Root vs IAM user, MFA, access keys, budgets.

---

## Phase 1 — Project skeleton

**Goal:** Monorepo with `client/` (React) and `server/` (Express), pushed to GitHub.

### 1.1 Create the repo

```powershell
cd C:\Users\paras\Desktop\postgresql
git init
```

Root **`.gitignore`**:
```gitignore
node_modules
.env
dist
coverage
*.log
.DS_Store
```

### 1.2 Frontend (React + Vite)

```powershell
npm create vite@latest client -- --template react
cd client
npm install
npm install react-router-dom
cd ..
```

Edit **`client/vite.config.js`** — add a dev proxy so the frontend can call `/api/...` without CORS:
```js
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  server: {
    proxy: {
      "/api": "http://localhost:5000", // forward /api/* to Express in development
    },
  },
});
```

> Why? In production CloudFront serves the frontend **and** `/api` on the same domain.
> The Vite proxy makes development behave the same way.

### 1.3 Backend (Express)

```powershell
mkdir server
cd server
npm init -y
npm install express pg dotenv bcryptjs jsonwebtoken zod pino pino-http swagger-jsdoc swagger-ui-express node-pg-migrate @aws-sdk/client-s3 @aws-sdk/s3-request-presigner
npm install -D nodemon pino-pretty vitest supertest eslint @eslint/js globals
mkdir src, src\routes, src\middleware, migrations, tests
cd ..
```

> We use **`bcryptjs`** (pure JavaScript) instead of `bcrypt` (native C++),
> so the Docker image builds on Alpine Linux without compilers.

| Package | Purpose |
|---|---|
| express | Web server and routes (v5 — async errors handled automatically) |
| pg | PostgreSQL client |
| dotenv | Load `.env` into `process.env` |
| bcryptjs | Hash passwords |
| jsonwebtoken | Create/verify JWT login tokens |
| zod | Validate request bodies |
| pino / pino-http | Structured JSON logging |
| swagger-jsdoc / swagger-ui-express | API docs at `/api/docs` |
| node-pg-migrate | Database migrations |
| @aws-sdk/client-s3, s3-request-presigner | Talk to S3, create presigned URLs |
| nodemon | Auto-restart in dev |
| vitest / supertest | Tests |
| eslint | Linting |

Edit **`server/package.json`** — add `"type": "module"` and scripts:
```json
{
  "name": "server",
  "version": "1.0.0",
  "type": "module",
  "scripts": {
    "dev": "nodemon src/index.js",
    "start": "node src/index.js",
    "migrate": "node-pg-migrate -m migrations",
    "test": "vitest run",
    "lint": "eslint ."
  }
}
```
(keep the `dependencies` / `devDependencies` npm added)

**`server/src/app.js`** — the Express app (no `listen` here, so tests can import it):
```js
import express from "express";

export const app = express();

app.use(express.json());

app.get("/api/health", (req, res) => {
  res.json({ status: "ok" });
});
```

**`server/src/index.js`** — starts the server:
```js
import "dotenv/config";
import { app } from "./app.js";

const PORT = process.env.PORT || 5000;
app.listen(PORT, () => console.log(`API running on http://localhost:${PORT}`));
```

**`server/.env`**:
```env
PORT=5000
```

Also create **`server/.env.example`** (this one IS committed — it documents required variables, no real secrets):
```env
PORT=5000
DATABASE_URL=postgres://postgres:postgres@localhost:5432/docsapp
JWT_SECRET=change-me
AWS_REGION=ap-south-1
S3_BUCKET=docsapp-dev-yourname
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
```

### 1.4 Run both

Terminal 1:
```powershell
cd server
npm run dev
```
Terminal 2:
```powershell
cd client
npm run dev
```

- http://localhost:5000/api/health → `{"status":"ok"}`
- http://localhost:5173/api/health → same response (through the Vite proxy ✅)

### 1.5 Push to GitHub

1. github.com → New repository → `docsapp` (no README, no .gitignore).
2.
   ```powershell
   git add .
   git commit -m "phase 1: project skeleton"
   git remote add origin https://github.com/<you>/docsapp.git
   git push -u origin main
   ```

### ✅ Done when
Both apps run, proxy works, code is on GitHub.

### 📚 You learned
Monorepo, Vite, ES modules, dev proxy, `.env` vs `.env.example`.

---

## Phase 2 — Docker for local development

**Goal:** PostgreSQL runs in Docker with one command.

### 2.1 Concepts (read first)

| Term | Meaning |
|---|---|
| Image | A template (e.g. `postgres:17`) |
| Container | A running instance of an image |
| Volume | Storage that survives container deletion |
| Port mapping `5432:5432` | `host:container` |
| Docker Compose | A YAML file describing multiple containers |

### 2.2 `docker-compose.yml` (in the repo root)

```yaml
services:
  db:
    image: postgres:17
    container_name: docsapp-db
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: docsapp
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
      - ./docker/init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      retries: 10

volumes:
  pgdata:
```

**`docker/init.sql`** — runs only the first time the volume is created; makes a separate DB for tests:
```sql
CREATE DATABASE docsapp_test;
```

### 2.3 Run it

```powershell
docker compose up -d          # start in background
docker ps                     # see running containers
docker logs docsapp-db        # see logs
docker exec -it docsapp-db psql -U postgres -d docsapp   # open SQL shell
```

Inside `psql`:
```sql
\l          -- list databases (you should see docsapp and docsapp_test)
\q          -- quit
```

Other useful commands:
```powershell
docker compose stop           # stop (data kept)
docker compose down           # remove containers (data kept in volume)
docker compose down -v        # remove containers AND data (fresh start)
```

### 2.4 Connect pgAdmin (optional)
Host `localhost`, port `5432`, user `postgres`, password `postgres`.

### ✅ Done when
`docker exec -it docsapp-db psql -U postgres -d docsapp` opens a SQL prompt.

### 📚 You learned
Images, containers, volumes, ports, compose, init scripts, healthchecks.

---

## Phase 3 — PostgreSQL & migrations

**Goal:** Learn SQL by hand, then create the real tables with migrations.

### 3.1 Practice SQL by hand (playground)

Open `psql` and type these one by one. Understand each result.

```sql
-- create a practice table
CREATE TABLE practice_users (
  id         SERIAL PRIMARY KEY,
  email      TEXT UNIQUE NOT NULL,
  created_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE practice_docs (
  id        SERIAL PRIMARY KEY,
  user_id   INT NOT NULL REFERENCES practice_users(id) ON DELETE CASCADE,
  file_name TEXT NOT NULL
);

INSERT INTO practice_users (email) VALUES ('a@test.com'), ('b@test.com');
INSERT INTO practice_docs (user_id, file_name) VALUES
  (1, 'Report 2026.pdf'), (1, 'photo.png'), (2, 'report-final.docx');

SELECT * FROM practice_users;
SELECT * FROM practice_docs WHERE user_id = 1;
SELECT * FROM practice_docs WHERE file_name ILIKE '%report%';      -- case-insensitive search
SELECT u.email, d.file_name FROM practice_docs d JOIN practice_users u ON u.id = d.user_id;
SELECT user_id, COUNT(*) FROM practice_docs GROUP BY user_id;
UPDATE practice_docs SET file_name = 'pic.png' WHERE id = 2;
INSERT INTO practice_users (email) VALUES ('a@test.com');           -- error! UNIQUE
INSERT INTO practice_docs (user_id, file_name) VALUES (99, 'x');    -- error! FOREIGN KEY
DELETE FROM practice_users WHERE id = 1;                            -- CASCADE deletes their docs
SELECT * FROM practice_docs;
EXPLAIN ANALYZE SELECT * FROM practice_docs WHERE user_id = 2;      -- see the query plan

DROP TABLE practice_docs, practice_users;
```

Concepts to understand: `PRIMARY KEY`, `SERIAL`, `UNIQUE`, `NOT NULL`, `REFERENCES` (foreign key),
`ON DELETE CASCADE`, `TIMESTAMPTZ`, `ILIKE`, `JOIN`, `GROUP BY`, `EXPLAIN`.

### 3.2 Why migrations?

You never run `CREATE TABLE` by hand in production. A **migration** is a versioned file that
changes the schema. The tool remembers which ones already ran (in a `pgmigrations` table),
so every environment (laptop, CI, production) gets the exact same schema.

### 3.3 Add the DB URL to `server/.env`

```env
PORT=5000
DATABASE_URL=postgres://postgres:postgres@localhost:5432/docsapp
```

### 3.4 Create migrations

```powershell
cd server
npx node-pg-migrate create create-users -m migrations
npx node-pg-migrate create create-documents -m migrations
```

Edit the generated **`migrations/<timestamp>_create-users.js`**:
```js
export const up = (pgm) => {
  pgm.sql(`
    CREATE TABLE users (
      id            SERIAL PRIMARY KEY,
      email         TEXT UNIQUE NOT NULL,
      password_hash TEXT NOT NULL,
      created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
    );
  `);
};

export const down = (pgm) => {
  pgm.sql(`DROP TABLE users;`);
};
```

Edit **`migrations/<timestamp>_create-documents.js`**:
```js
export const up = (pgm) => {
  pgm.sql(`
    CREATE TABLE documents (
      id          SERIAL PRIMARY KEY,
      user_id     INT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
      file_name   TEXT NOT NULL,
      s3_key      TEXT NOT NULL UNIQUE,
      size_bytes  BIGINT NOT NULL,
      mime_type   TEXT NOT NULL,
      uploaded_at TIMESTAMPTZ NOT NULL DEFAULT now()
    );
    CREATE INDEX idx_documents_user_uploaded ON documents (user_id, uploaded_at DESC);
  `);
};

export const down = (pgm) => {
  pgm.sql(`DROP TABLE documents;`);
};
```

> We write raw SQL inside `pgm.sql()` on purpose — you keep practicing real SQL.

Run:
```powershell
npm run migrate up      # applies both
npm run migrate down    # rolls back the last one
npm run migrate up      # apply again
```

Check in psql: `\dt` (list tables), `\d documents` (describe table), `SELECT * FROM pgmigrations;`

### 3.5 Database connection — `server/src/db.js`

```js
import pg from "pg";

// A Pool keeps a few connections open and reuses them (much faster than connecting per request)
export const pool = new pg.Pool({
  connectionString: process.env.DATABASE_URL,
  max: 10,
});
```

Test it — add to **`src/app.js`**:
```js
import { pool } from "./db.js";

app.get("/api/health", async (req, res) => {
  await pool.query("SELECT 1"); // fails if DB is down
  res.json({ status: "ok" });
});
```

### ✅ Done when
`npm run migrate up` works and `/api/health` returns ok while Docker DB is running
(and fails if you `docker compose stop`).

### 📚 You learned
DDL, constraints, indexes, migrations up/down, connection pooling.

---

## Phase 4 — Auth API + Swagger

**Goal:** Signup, login, JWT-protected routes, documented in Swagger.

### 4.1 Concepts

- **Never store plain passwords.** Store `bcrypt` hash. On login, compare.
- **JWT** = signed token `{ userId }`. Server signs it with `JWT_SECRET`; client sends it as
  `Authorization: Bearer <token>`; server verifies signature on every request.
- **Status codes:** `201` created, `400` bad input, `401` not logged in / wrong password,
  `404` not found, `409` conflict (email exists), `500` server error.

Add to `server/.env`:
```env
JWT_SECRET=dev-secret-change-me
```

### 4.2 Logging + error handling — `server/src/app.js` (full version)

```js
import express from "express";
import pinoHttp from "pino-http";
import swaggerUi from "swagger-ui-express";
import { ZodError } from "zod";
import { pool } from "./db.js";
import { swaggerSpec } from "./swagger.js";
import authRoutes from "./routes/auth.js";
import documentRoutes from "./routes/documents.js";

export const app = express();

app.use(express.json());
if (process.env.NODE_ENV !== "test") app.use(pinoHttp());

app.get("/api/health", async (req, res) => {
  await pool.query("SELECT 1");
  res.json({ status: "ok" });
});

app.use("/api/docs", swaggerUi.serve, swaggerUi.setup(swaggerSpec));
app.use("/api/auth", authRoutes);
app.use("/api/documents", documentRoutes);

// 404 for unknown API routes
app.use("/api", (req, res) => res.status(404).json({ error: "Not found" }));

// Central error handler (Express 5 forwards async errors here automatically)
app.use((err, req, res, next) => {
  if (err instanceof ZodError) {
    return res.status(400).json({ error: "Validation failed", details: err.issues });
  }
  req.log?.error(err);
  res.status(500).json({ error: "Internal server error" });
});
```

> Create an empty `src/routes/documents.js` for now so the import works:
> ```js
> import { Router } from "express";
> const router = Router();
> export default router;
> ```

For pretty logs in dev, change the dev script in `package.json`:
```json
"dev": "nodemon src/index.js | pino-pretty"
```

### 4.3 Swagger setup — `server/src/swagger.js`

```js
import swaggerJsdoc from "swagger-jsdoc";

export const swaggerSpec = swaggerJsdoc({
  definition: {
    openapi: "3.0.3",
    info: { title: "DocsApp API", version: "1.0.0" },
    components: {
      securitySchemes: {
        bearerAuth: { type: "http", scheme: "bearer", bearerFormat: "JWT" },
      },
    },
  },
  apis: ["./src/routes/*.js"], // reads @openapi comments from route files
});
```

### 4.4 Auth middleware — `server/src/middleware/auth.js`

```js
import jwt from "jsonwebtoken";

export function requireAuth(req, res, next) {
  const header = req.headers.authorization || "";
  const [scheme, token] = header.split(" ");
  if (scheme !== "Bearer" || !token) {
    return res.status(401).json({ error: "Missing token" });
  }
  try {
    const payload = jwt.verify(token, process.env.JWT_SECRET);
    req.userId = payload.userId; // trusted: comes from a verified signature
    next();
  } catch {
    res.status(401).json({ error: "Invalid or expired token" });
  }
}
```

### 4.5 Auth routes — `server/src/routes/auth.js`

```js
import { Router } from "express";
import bcrypt from "bcryptjs";
import jwt from "jsonwebtoken";
import { z } from "zod";
import { pool } from "../db.js";
import { requireAuth } from "../middleware/auth.js";

const router = Router();

const credentialsSchema = z.object({
  email: z.string().email().toLowerCase(),
  password: z.string().min(8, "Password must be at least 8 characters"),
});

/**
 * @openapi
 * /api/auth/signup:
 *   post:
 *     tags: [Auth]
 *     summary: Create an account
 *     requestBody:
 *       required: true
 *       content:
 *         application/json:
 *           schema:
 *             type: object
 *             required: [email, password]
 *             properties:
 *               email: { type: string, example: user@test.com }
 *               password: { type: string, example: password123 }
 *     responses:
 *       201: { description: User created }
 *       400: { description: Validation failed }
 *       409: { description: Email already registered }
 */
router.post("/signup", async (req, res) => {
  const { email, password } = credentialsSchema.parse(req.body);
  const passwordHash = await bcrypt.hash(password, 10);
  try {
    const { rows } = await pool.query(
      "INSERT INTO users (email, password_hash) VALUES ($1, $2) RETURNING id, email, created_at",
      [email, passwordHash]
    );
    res.status(201).json(rows[0]);
  } catch (err) {
    if (err.code === "23505") {
      // 23505 = PostgreSQL unique_violation
      return res.status(409).json({ error: "Email already registered" });
    }
    throw err;
  }
});

/**
 * @openapi
 * /api/auth/login:
 *   post:
 *     tags: [Auth]
 *     summary: Log in and receive a JWT
 *     requestBody:
 *       required: true
 *       content:
 *         application/json:
 *           schema:
 *             type: object
 *             properties:
 *               email: { type: string, example: user@test.com }
 *               password: { type: string, example: password123 }
 *     responses:
 *       200: { description: "{ token, user }" }
 *       401: { description: Wrong email or password }
 */
router.post("/login", async (req, res) => {
  const { email, password } = credentialsSchema.parse(req.body);
  const { rows } = await pool.query(
    "SELECT id, email, password_hash FROM users WHERE email = $1",
    [email]
  );
  const user = rows[0];
  // same error for "no user" and "wrong password" — don't reveal which emails exist
  if (!user || !(await bcrypt.compare(password, user.password_hash))) {
    return res.status(401).json({ error: "Wrong email or password" });
  }
  const token = jwt.sign({ userId: user.id }, process.env.JWT_SECRET, { expiresIn: "1d" });
  res.json({ token, user: { id: user.id, email: user.email } });
});

/**
 * @openapi
 * /api/auth/me:
 *   get:
 *     tags: [Auth]
 *     summary: Current user
 *     security: [{ bearerAuth: [] }]
 *     responses:
 *       200: { description: Current user }
 *       401: { description: Not logged in }
 */
router.get("/me", requireAuth, async (req, res) => {
  const { rows } = await pool.query(
    "SELECT id, email, created_at FROM users WHERE id = $1",
    [req.userId]
  );
  res.json(rows[0]);
});

export default router;
```

### 4.6 Try it

1. Open http://localhost:5000/api/docs
2. `POST /api/auth/signup` → 201
3. Same again → 409
4. `POST /api/auth/login` → copy `token`
5. Click **Authorize** (top right) → paste token
6. `GET /api/auth/me` → your user ✅
7. Check psql: `SELECT * FROM users;` — notice `password_hash` is not your password.

### ✅ Done when
Signup/login/me work from Swagger UI.

### 📚 You learned
Hashing, JWT, middleware, validation, error handler, SQL injection protection
(`$1` parameters — **never** build SQL with string concatenation), OpenAPI docs.

---

## Phase 5 — S3 setup & presigned uploads

**Goal:** Upload files from the browser directly to a private S3 bucket.

### 5.1 Concepts

| Term | Meaning |
|---|---|
| Bucket | A container for files (globally unique name) |
| Object | A file in a bucket |
| Key | The object's "path", e.g. `users/7/uuid-report.pdf` |
| Block Public Access | Keep ON — nobody can read files without permission |
| IAM policy | JSON saying who can do what on which resource |
| Presigned URL | Temporary URL signed with your credentials allowing ONE action on ONE key |
| CORS | Browser rule: S3 must allow your website's origin to PUT/GET |

### 5.2 Create the dev bucket (AWS Console)

1. **S3 → Create bucket**
   - Name: `docsapp-dev-yourname`
   - Region: `ap-south-1`
   - **Block all public access: ON** ✅
   - Versioning: off (for now), Encryption: SSE-S3 (default)
2. Bucket → **Permissions → CORS** → paste:
   ```json
   [
     {
       "AllowedOrigins": ["http://localhost:5173"],
       "AllowedMethods": ["PUT", "GET"],
       "AllowedHeaders": ["*"],
       "ExposeHeaders": ["ETag"],
       "MaxAgeSeconds": 3000
     }
   ]
   ```

### 5.3 Create a least-privilege IAM user for local dev

1. **IAM → Policies → Create policy → JSON**, name `docsapp-dev-s3`:
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Action": ["s3:PutObject", "s3:GetObject", "s3:DeleteObject"],
         "Resource": "arn:aws:s3:::docsapp-dev-yourname/*"
       }
     ]
   }
   ```
2. **IAM → Users → Create user** `docsapp-dev` (no console access) → attach `docsapp-dev-s3`.
3. Create an access key → put in `server/.env`:
   ```env
   AWS_REGION=ap-south-1
   S3_BUCKET=docsapp-dev-yourname
   AWS_ACCESS_KEY_ID=AKIA...
   AWS_SECRET_ACCESS_KEY=...
   ```

> The app uses this weak user, not your admin. If the key leaks, attackers can only touch one bucket.
> In production we will use **no keys at all** (EC2 IAM role).

### 5.4 S3 client — `server/src/s3.js`

```js
import { S3Client } from "@aws-sdk/client-s3";

// Credentials are found automatically:
//   local  -> AWS_ACCESS_KEY_ID / AWS_SECRET_ACCESS_KEY from .env
//   EC2    -> the instance IAM role
export const s3 = new S3Client({
  region: process.env.AWS_REGION,
  // Newer SDK versions add checksum params to presigned PUT URLs, which breaks
  // browser uploads. Only add checksums when S3 requires them.
  requestChecksumCalculation: "WHEN_REQUIRED",
});

export const BUCKET = process.env.S3_BUCKET;
```

### 5.5 Upload routes — `server/src/routes/documents.js` (part 1)

```js
import { Router } from "express";
import { randomUUID } from "node:crypto";
import { z } from "zod";
import {
  PutObjectCommand,
  HeadObjectCommand,
} from "@aws-sdk/client-s3";
import { getSignedUrl } from "@aws-sdk/s3-request-presigner";
import { pool } from "../db.js";
import { s3, BUCKET } from "../s3.js";
import { requireAuth } from "../middleware/auth.js";

const router = Router();
router.use(requireAuth); // every documents route needs login

const MAX_SIZE = 10 * 1024 * 1024; // 10 MB
const ALLOWED_TYPES = [
  "application/pdf",
  "image/png",
  "image/jpeg",
  "text/plain",
  "application/vnd.openxmlformats-officedocument.wordprocessingml.document",
];

const uploadUrlSchema = z.object({
  fileName: z.string().min(1).max(255),
  mimeType: z.enum(ALLOWED_TYPES),
  size: z.number().int().positive().max(MAX_SIZE),
});

/**
 * @openapi
 * /api/documents/upload-url:
 *   post:
 *     tags: [Documents]
 *     summary: Get a presigned URL to upload a file directly to S3
 *     security: [{ bearerAuth: [] }]
 *     requestBody:
 *       required: true
 *       content:
 *         application/json:
 *           schema:
 *             type: object
 *             properties:
 *               fileName: { type: string, example: report.pdf }
 *               mimeType: { type: string, example: application/pdf }
 *               size: { type: integer, example: 12345 }
 *     responses:
 *       200: { description: "{ uploadUrl, key }" }
 */
router.post("/upload-url", async (req, res) => {
  const { fileName, mimeType, size } = uploadUrlSchema.parse(req.body);
  const safeName = fileName.replace(/[^\w.\- ]/g, "_");
  const key = `users/${req.userId}/${randomUUID()}-${safeName}`;

  const command = new PutObjectCommand({
    Bucket: BUCKET,
    Key: key,
    ContentType: mimeType,
    ContentLength: size,
  });
  const uploadUrl = await getSignedUrl(s3, command, { expiresIn: 300 }); // 5 minutes
  res.json({ uploadUrl, key });
});

const confirmSchema = z.object({
  key: z.string(),
  fileName: z.string().min(1).max(255),
});

/**
 * @openapi
 * /api/documents/confirm:
 *   post:
 *     tags: [Documents]
 *     summary: Save document metadata after the file was uploaded to S3
 *     security: [{ bearerAuth: [] }]
 *     requestBody:
 *       required: true
 *       content:
 *         application/json:
 *           schema:
 *             type: object
 *             properties:
 *               key: { type: string }
 *               fileName: { type: string }
 *     responses:
 *       201: { description: Document saved }
 *       404: { description: File not found in S3 }
 */
router.post("/confirm", async (req, res) => {
  const { key, fileName } = confirmSchema.parse(req.body);

  // A user may only confirm keys inside their own folder
  if (!key.startsWith(`users/${req.userId}/`)) {
    return res.status(404).json({ error: "Not found" });
  }

  // Ask S3 for the real size/type — never trust the client
  let head;
  try {
    head = await s3.send(new HeadObjectCommand({ Bucket: BUCKET, Key: key }));
  } catch {
    return res.status(404).json({ error: "File not found in S3" });
  }

  const { rows } = await pool.query(
    `INSERT INTO documents (user_id, file_name, s3_key, size_bytes, mime_type)
     VALUES ($1, $2, $3, $4, $5)
     RETURNING id, file_name, size_bytes, mime_type, uploaded_at`,
    [req.userId, fileName, key, head.ContentLength, head.ContentType]
  );
  res.status(201).json(rows[0]);
});

export default router;
```

> **Note:** The API returns `404` (not `403`) when a user touches someone else's document.
> This (1) doesn't reveal that the document exists, and (2) matters for CloudFront later
> (Phase 11 maps `403` to the React app).

### 5.6 Try it (before the frontend exists)

1. Swagger: login → Authorize → `POST /api/documents/upload-url` with a real file's size
   and type → copy `uploadUrl` and `key`.
2. Upload with curl (PowerShell — use `curl.exe`, not the `curl` alias):
   ```powershell
   curl.exe -X PUT -H "Content-Type: application/pdf" --upload-file .\test.pdf "<uploadUrl>"
   ```
3. Check the object in the S3 console.
4. Swagger: `POST /api/documents/confirm` → 201. Check `SELECT * FROM documents;`

### ✅ Done when
A file lands in S3 under `users/<id>/...` and a row exists in `documents`.

### 📚 You learned
Buckets, keys, IAM least privilege, presigned URLs, CORS, why files bypass your server.

---

## Phase 6 — Documents API (list, search, download, delete)

**Goal:** Complete the documents API.

Add to **`server/src/routes/documents.js`** (above `export default router`), and update the import:
```js
import {
  PutObjectCommand,
  HeadObjectCommand,
  GetObjectCommand,
  DeleteObjectCommand,
} from "@aws-sdk/client-s3";
```

```js
const listSchema = z.object({
  search: z.string().max(100).optional().default(""),
  page: z.coerce.number().int().min(1).optional().default(1),
});

const PAGE_SIZE = 20;

/**
 * @openapi
 * /api/documents:
 *   get:
 *     tags: [Documents]
 *     summary: List my documents (search by file name)
 *     security: [{ bearerAuth: [] }]
 *     parameters:
 *       - { in: query, name: search, schema: { type: string } }
 *       - { in: query, name: page, schema: { type: integer, default: 1 } }
 *     responses:
 *       200: { description: "{ items, total, page, pageSize }" }
 */
router.get("/", async (req, res) => {
  const { search, page } = listSchema.parse(req.query);
  // escape LIKE wildcards so a search for "100%" works literally
  const pattern = `%${search.replace(/[\\%_]/g, "\\$&")}%`;
  const offset = (page - 1) * PAGE_SIZE;

  const { rows } = await pool.query(
    `SELECT id, file_name, size_bytes, mime_type, uploaded_at,
            COUNT(*) OVER() AS total
       FROM documents
      WHERE user_id = $1 AND file_name ILIKE $2
      ORDER BY uploaded_at DESC
      LIMIT $3 OFFSET $4`,
    [req.userId, pattern, PAGE_SIZE, offset]
  );

  res.json({
    items: rows.map(({ total, ...doc }) => doc),
    total: rows[0] ? Number(rows[0].total) : 0,
    page,
    pageSize: PAGE_SIZE,
  });
});

// helper: load a document only if it belongs to the current user
async function findOwnDocument(id, userId) {
  const { rows } = await pool.query(
    "SELECT * FROM documents WHERE id = $1 AND user_id = $2",
    [id, userId]
  );
  return rows[0];
}

/**
 * @openapi
 * /api/documents/{id}/download:
 *   get:
 *     tags: [Documents]
 *     summary: Get a short-lived download URL
 *     security: [{ bearerAuth: [] }]
 *     parameters:
 *       - { in: path, name: id, required: true, schema: { type: integer } }
 *     responses:
 *       200: { description: "{ url }" }
 *       404: { description: Not found }
 */
router.get("/:id/download", async (req, res) => {
  const doc = await findOwnDocument(Number(req.params.id), req.userId);
  if (!doc) return res.status(404).json({ error: "Not found" });

  const command = new GetObjectCommand({
    Bucket: BUCKET,
    Key: doc.s3_key,
    ResponseContentDisposition: `attachment; filename="${encodeURIComponent(doc.file_name)}"`,
  });
  const url = await getSignedUrl(s3, command, { expiresIn: 60 });
  res.json({ url });
});

/**
 * @openapi
 * /api/documents/{id}:
 *   delete:
 *     tags: [Documents]
 *     summary: Delete a document (S3 object + database row)
 *     security: [{ bearerAuth: [] }]
 *     parameters:
 *       - { in: path, name: id, required: true, schema: { type: integer } }
 *     responses:
 *       204: { description: Deleted }
 *       404: { description: Not found }
 */
router.delete("/:id", async (req, res) => {
  const doc = await findOwnDocument(Number(req.params.id), req.userId);
  if (!doc) return res.status(404).json({ error: "Not found" });

  await s3.send(new DeleteObjectCommand({ Bucket: BUCKET, Key: doc.s3_key }));
  await pool.query("DELETE FROM documents WHERE id = $1", [doc.id]);
  res.status(204).end();
});
```

### 6.1 Security test (do this manually!)

1. Sign up **user A** and **user B**.
2. Upload a document as A.
3. Login as B → `GET /api/documents` → must be empty.
4. As B, call `GET /api/documents/<A's id>/download` → must be **404**.

### ✅ Done when
List, search, pagination, download and delete all work, and user B can't see user A's files.

### 📚 You learned
`ILIKE`, pagination (`LIMIT/OFFSET`), window function `COUNT(*) OVER()`,
ownership checks, presigned GET URLs.

---

## Phase 7 — React frontend

**Goal:** Login/Signup pages, Dashboard with Home (upload) and Documents (search list).

### 7.1 Structure

```
client/src/
├── api.js
├── main.jsx
├── App.jsx
├── components/
│   ├── ProtectedRoute.jsx
│   ├── DashboardLayout.jsx
│   └── UploadBox.jsx
└── pages/
    ├── Login.jsx
    ├── Signup.jsx
    ├── Home.jsx
    └── Documents.jsx
```

### 7.2 API helper — `client/src/api.js`

```js
export function getToken() {
  return localStorage.getItem("token");
}

export async function api(path, { method = "GET", body } = {}) {
  const token = getToken();
  const res = await fetch(`/api${path}`, {
    method,
    headers: {
      "Content-Type": "application/json",
      ...(token && { Authorization: `Bearer ${token}` }),
    },
    body: body ? JSON.stringify(body) : undefined,
  });

  // token expired -> log out
  if (res.status === 401 && token) {
    localStorage.removeItem("token");
    window.location.href = "/login";
    return;
  }

  const data = res.status === 204 ? null : await res.json();
  if (!res.ok) throw new Error(data?.error || "Request failed");
  return data;
}

// fetch() can't report upload progress, so we use XMLHttpRequest for the S3 PUT
export function uploadToS3(url, file, onProgress) {
  return new Promise((resolve, reject) => {
    const xhr = new XMLHttpRequest();
    xhr.open("PUT", url);
    xhr.setRequestHeader("Content-Type", file.type);
    xhr.upload.onprogress = (e) => {
      if (e.lengthComputable) onProgress(Math.round((e.loaded / e.total) * 100));
    };
    xhr.onload = () => (xhr.status < 300 ? resolve() : reject(new Error("Upload failed")));
    xhr.onerror = () => reject(new Error("Upload failed (network/CORS)"));
    xhr.send(file);
  });
}
```

### 7.3 Routing — `client/src/main.jsx`

```jsx
import { StrictMode } from "react";
import { createRoot } from "react-dom/client";
import { BrowserRouter } from "react-router-dom";
import App from "./App.jsx";
import "./index.css";

createRoot(document.getElementById("root")).render(
  <StrictMode>
    <BrowserRouter>
      <App />
    </BrowserRouter>
  </StrictMode>
);
```

**`client/src/App.jsx`**:
```jsx
import { Routes, Route, Navigate } from "react-router-dom";
import Login from "./pages/Login.jsx";
import Signup from "./pages/Signup.jsx";
import Home from "./pages/Home.jsx";
import Documents from "./pages/Documents.jsx";
import ProtectedRoute from "./components/ProtectedRoute.jsx";
import DashboardLayout from "./components/DashboardLayout.jsx";

export default function App() {
  return (
    <Routes>
      <Route path="/login" element={<Login />} />
      <Route path="/signup" element={<Signup />} />
      <Route
        path="/dashboard"
        element={
          <ProtectedRoute>
            <DashboardLayout />
          </ProtectedRoute>
        }
      >
        <Route index element={<Home />} />
        <Route path="documents" element={<Documents />} />
      </Route>
      <Route path="*" element={<Navigate to="/dashboard" />} />
    </Routes>
  );
}
```

**`client/src/components/ProtectedRoute.jsx`**:
```jsx
import { Navigate } from "react-router-dom";
import { getToken } from "../api.js";

export default function ProtectedRoute({ children }) {
  return getToken() ? children : <Navigate to="/login" replace />;
}
```

**`client/src/components/DashboardLayout.jsx`**:
```jsx
import { NavLink, Outlet, useNavigate } from "react-router-dom";

export default function DashboardLayout() {
  const navigate = useNavigate();
  function logout() {
    localStorage.removeItem("token");
    navigate("/login");
  }
  return (
    <div style={{ display: "flex", minHeight: "100vh" }}>
      <nav style={{ width: 200, padding: 16, borderRight: "1px solid #ddd" }}>
        <h3>DocsApp</h3>
        <p><NavLink to="/dashboard" end>Home</NavLink></p>
        <p><NavLink to="/dashboard/documents">Documents</NavLink></p>
        <button onClick={logout}>Logout</button>
      </nav>
      <main style={{ flex: 1, padding: 24 }}>
        <Outlet />
      </main>
    </div>
  );
}
```

### 7.4 Auth pages

**`client/src/pages/Login.jsx`**:
```jsx
import { useState } from "react";
import { Link, useNavigate } from "react-router-dom";
import { api } from "../api.js";

export default function Login() {
  const navigate = useNavigate();
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const [error, setError] = useState("");

  async function handleSubmit(e) {
    e.preventDefault();
    setError("");
    try {
      const { token } = await api("/auth/login", { method: "POST", body: { email, password } });
      localStorage.setItem("token", token);
      navigate("/dashboard");
    } catch (err) {
      setError(err.message);
    }
  }

  return (
    <form onSubmit={handleSubmit} style={{ maxWidth: 320, margin: "80px auto" }}>
      <h2>Login</h2>
      <input placeholder="Email" value={email} onChange={(e) => setEmail(e.target.value)} />
      <input type="password" placeholder="Password" value={password}
             onChange={(e) => setPassword(e.target.value)} />
      {error && <p style={{ color: "red" }}>{error}</p>}
      <button type="submit">Login</button>
      <p>No account? <Link to="/signup">Sign up</Link></p>
    </form>
  );
}
```

**`client/src/pages/Signup.jsx`** — same as Login, but:
- title "Sign up"
- calls `api("/auth/signup", ...)`, then on success `navigate("/login")`
- link back to `/login`

(Write this one yourself — good exercise.)

### 7.5 Upload box (click + drag & drop) — `client/src/components/UploadBox.jsx`

```jsx
import { useRef, useState } from "react";
import { api, uploadToS3 } from "../api.js";

export default function UploadBox() {
  const inputRef = useRef(null);
  const [dragging, setDragging] = useState(false);
  const [progress, setProgress] = useState(null);
  const [message, setMessage] = useState("");

  async function uploadFile(file) {
    if (!file) return;
    setMessage("");
    setProgress(0);
    try {
      // 1. ask API for presigned URL
      const { uploadUrl, key } = await api("/documents/upload-url", {
        method: "POST",
        body: { fileName: file.name, mimeType: file.type, size: file.size },
      });
      // 2. upload directly to S3
      await uploadToS3(uploadUrl, file, setProgress);
      // 3. tell API it's done
      await api("/documents/confirm", { method: "POST", body: { key, fileName: file.name } });
      setMessage(`✅ ${file.name} uploaded`);
    } catch (err) {
      setMessage(`❌ ${err.message}`);
    } finally {
      setProgress(null);
    }
  }

  return (
    <div
      onClick={() => inputRef.current.click()}
      onDragOver={(e) => { e.preventDefault(); setDragging(true); }}
      onDragLeave={() => setDragging(false)}
      onDrop={(e) => {
        e.preventDefault();
        setDragging(false);
        uploadFile(e.dataTransfer.files[0]);
      }}
      style={{
        border: "2px dashed #888",
        borderRadius: 12,
        padding: 48,
        textAlign: "center",
        cursor: "pointer",
        background: dragging ? "#eef6ff" : "transparent",
      }}
    >
      <input
        ref={inputRef}
        type="file"
        hidden
        accept=".pdf,.png,.jpg,.jpeg,.txt,.docx"
        onChange={(e) => { uploadFile(e.target.files[0]); e.target.value = ""; }}
      />
      <p>Drag & drop a file here, or click to choose</p>
      <small>PDF, PNG, JPG, TXT, DOCX — max 10 MB</small>
      {progress !== null && <progress value={progress} max="100" style={{ width: "100%" }} />}
      {message && <p>{message}</p>}
    </div>
  );
}
```

**`client/src/pages/Home.jsx`**:
```jsx
import UploadBox from "../components/UploadBox.jsx";

export default function Home() {
  return (
    <>
      <h2>Home</h2>
      <UploadBox />
    </>
  );
}
```

### 7.6 Documents page with debounced search — `client/src/pages/Documents.jsx`

```jsx
import { useEffect, useState } from "react";
import { api } from "../api.js";

function formatSize(bytes) {
  if (bytes < 1024) return `${bytes} B`;
  if (bytes < 1024 * 1024) return `${(bytes / 1024).toFixed(1)} KB`;
  return `${(bytes / 1024 / 1024).toFixed(1)} MB`;
}

export default function Documents() {
  const [search, setSearch] = useState("");
  const [page, setPage] = useState(1);
  const [data, setData] = useState({ items: [], total: 0, pageSize: 20 });
  const [error, setError] = useState("");

  // debounce: wait 300ms after the user stops typing before calling the API
  useEffect(() => {
    const timer = setTimeout(() => {
      const params = new URLSearchParams({ search, page });
      api(`/documents?${params}`).then(setData).catch((e) => setError(e.message));
    }, 300);
    return () => clearTimeout(timer);
  }, [search, page]);

  async function download(id) {
    const { url } = await api(`/documents/${id}/download`);
    window.location.href = url;
  }

  async function remove(id) {
    if (!confirm("Delete this document?")) return;
    await api(`/documents/${id}`, { method: "DELETE" });
    setData((d) => ({ ...d, items: d.items.filter((doc) => doc.id !== id), total: d.total - 1 }));
  }

  const totalPages = Math.max(1, Math.ceil(data.total / data.pageSize));

  return (
    <>
      <h2>My Documents</h2>
      <input
        placeholder="Search by file name…"
        value={search}
        onChange={(e) => { setSearch(e.target.value); setPage(1); }}
      />
      {error && <p style={{ color: "red" }}>{error}</p>}
      <table width="100%">
        <thead>
          <tr><th align="left">Name</th><th>Size</th><th>Uploaded</th><th /></tr>
        </thead>
        <tbody>
          {data.items.map((doc) => (
            <tr key={doc.id}>
              <td>{doc.file_name}</td>
              <td align="center">{formatSize(doc.size_bytes)}</td>
              <td align="center">{new Date(doc.uploaded_at).toLocaleString()}</td>
              <td align="right">
                <button onClick={() => download(doc.id)}>Download</button>{" "}
                <button onClick={() => remove(doc.id)}>Delete</button>
              </td>
            </tr>
          ))}
          {data.items.length === 0 && (
            <tr><td colSpan="4" align="center">No documents found</td></tr>
          )}
        </tbody>
      </table>
      <p>
        <button disabled={page <= 1} onClick={() => setPage(page - 1)}>Prev</button>
        {" "}Page {page} / {totalPages}{" "}
        <button disabled={page >= totalPages} onClick={() => setPage(page + 1)}>Next</button>
      </p>
    </>
  );
}
```

> `size_bytes` is `BIGINT` → `pg` returns it as a **string**. `formatSize("2048")` still works
> because JS converts it in math, but be aware of this (bigint safety).

### 7.7 Style it (optional)
Use plain CSS, or install Tailwind CSS (`npm install tailwindcss @tailwindcss/vite`, follow
tailwindcss.com Vite guide). Styling is not the learning goal — keep it simple.

### ✅ Done when
Full flow works in the browser: signup → login → upload (click + drag) → see in Documents →
search → download → delete → logout.

### 📚 You learned
React Router (nested routes, protected routes), state, effects, debounce, drag & drop events,
XHR upload progress.

---

## Phase 8 — Testing & code quality

**Goal:** Automated tests that CI can run. Most important test: **users can't see each other's data**.

### 8.1 Test environment

**`server/.env.test`** (contains no real secrets, so it's fine to commit it):
```env
NODE_ENV=test
DATABASE_URL=postgres://postgres:postgres@localhost:5432/docsapp_test
JWT_SECRET=test-secret
AWS_REGION=ap-south-1
S3_BUCKET=test-bucket
```

**`server/vitest.config.js`**:
```js
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    setupFiles: ["./tests/setup.js"],
    fileParallelism: false, // tests share one DB, run files one by one
  },
});
```

**`server/tests/setup.js`**:
```js
import dotenv from "dotenv";
dotenv.config({ path: ".env.test" }); // in CI, env vars are already set; this is a no-op there
```

Run migrations on the test DB once:
```powershell
$env:DATABASE_URL="postgres://postgres:postgres@localhost:5432/docsapp_test"; npm run migrate up; Remove-Item Env:DATABASE_URL
```

### 8.2 Tests — `server/tests/documents.test.js`

```js
import { describe, it, expect, beforeEach, vi } from "vitest";
import request from "supertest";

// Fake S3 — tests must not call AWS
vi.mock("../src/s3.js", () => ({
  BUCKET: "test-bucket",
  s3: { send: vi.fn().mockResolvedValue({ ContentLength: 100, ContentType: "application/pdf" }) },
}));
vi.mock("@aws-sdk/s3-request-presigner", () => ({
  getSignedUrl: vi.fn().mockResolvedValue("https://fake-s3/presigned"),
}));

const { app } = await import("../src/app.js");
const { pool } = await import("../src/db.js");

async function signupAndLogin(email) {
  await request(app).post("/api/auth/signup").send({ email, password: "password123" });
  const res = await request(app).post("/api/auth/login").send({ email, password: "password123" });
  return { token: res.body.token, id: res.body.user.id };
}

beforeEach(async () => {
  await pool.query("TRUNCATE users RESTART IDENTITY CASCADE");
});

describe("auth", () => {
  it("rejects duplicate email", async () => {
    await request(app).post("/api/auth/signup").send({ email: "a@t.com", password: "password123" });
    const res = await request(app).post("/api/auth/signup").send({ email: "a@t.com", password: "password123" });
    expect(res.status).toBe(409);
  });

  it("rejects wrong password", async () => {
    await signupAndLogin("a@t.com");
    const res = await request(app).post("/api/auth/login").send({ email: "a@t.com", password: "wrongpass1" });
    expect(res.status).toBe(401);
  });

  it("blocks documents without a token", async () => {
    const res = await request(app).get("/api/documents");
    expect(res.status).toBe(401);
  });
});

describe("documents isolation", () => {
  it("user B cannot see or download user A's documents", async () => {
    const a = await signupAndLogin("a@t.com");
    const b = await signupAndLogin("b@t.com");

    const confirm = await request(app)
      .post("/api/documents/confirm")
      .set("Authorization", `Bearer ${a.token}`)
      .send({ key: `users/${a.id}/x-report.pdf`, fileName: "report.pdf" });
    expect(confirm.status).toBe(201);

    const listB = await request(app).get("/api/documents").set("Authorization", `Bearer ${b.token}`);
    expect(listB.body.items).toHaveLength(0);

    const dl = await request(app)
      .get(`/api/documents/${confirm.body.id}/download`)
      .set("Authorization", `Bearer ${b.token}`);
    expect(dl.status).toBe(404);
  });

  it("user cannot confirm a key in another user's folder", async () => {
    const a = await signupAndLogin("a@t.com");
    const res = await request(app)
      .post("/api/documents/confirm")
      .set("Authorization", `Bearer ${a.token}`)
      .send({ key: "users/999/evil.pdf", fileName: "evil.pdf" });
    expect(res.status).toBe(404);
  });

  it("search filters by file name", async () => {
    const a = await signupAndLogin("a@t.com");
    for (const name of ["Report 2026.pdf", "photo.png"]) {
      await request(app)
        .post("/api/documents/confirm")
        .set("Authorization", `Bearer ${a.token}`)
        .send({ key: `users/${a.id}/${name}`, fileName: name });
    }
    const res = await request(app)
      .get("/api/documents?search=report")
      .set("Authorization", `Bearer ${a.token}`);
    expect(res.body.items.map((d) => d.file_name)).toEqual(["Report 2026.pdf"]);
  });
});
```

Run: `npm test`

### 8.3 ESLint — `server/eslint.config.js`

```js
import js from "@eslint/js";
import globals from "globals";

export default [
  js.configs.recommended,
  {
    languageOptions: { globals: { ...globals.node } },
    rules: { "no-unused-vars": ["error", { argsIgnorePattern: "^_|next" }] },
  },
  { ignores: ["node_modules", "coverage"] },
];
```

Run: `npm run lint`. The Vite template already includes ESLint for the client (`npm run lint` in `client/`).

### 8.4 Frontend tests (optional but good)

```powershell
cd client
npm install -D vitest @testing-library/react @testing-library/jest-dom jsdom
```
Add `"test": "vitest run"` to `client/package.json` and in `vite.config.js` add
`test: { environment: "jsdom" }`. Write one test: render `<Login />` inside `<MemoryRouter>`
and check the "Login" heading appears.

> If you skip client tests, remove the client test step from CI in Phase 10.

### ✅ Done when
`npm test` and `npm run lint` pass in `server/` (and `client/`).

### 📚 You learned
Integration tests, mocking external services, test DB isolation, linting.

---

## Phase 9 — Production Docker image

**Goal:** A small, secure image for the API.

### 9.1 `server/Dockerfile`

```dockerfile
# ---- Stage 1: install production dependencies ----
FROM node:24-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev

# ---- Stage 2: runtime image ----
FROM node:24-alpine
WORKDIR /app
ENV NODE_ENV=production

COPY --from=deps /app/node_modules ./node_modules
COPY package.json ./
COPY src ./src
COPY migrations ./migrations

# don't run as root
USER node

EXPOSE 5000
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD wget -qO- http://localhost:5000/api/health || exit 1

CMD ["node", "src/index.js"]
```

### 9.2 `server/.dockerignore`

```
node_modules
.env
.env.*
tests
coverage
*.log
Dockerfile
```

> `node-pg-migrate` must be in **dependencies** (not devDependencies) because production
> runs migrations from inside this image. We installed it that way in Phase 1.

> `index.js` does `import "dotenv/config"` — in production there is no `.env` file inside the
> image; env vars are passed with `--env-file`, and dotenv simply does nothing.

### 9.3 Build and run locally

```powershell
cd server
docker build -t docsapp-api .
docker images docsapp-api          # check the size (~150–200 MB)

# run it, pointing at the DB container (host.docker.internal = your laptop from inside Docker)
docker run --rm -p 5001:5000 `
  -e DATABASE_URL=postgres://postgres:postgres@host.docker.internal:5432/docsapp `
  -e JWT_SECRET=dev-secret-change-me `
  --env-file .env `
  docsapp-api
```
Open http://localhost:5001/api/health.

### 9.4 Optional: full stack in compose

Add an `api` service to `docker-compose.yml` that `build: ./server`, `depends_on: db`
(condition `service_healthy`) and uses `DATABASE_URL=postgres://postgres:postgres@db:5432/docsapp`.
Notice: inside the compose network, the hostname is the **service name** `db`.

### ✅ Done when
The image builds and `/api/health` works from the container.

### 📚 You learned
Multi-stage builds, layer caching (copy `package.json` before source), non-root user,
`.dockerignore`, healthchecks, container networking.

---

## Phase 10 — CI with GitHub Actions

**Goal:** Every Pull Request runs lint + tests + build automatically.

### 10.1 Concepts

| Term | Meaning |
|---|---|
| Workflow | A YAML file in `.github/workflows/` |
| Trigger (`on`) | When it runs: `pull_request`, `push`, `workflow_call`... |
| Job | A set of steps on one fresh VM (`ubuntu-latest`) |
| Step | A command (`run`) or a reusable action (`uses`) |
| Service container | e.g. PostgreSQL running next to your job |

### 10.2 `.github/workflows/ci.yml`

```yaml
name: CI

on:
  pull_request:
  workflow_call: # lets deploy.yml reuse this workflow

jobs:
  server:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: server

    services:
      postgres:
        image: postgres:17
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: docsapp_test
        ports: ["5432:5432"]
        options: >-
          --health-cmd pg_isready
          --health-interval 5s
          --health-timeout 5s
          --health-retries 10

    env:
      NODE_ENV: test
      DATABASE_URL: postgres://postgres:postgres@localhost:5432/docsapp_test
      JWT_SECRET: test-secret
      AWS_REGION: ap-south-1
      S3_BUCKET: test-bucket

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 24
          cache: npm
          cache-dependency-path: server/package-lock.json
      - run: npm ci
      - run: npm run lint
      - run: npm run migrate up
      - run: npm test
      - run: docker build -t docsapp-api:ci .

  client:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: client
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 24
          cache: npm
          cache-dependency-path: client/package-lock.json
      - run: npm ci
      - run: npm run lint
      - run: npm test   # remove if you skipped client tests
      - run: npm run build
```

### 10.3 Branch workflow

```powershell
git checkout -b phase-10-ci
git add .
git commit -m "phase 10: add CI"
git push -u origin phase-10-ci
```
GitHub → **Pull requests → New PR** → watch the checks run in the **Actions** tab.

### 10.4 Branch protection

GitHub repo → **Settings → Branches → Add rule** (or Rulesets) for `main`:
- ✅ Require a pull request before merging
- ✅ Require status checks to pass → select `server` and `client`

Now nothing broken can reach `main`.

### ✅ Done when
A PR shows green checks; a deliberately failing test turns them red.

### 📚 You learned
CI pipelines, service containers, npm caching, branch protection, PR workflow.

---

## Phase 11 — AWS infrastructure (manual, by hand)

**Goal:** Create every AWS resource by hand once, deploy manually, and understand each piece.

All in region **ap-south-1**. Use the **default VPC** (it's free and has public subnets — no NAT needed).

### 11.1 Security groups (EC2 → Security Groups)

**`sg-api`** (for EC2):
- Inbound: **HTTP 80** from **prefix list** `com.amazonaws.global.cloudfront.origin-facing`
  (only CloudFront can reach your server)
- Outbound: all (default)
- No SSH rule — we use **SSM Session Manager** instead.

**`sg-db`** (for RDS):
- Inbound: **PostgreSQL 5432**, source = **`sg-api`** (security group, not an IP!)

### 11.2 RDS PostgreSQL

RDS → **Create database**:
- Standard create → **PostgreSQL 17**
- Template: **Free tier**
- DB instance identifier: `docsapp-db`
- Master username `postgres`, password: generate a strong one → **save it**
- Instance: **db.t4g.micro**
- Storage: gp3, **20 GB**, **disable storage autoscaling**
- Connectivity: default VPC, **Public access: No**, security group **`sg-db`** (remove default)
- Additional config → Initial database name: **`docsapp`**
- Backups: 1 day retention; disable Performance Insights/Enhanced monitoring if shown
- Create (takes ~10 min) → copy the **Endpoint**

Production `DATABASE_URL` (RDS requires SSL):
```
postgres://postgres:<PASSWORD>@<ENDPOINT>:5432/docsapp?sslmode=no-verify
```
> `no-verify` = encrypted but doesn't verify the certificate. Fine for learning; in Phase 14
> you can download the RDS CA bundle and verify properly.

### 11.3 Prod documents bucket

S3 → Create bucket `docsapp-prod-yourname` (Block public access **ON**).
CORS — fill in later once you know the CloudFront domain (step 11.8).

### 11.4 ECR repository

ECR → **Create repository** → private → `docsapp-api`.
Then → **Lifecycle policy** → rule: *Image count more than 5 → expire* (stay under 500 MB free).

### 11.5 Secrets in SSM Parameter Store

Systems Manager → **Parameter Store → Create parameter** (Standard tier, free):

| Name | Type | Value |
|---|---|---|
| `/docsapp/prod/DATABASE_URL` | SecureString | the URL from 11.2 |
| `/docsapp/prod/JWT_SECRET` | SecureString | long random string* |
| `/docsapp/prod/AWS_REGION` | String | `ap-south-1` |
| `/docsapp/prod/S3_BUCKET` | String | `docsapp-prod-yourname` |
| `/docsapp/prod/NODE_ENV` | String | `production` |
| `/docsapp/prod/PORT` | String | `5000` |

*Generate one: `node -e "console.log(require('crypto').randomBytes(48).toString('hex'))"`

### 11.6 IAM role for EC2

IAM → **Roles → Create role** → Trusted entity: **AWS service → EC2** → name `docsapp-ec2-role`.
Attach managed policies:
- `AmazonSSMManagedInstanceCore` (Session Manager + Run Command)
- `AmazonEC2ContainerRegistryReadOnly` (pull images)
- `CloudWatchAgentServerPolicy` (send logs)

Add an **inline policy** `docsapp-ec2-app`:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:PutObject", "s3:GetObject", "s3:DeleteObject"],
      "Resource": "arn:aws:s3:::docsapp-prod-yourname/*"
    },
    {
      "Effect": "Allow",
      "Action": ["ssm:GetParametersByPath", "ssm:GetParameters"],
      "Resource": "arn:aws:ssm:ap-south-1:<ACCOUNT_ID>:parameter/docsapp/prod*"
    }
  ]
}
```

### 11.7 EC2 instance

EC2 → **Launch instance**:
- Name: `docsapp-api`
- AMI: **Amazon Linux 2023** (64-bit x86)
- Type: **t3.micro** (free-tier eligible)
- Key pair: **Proceed without a key pair** (we use Session Manager)
- Network: default VPC, auto-assign public IP **enabled**, security group **`sg-api`**
- Storage: 8–20 GB gp3 (free tier allows up to 30 GB)
- Advanced → IAM instance profile: **`docsapp-ec2-role`**
- Advanced → Metadata: IMDSv2 **required**, **hop limit 2** (so Docker containers can get role credentials)
- Advanced → **User data**:
  ```bash
  #!/bin/bash
  dnf update -y
  dnf install -y docker
  systemctl enable --now docker
  usermod -aG docker ec2-user
  # 1 GB swap (t3.micro has only 1 GB RAM)
  dd if=/dev/zero of=/swapfile bs=1M count=1024
  chmod 600 /swapfile && mkswap /swapfile && swapon /swapfile
  echo '/swapfile swap swap defaults 0 0' >> /etc/fstab
  mkdir -p /opt/docsapp
  ```

Then: EC2 → **Elastic IPs → Allocate → Associate** with the instance (stable address/DNS even
after stop/start). Copy the instance's **Public IPv4 DNS**, e.g.
`ec2-13-233-1-2.ap-south-1.compute.amazonaws.com`.

**Connect:** select instance → **Connect → Session Manager → Connect** (browser shell, no SSH keys!).
```bash
sudo su -
docker -v
aws sts get-caller-identity     # shows the role — no keys configured!
```

### 11.8 Deploy script on EC2

In the Session Manager shell (as root), create **`/opt/docsapp/deploy.sh`**:
```bash
cat > /opt/docsapp/deploy.sh <<'EOF'
#!/bin/bash
# usage: deploy.sh <image-tag>
set -euo pipefail

TAG="$1"
REGION="ap-south-1"
ACCOUNT=$(aws sts get-caller-identity --query Account --output text)
REGISTRY="$ACCOUNT.dkr.ecr.$REGION.amazonaws.com"
IMAGE="$REGISTRY/docsapp-api:$TAG"
ENV_FILE="/opt/docsapp/.env"

echo "==> Logging in to ECR"
aws ecr get-login-password --region "$REGION" | docker login --username AWS --password-stdin "$REGISTRY"

echo "==> Loading secrets from SSM Parameter Store"
aws ssm get-parameters-by-path --path /docsapp/prod --with-decryption --region "$REGION" \
  --query 'Parameters[*].[Name,Value]' --output text \
  | while IFS=$'\t' read -r name value; do echo "${name##*/}=$value"; done > "$ENV_FILE"
chmod 600 "$ENV_FILE"

echo "==> Pulling $IMAGE"
docker pull "$IMAGE"

echo "==> Running migrations"
docker run --rm --env-file "$ENV_FILE" "$IMAGE" npm run migrate up

echo "==> Restarting API container"
docker rm -f docsapp-api 2>/dev/null || true
docker run -d --name docsapp-api --restart unless-stopped \
  -p 80:5000 --env-file "$ENV_FILE" \
  --log-driver awslogs \
  --log-opt awslogs-region="$REGION" \
  --log-opt awslogs-group=/docsapp/api \
  --log-opt awslogs-create-group=true \
  "$IMAGE"

echo "==> Cleaning old images"
docker image prune -af

echo "==> Done: $TAG"
EOF
chmod +x /opt/docsapp/deploy.sh
```

### 11.9 First manual deploy of the API

On your **laptop**:
```powershell
$ACCOUNT = aws sts get-caller-identity --query Account --output text
$REGISTRY = "$ACCOUNT.dkr.ecr.ap-south-1.amazonaws.com"
aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin $REGISTRY
cd server
docker build -t "$REGISTRY/docsapp-api:manual-1" .
docker push "$REGISTRY/docsapp-api:manual-1"
```
On **EC2** (Session Manager):
```bash
/opt/docsapp/deploy.sh manual-1
curl localhost/api/health          # {"status":"ok"}
docker logs docsapp-api
```

### 11.10 Frontend bucket + CloudFront

1. **S3 → Create bucket** `docsapp-web-yourname` (Block public access **ON**).
2. Build and upload the React app from your laptop:
   ```powershell
   cd client
   npm run build
   aws s3 sync dist s3://docsapp-web-yourname --delete
   ```
3. **CloudFront → Create distribution** (choose the free/pay-as-you-go plan if asked):
   - **Origin 1:** `docsapp-web-yourname` S3 bucket → Origin access: **Origin access control (OAC)** → create new OAC
   - Default behavior (`*`): Viewer protocol **Redirect HTTP to HTTPS**, Cache policy **CachingOptimized**
   - **WAF: do not enable** (costs money)
   - Default root object: **`index.html`**
   - Create → CloudFront shows a **bucket policy** → copy it → S3 bucket → Permissions → Bucket policy → paste.
4. **Add origin 2:** Origins → Create origin
   - Origin domain: the EC2 **Public IPv4 DNS** (`ec2-...compute.amazonaws.com`)
   - Protocol: **HTTP only**, port **80**
5. **Create a cache policy for the API** (Policies → Cache → Create) `docsapp-api-nocache`:
   - Min TTL 0, Default TTL 0, Max TTL 1
   - Headers: include **`Authorization`**
   - Query strings: **All**
   > Why: CloudFront only forwards `Authorization` to the origin if it's in the cache key.
   > TTL 0 means nothing is actually cached, and including the header in the key means one
   > user's response can never be served to another.
6. **Add behavior:** Behaviors → Create behavior
   - Path pattern: **`/api/*`** → origin: EC2 origin
   - Viewer protocol: Redirect HTTP to HTTPS
   - Allowed methods: **GET, HEAD, OPTIONS, PUT, POST, PATCH, DELETE**
   - Cache policy: **`docsapp-api-nocache`**
   - Origin request policy: **AllViewerExceptHostHeader**
7. **React Router refresh fix:** Error pages → Create custom error response:
   - HTTP error code **403** → Customize → Response page path `/index.html`, response code **200**
   > S3 with OAC returns 403 for unknown paths like `/dashboard/documents`. We map that to
   > `index.html` so React Router handles it. (That's why our API never returns 403.)
8. Wait until status is **Deployed** → copy the domain `dxxxxxxxx.cloudfront.net`.

### 11.11 Prod bucket CORS

S3 → `docsapp-prod-yourname` → Permissions → CORS:
```json
[
  {
    "AllowedOrigins": ["https://dxxxxxxxx.cloudfront.net"],
    "AllowedMethods": ["PUT", "GET"],
    "AllowedHeaders": ["*"],
    "ExposeHeaders": ["ETag"],
    "MaxAgeSeconds": 3000
  }
]
```

### 11.12 Test production 🎉

- `https://dxxxxxxxx.cloudfront.net/api/health` → ok
- `https://dxxxxxxxx.cloudfront.net/api/docs` → Swagger
- `https://dxxxxxxxx.cloudfront.net` → app → signup → upload → documents → download → delete
- Refresh on `/dashboard/documents` → still works

**Look at RDS from EC2** (it's private, so go through EC2):
```bash
source <(sed 's/^/export /' /opt/docsapp/.env)
docker run --rm -it postgres:17 psql "$DATABASE_URL"
```

### ✅ Done when
The app works on the CloudFront URL over HTTPS.

### 📚 You learned
VPC basics, security groups referencing each other, private RDS, IAM roles vs keys,
IMDS, SSM Parameter Store, Session Manager, ECR, CloudFront origins/behaviors/OAC,
cache vs origin request policies.

---

## Phase 12 — CD: automatic deployment

**Goal:** Merge to `main` → tests → build → push → deploy → upload frontend → smoke test. No manual steps, **no AWS keys stored in GitHub**.

### 12.1 GitHub OIDC provider (once)

IAM → **Identity providers → Add provider**:
- Type: **OpenID Connect**
- Provider URL: `https://token.actions.githubusercontent.com`
- Audience: `sts.amazonaws.com`

### 12.2 Deploy role for GitHub Actions

IAM → Roles → Create role → **Web identity** → provider above, audience `sts.amazonaws.com`
→ GitHub organization: `<your-github-user>`, repository: `docsapp`, branch: `main`
→ name `docsapp-github-deploy`.

Check the **trust policy** looks like:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          "token.actions.githubusercontent.com:sub": "repo:<your-github-user>/docsapp:ref:refs/heads/main"
        }
      }
    }
  ]
}
```
> Only the `main` branch of **your** repo can assume this role.

Inline permissions policy `docsapp-deploy`:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "ecr:GetAuthorizationToken",
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ecr:BatchCheckLayerAvailability", "ecr:InitiateLayerUpload", "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload", "ecr:PutImage", "ecr:BatchGetImage"
      ],
      "Resource": "arn:aws:ecr:ap-south-1:<ACCOUNT_ID>:repository/docsapp-api"
    },
    {
      "Effect": "Allow",
      "Action": "ssm:SendCommand",
      "Resource": [
        "arn:aws:ec2:ap-south-1:<ACCOUNT_ID>:instance/<INSTANCE_ID>",
        "arn:aws:ssm:ap-south-1::document/AWS-RunShellScript"
      ]
    },
    {
      "Effect": "Allow",
      "Action": ["ssm:GetCommandInvocation"],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:ListBucket"],
      "Resource": "arn:aws:s3:::docsapp-web-yourname"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:PutObject", "s3:DeleteObject"],
      "Resource": "arn:aws:s3:::docsapp-web-yourname/*"
    },
    {
      "Effect": "Allow",
      "Action": "cloudfront:CreateInvalidation",
      "Resource": "arn:aws:cloudfront::<ACCOUNT_ID>:distribution/<DISTRIBUTION_ID>"
    }
  ]
}
```

### 12.3 GitHub repository variables

GitHub repo → **Settings → Secrets and variables → Actions → Variables** tab:

| Variable | Value |
|---|---|
| `AWS_REGION` | `ap-south-1` |
| `AWS_DEPLOY_ROLE_ARN` | `arn:aws:iam::<ACCOUNT_ID>:role/docsapp-github-deploy` |
| `EC2_INSTANCE_ID` | `i-0abc...` |
| `FRONTEND_BUCKET` | `docsapp-web-yourname` |
| `CLOUDFRONT_DISTRIBUTION_ID` | `E1234...` |
| `CLOUDFRONT_DOMAIN` | `dxxxxxxxx.cloudfront.net` |

> These aren't secrets — the role ARN is useless without passing the OIDC trust check.

### 12.4 `.github/workflows/deploy.yml`

```yaml
name: Deploy

on:
  push:
    branches: [main]

concurrency: deploy-production # never run two deploys at once

jobs:
  ci:
    uses: ./.github/workflows/ci.yml

  deploy:
    needs: ci
    runs-on: ubuntu-latest
    permissions:
      id-token: write # required for OIDC
      contents: read
    env:
      AWS_REGION: ${{ vars.AWS_REGION }}
      IMAGE_TAG: ${{ github.sha }}

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials (OIDC, no keys)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ vars.AWS_DEPLOY_ROLE_ARN }}
          aws-region: ${{ vars.AWS_REGION }}

      - name: Login to ECR
        id: ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build and push API image
        run: |
          IMAGE=${{ steps.ecr.outputs.registry }}/docsapp-api:$IMAGE_TAG
          docker build -t $IMAGE server
          docker push $IMAGE

      - name: Deploy API to EC2 via SSM
        run: |
          CMD_ID=$(aws ssm send-command \
            --instance-ids ${{ vars.EC2_INSTANCE_ID }} \
            --document-name AWS-RunShellScript \
            --parameters commands="/opt/docsapp/deploy.sh $IMAGE_TAG" \
            --query Command.CommandId --output text)
          echo "SSM command: $CMD_ID"

          for i in $(seq 1 60); do
            sleep 5
            STATUS=$(aws ssm get-command-invocation --command-id $CMD_ID \
              --instance-id ${{ vars.EC2_INSTANCE_ID }} --query Status --output text 2>/dev/null || echo Pending)
            echo "status: $STATUS"
            case $STATUS in
              Success) break ;;
              Failed|Cancelled|TimedOut)
                aws ssm get-command-invocation --command-id $CMD_ID \
                  --instance-id ${{ vars.EC2_INSTANCE_ID }} \
                  --query '[StandardOutputContent,StandardErrorContent]' --output text
                exit 1 ;;
            esac
          done
          [ "$STATUS" = "Success" ] || { echo "Timed out"; exit 1; }

      - uses: actions/setup-node@v4
        with:
          node-version: 24
          cache: npm
          cache-dependency-path: client/package-lock.json

      - name: Build and upload frontend
        working-directory: client
        run: |
          npm ci
          npm run build
          aws s3 sync dist s3://${{ vars.FRONTEND_BUCKET }} --delete
          aws cloudfront create-invalidation \
            --distribution-id ${{ vars.CLOUDFRONT_DISTRIBUTION_ID }} --paths "/*"

      - name: Smoke test
        run: |
          sleep 10
          curl --fail --retry 5 --retry-delay 5 https://${{ vars.CLOUDFRONT_DOMAIN }}/api/health
```

### 12.5 Try it

1. Make a visible change (e.g. title "DocsApp v2") on a branch → PR → CI green → **merge**.
2. Watch **Actions → Deploy**.
3. Refresh the CloudFront URL → see the change. 🎉

### 12.6 (Optional) Manual approval

GitHub → Settings → **Environments → New** `production` → Required reviewers: you.
Add `environment: production` to the `deploy` job.
⚠️ Then the OIDC `sub` becomes `repo:<user>/docsapp:environment:production` — update the role's
trust policy to match.

### ✅ Done when
Merging a PR deploys automatically and the smoke test passes.

### 📚 You learned
CD pipelines, reusable workflows, OIDC federation, SSM Run Command, immutable image tags
(git SHA), cache invalidation, smoke tests.

---

## Phase 13 — Monitoring & operations

**Goal:** Know when things break, and know how to fix them.

### 13.1 Logs
CloudWatch → **Log groups → `/docsapp/api`** → open a stream → see pino JSON logs.
Try **Logs Insights**:
```
fields @timestamp, req.method, req.url, res.statusCode, responseTime
| filter res.statusCode >= 400
| sort @timestamp desc
| limit 50
```

Set **retention** on the log group to **7 days** (keeps you in free tier).

### 13.2 Alarms (CloudWatch → Alarms → Create, free tier = 10 alarms)
- EC2 `CPUUtilization` > 80% for 10 min
- EC2 `StatusCheckFailed` ≥ 1
- RDS `FreeStorageSpace` < 2 GB
- RDS `CPUUtilization` > 80%

Action: send to an **SNS topic** with your email (SNS email notifications are free in small amounts).

### 13.3 Practice: rollback
Find an older image tag (ECR → `docsapp-api` → images, tags = git SHAs). On EC2:
```bash
/opt/docsapp/deploy.sh <older-sha>
```
(Migrations are forward-only here — rollback works as long as the old code is compatible with the
new schema. Learn this lesson: **make migrations backward compatible**.)

### 13.4 Practice: restore DB
RDS → Snapshots → take a manual snapshot → **Restore** to a new instance → inspect it → **delete the restored instance** (it costs credits!).

### 13.5 Practice: break things on purpose
- Stop the container (`docker stop docsapp-api`) → watch `/api/health` fail; `--restart unless-stopped` doesn't restart a manual stop — run `docker start docsapp-api`.
- Reboot EC2 → container comes back automatically.
- Push a failing test → deploy is blocked.

### ✅ Done when
You can find an error in logs, receive an alarm email, and roll back.

---

## Phase 14 — Bonus / level up

Pick any, in any order:

1. **Fast search with pg_trgm** (new migration):
   ```sql
   CREATE EXTENSION IF NOT EXISTS pg_trgm;
   CREATE INDEX idx_documents_file_name_trgm ON documents USING gin (file_name gin_trgm_ops);
   ```
   Compare `EXPLAIN ANALYZE` before/after with 100k fake rows (`generate_series`).
2. **Keyset pagination** instead of `OFFSET` (`WHERE uploaded_at < $cursor`).
3. **Rate limiting** login/signup with `express-rate-limit`.
4. **httpOnly cookies** instead of `localStorage` for the JWT (protects against XSS token theft).
5. **Refresh tokens** + short-lived access tokens.
6. **Proper RDS SSL**: download the RDS CA bundle, use `ssl: { ca }` in `pg.Pool`.
7. **S3 extras**: versioning, lifecycle rule (move to Glacier after 90 days), orphan cleanup
   (uploaded-but-never-confirmed files: lifecycle rule or scheduled job).
8. **Staging environment**: second set of resources, deploy on push to a `develop` branch.
9. **Terraform** (Infrastructure as Code): run the [Teardown checklist](#teardown-checklist),
   then recreate everything from `infra/*.tf` with `terraform apply`. Best way to really
   understand Phase 11.
10. **Custom domain**: buy a domain (~$10/yr), Route 53 hosted zone ($0.50/month), free
    ACM certificate in **us-east-1** for CloudFront.
11. **Dependabot** (`.github/dependabot.yml`) + ECR image scanning on push.
12. **Thumbnails**: S3 event → Lambda creates image previews (Lambda free tier is generous).

---

## Teardown checklist

When you finish learning (or take a long break), delete in this order to stop all charges:

- [ ] CloudFront distribution → **Disable**, wait, then **Delete**
- [ ] EC2 instance → **Terminate**
- [ ] **Release the Elastic IP** (charged even when not attached!)
- [ ] RDS → Delete `docsapp-db` (optionally keep a final snapshot — snapshots cost storage)
- [ ] RDS manual snapshots → delete
- [ ] ECR repository `docsapp-api` → delete
- [ ] S3 buckets (`docsapp-web-*`, `docsapp-prod-*`, `docsapp-dev-*`) → **Empty** → Delete
- [ ] CloudWatch log groups and alarms → delete
- [ ] SSM parameters `/docsapp/*` → delete
- [ ] IAM: roles `docsapp-ec2-role`, `docsapp-github-deploy`; user `docsapp-dev` (+ its access key); policies; OIDC provider
- [ ] Security groups `sg-api`, `sg-db`
- [ ] Check **Billing → Bills** next day: everything ~$0

---

## Troubleshooting

| Problem | Likely cause / fix |
|---|---|
| `port 5432 already in use` | PostgreSQL installed on Windows → stop its service, or map `5433:5432` |
| `ECONNREFUSED 127.0.0.1:5432` | DB container not running → `docker compose up -d` |
| `relation "users" does not exist` | Migrations not run on that DB → `npm run migrate up` |
| Browser upload fails, console says **CORS** | Bucket CORS missing your origin (`http://localhost:5173` / CloudFront URL), or method `PUT` missing |
| S3 upload returns **403 SignatureDoesNotMatch** | `Content-Type` sent by browser ≠ the one used to sign; or file size ≠ `ContentLength` |
| Presigned PUT fails with checksum error | Add `requestChecksumCalculation: "WHEN_REQUIRED"` to `S3Client` |
| `401` on every request after some time | JWT expired (1 day) → login again |
| API works on EC2 (`curl localhost`) but not via CloudFront | `sg-api` missing CloudFront prefix list; origin protocol not HTTP/80; wrong behavior path `/api/*` |
| Via CloudFront, `GET` requests get **401** but `POST` works | `Authorization` not in the API cache policy (Phase 11.10 step 5) |
| Refreshing `/dashboard/documents` gives XML error | Custom error response 403 → `/index.html` (200) missing |
| Container on EC2: `Could not load credentials` | EC2 role not attached, or IMDS hop limit is 1 → set to 2 |
| RDS: `no pg_hba.conf entry ... no encryption` | Add `?sslmode=no-verify` to `DATABASE_URL` |
| RDS connection timeout | `sg-db` doesn't allow 5432 from `sg-api` |
| GitHub Actions: `Not authorized to perform sts:AssumeRoleWithWebIdentity` | Trust policy `sub` doesn't match repo/branch/environment; missing `permissions: id-token: write` |
| SSM `send-command` → `InvalidInstanceId` | Instance not registered with SSM: role missing `AmazonSSMManagedInstanceCore`, or instance has no internet access |
| EC2 runs out of memory | Swap not set up (see user data), or build images in CI, never on EC2 |
| Unexpected AWS bill | Check Billing → Cost Explorer by service; look for NAT Gateway, Load Balancer, unattached Elastic IP, forgotten RDS |

---

## Glossary

| Term | Meaning |
|---|---|
| **API** | Backend endpoints the frontend calls |
| **JWT** | Signed token proving who the user is |
| **bcrypt** | Slow hash function for passwords |
| **Migration** | Versioned schema change file |
| **Connection pool** | Reused set of DB connections |
| **Presigned URL** | Temporary URL allowing one S3 action |
| **CORS** | Browser rule controlling cross-origin requests |
| **IAM user / role** | Identity with long-term keys / identity assumed temporarily (no keys) |
| **Least privilege** | Give only the permissions needed, on only the needed resources |
| **Security group** | Virtual firewall around EC2/RDS |
| **VPC** | Your private network in AWS |
| **IMDS** | Instance Metadata Service — how EC2 gives role credentials to apps |
| **SSM** | Systems Manager — Parameter Store, Session Manager, Run Command |
| **ECR** | AWS's private Docker registry |
| **CloudFront** | AWS CDN; gives free HTTPS and routes paths to origins |
| **OAC** | Origin Access Control — lets only CloudFront read a private bucket |
| **OIDC** | Lets GitHub Actions get short-lived AWS credentials without stored keys |
| **CI** | Continuous Integration — test every change automatically |
| **CD** | Continuous Deployment — deploy every passing change automatically |
| **Smoke test** | Quick check that the deployed app is alive |
| **IaC** | Infrastructure as Code (Terraform, CloudFormation) |

---

**Next step:** start with [Phase 0](#phase-0--tools--accounts). Good luck — and commit after every phase! 🚀

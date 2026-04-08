# SETUP CHECKLIST: Before Phase 4 Code Generation

## 1. SYSTEM PREREQUISITES (Must Have)

### 1.1 Node.js Runtime
- **Download:** https://nodejs.org (choose LTS 20.x or latest LTS)
- **Verify:** Open terminal, run `node --version` and `npm --version`
- **Expected:** v20.x.x and npm 9+

### 1.2 PostgreSQL Database
**Choose ONE option:**

**Option A: Cloud (Recommended for quick demo)**
- Supabase (https://supabase.com) - Free tier included, managed PostgreSQL
  - Create free project → copy connection string
  - No local setup needed
  
**Option B: Local PostgreSQL**
- Download: https://www.postgresql.org/download (Windows installer)
- After install: create a database named `cambus_dev`
- Connection string format: `postgresql://user:password@localhost:5432/cambus_dev`

**Option C: Docker (If you have Docker installed)**
```bash
docker run --name postgres-cambus -e POSTGRES_PASSWORD=dev -e POSTGRES_DB=cambus_dev -p 5432:5432 -d postgres:15
```

→ **You'll need ONE connection string in format:** `postgresql://user:password@host:port/database`

---

## 2. CODE EDITOR & EXTENSIONS

### 2.1 Visual Studio Code (You have this)
- Already using VS Code ✓

### 2.2 Recommended Extensions (Install from VS Code store)
- **Prisma** (`prisma.prisma`) - Schema syntax highlighting + formatting
- **Thunder Client** or **REST Client** - Test API endpoints during development
- **ESLint** - Code quality
- **Prettier** - Code formatting (optional but recommended)

---

## 3. NPM PACKAGES TO INSTALL

**These will be installed automatically during `npm install`, but here's what will be included:**

### Frontend/UI Stack
```
- next@14.0+
- react@18.2+
- react-dom@18.2+
- tailwindcss@3.3+
- lucide-react@0.294+
- clsx@2.0+
```

### Backend/Data Stack
```
- @prisma/client@5.7+
- prisma@5.7+ (dev)
- next-auth@5.0.0-beta+
```

### Forms/Validation
```
- react-hook-form@7.48+
- @hookform/resolvers@3.3+
- zod@3.22+
```

### Server State/Queries
```
- @tanstack/react-query@5.25+
```

### Utilities
```
- date-fns@2.30+
- papaparse@5.4+ (CSV export)
```

### TypeScript & Dev Tools
```
- typescript@5.3+
- @types/react@18.2+
- @types/node@20.10+
- @types/papaparse@5.3+
- ts-node@10.9+
- eslint@8.54+
- eslint-config-next@14.0+
```

→ **Action:** Do NOT manually install these. Just confirm you'll run `npm install` after we generate the package.json

---

## 4. GIT & VERSION CONTROL

### 4.1 Git CLI
- **Windows:** Download from https://git-scm.com
- **Verify:** `git --version`
- Already at `C:\Users\rothl\CAMBUS` with git initialized ✓

---

## 5. ENVIRONMENT VARIABLES (Template)

After code generation, you'll create `.env.local` at project root with:

```env
# Database
DATABASE_URL="postgresql://user:password@host:port/database"

# NextAuth
NEXTAUTH_SECRET="openssl rand -base64 32"  # Generate secure random key
NEXTAUTH_URL="http://localhost:3000"

# Future: Email, external services
# SENDGRID_API_KEY=""
# SLACK_WEBHOOK=""
```

→ **Action:** Don't create this yet. Will provide after project init.

---

## 6. PRE-INSTALL CHECKLIST

Before we start code generation, confirm:

- [ ] Node.js 20+ installed on Windows
- [ ] PostgreSQL OR Supabase account ready (with connection string)
- [ ] VS Code open and ready
- [ ] `C:\Users\rothl\CAMBUS` folder exists (it does)
- [ ] You have git initialized locally (you do)
- [ ] You're ready to run `npm install` (takes ~2-3 min on first run)

---

## 7. STEP-BY-STEP STARTUP (After Code Generation)

```bash
# 1. Navigate to project
cd C:\Users\rothl\CAMBUS

# 2. Install all dependencies
npm install
# → Takes 2-3 minutes, downloads ~1.2GB node_modules

# 3. Create .env.local with your DATABASE_URL
# (template will be provided)

# 4. Generate Prisma client
npx prisma generate

# 5. Run database migrations
npx prisma migrate dev --name init

# 6. Seed database with sample data
npx prisma db seed

# 7. Start dev server
npm run dev
# → Opens http://localhost:3000

# 8. Login with seed credentials (will be in README)
```

---

## 8. WHAT YOU'LL HAVE AFTER SETUP

✅ Full Next.js project structure  
✅ PostgreSQL database with schema  
✅ API routes + Server Actions working  
✅ Authentication system (NextAuth)  
✅ React Query dev tools  
✅ Tailwind + shadcn/ui components  
✅ Seed data (10 employees, sample shifts, etc.)  
✅ Dev server running at http://localhost:3000  

---

## 9. DATABASE CONNECTION STRING EXAMPLES

**Supabase (easiest for demo):**
```
postgresql://postgres.[PROJECT_ID]:[PASSWORD]@aws-0-[REGION].pooling.supabase.com:6543/postgres
```
→ Copy from Supabase dashboard → Connection pooling

**Local PostgreSQL:**
```
postgresql://postgres:your_password@localhost:5432/cambus_dev
```

**Railway.app (alternative cloud):**
```
postgresql://[user]:[password]@[host]:[port]/[database]
```

---

## 10. READY?

Once you confirm:
1. Node.js installed
2. Database choice decided + connection string ready
3. These checklist items reviewed

I'll generate the full codebase with:
- Prisma schema
- Package.json
- Next.js app structure
- All routes & layouts
- Service layer
- Sample seed data
- README with quick-start
- env.local template

**Reply with:**
- Database choice (Supabase cloud OR local PostgreSQL)
- Connection string (or confirmation to use default local: `postgresql://postgres:postgres@localhost:5432/cambus_dev`)
- Any final tweaks to architecture doc

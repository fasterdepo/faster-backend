# Quick Start: Split Frontend & Backend

## Two Independent Repositories

### 1. Frontend Repository: `fasterdepo-frontend/`
**Technology:** Next.js 14 + React 18 + Tailwind CSS  
**Port:** 3000  
**Purpose:** User-facing web application

### 2. Backend Repository: `fasterdepo-backend/`
**Technology:** Express.js + PostgreSQL + Redis  
**Port:** 3001  
**Purpose:** REST API server + blockchain indexing

---

## 🚀 Get Started in 5 Minutes

### Step 1: Install & Configure Backend

```bash
cd fasterdepo-backend

# Install dependencies
npm install && npm --prefix app install

# Setup database
npm --prefix app run migrate

# Start backend (runs on http://localhost:3001)
npm run dev
```

### Step 2: Install & Configure Frontend (New Terminal)

```bash
cd fasterdepo-frontend

# Install dependencies  
npm install

# Create .env.local
cp .env.example .env.local

# .env.local should contain:
# NEXT_PUBLIC_API_URL=http://localhost:3001

# Start frontend (runs on http://localhost:3000)
npm run dev
```

### Step 3: Open in Browser

```
Frontend: http://localhost:3000
Backend API: http://localhost:3001
Swagger Docs: http://localhost:3001/api/docs
```

---

## 📁 Repository Structure

**Frontend** (`fasterdepo-frontend/`)
- `src/app/` - Next.js pages
- `src/components/` - React components
- `src/services/` - API integration
- `.env.example` - Environment template
- `package.json` - Dependencies

**Backend** (`fasterdepo-backend/`)
- `app/src/api/` - Express routes & controllers
- `app/src/db/` - Database schema
- `app/migrations/` - Database migrations
- `app/.env.example` - Environment template
- `app/package.json` - Dependencies

---

## ⚙️ Environment Variables

### Frontend: `.env.local`
```bash
# REQUIRED
NEXT_PUBLIC_API_URL=http://localhost:3001

# Optional
NEXT_PUBLIC_STELLAR_NETWORK=testnet
```

### Backend: `app/.env`
```bash
# Database
DATABASE_URL=postgresql://user:pass@localhost:5432/fasterdepo

# Redis
REDIS_URL=redis://localhost:6379

# Server
PORT=3001
NODE_ENV=development
JWT_SECRET=your-secret

# Stellar
STELLAR_NETWORK=testnet
```

---

## 🐳 Using Docker

### Backend (with PostgreSQL & Redis)
```bash
cd fasterdepo-backend
docker-compose up -d
# Services: Backend:3001, Postgres:5432, Redis:6379
```

### Frontend (connects to backend)
```bash
cd fasterdepo-frontend
docker-compose up -d
# Frontend: http://localhost:3000
# Make sure NEXT_PUBLIC_API_URL points to your backend
```

---

## 📝 Common Commands

### Frontend
```bash
npm run dev           # Start dev server
npm run build         # Build for production
npm test              # Run unit tests
npm run test:e2e      # Run E2E tests
npm run lint          # Check code quality
npm run format        # Format code
```

### Backend
```bash
npm run dev                    # Start dev server
npm run build                  # Build for production
npm --prefix app run test      # Run unit tests
npm run test:integration       # Run integration tests
npm run test:e2e              # Run E2E tests
npm --prefix app run migrate  # Run database migrations
npm run lint                   # Check code quality
npm run format                 # Format code
```

---

## 🔗 Communication

- **Frontend** (port 3000) makes HTTP requests to **Backend** (port 3001)
- Configure via `NEXT_PUBLIC_API_URL` environment variable
- All API endpoints documented in Swagger: `http://localhost:3001/api/docs`

---

## 🗄️ Database

### Run Migrations
```bash
cd fasterdepo-backend
npm --prefix app run migrate
```

### View/Edit Schema
```bash
npm --prefix app run drizzle:studio
# Opens GUI at http://localhost:5555
```

### Generate Migrations After Schema Changes
```bash
# Edit app/src/db/schema.ts
npm --prefix app run migrate:generate
npm --prefix app run migrate
```

---

## 🧪 Testing

### Run All Tests
```bash
# Frontend
cd fasterdepo-frontend && npm test

# Backend
cd fasterdepo-backend && npm --prefix app run test
```

### E2E Testing (requires both services running)
```bash
# Frontend E2E
npm --prefix fasterdepo-frontend run test:e2e

# Backend E2E
npm --prefix fasterdepo-backend run test:e2e
```

---

## 🔧 Troubleshooting

**Frontend can't connect to backend:**
- Check `NEXT_PUBLIC_API_URL` in `.env.local`
- Verify backend is running: `http://localhost:3001`
- Check browser console for CORS errors

**Backend won't start:**
- Check PostgreSQL is running
- Verify `DATABASE_URL` in `app/.env`
- Run migrations: `npm --prefix app run migrate`

**Database errors:**
- Ensure PostgreSQL is accessible
- Check connection string: `DATABASE_URL`
- Check logs: `npm --prefix app run migrate:check`

**Port already in use:**
- Frontend: Edit `next.config.js` or use `-p` flag
- Backend: Change `PORT` in `app/.env`

---

## 📚 Documentation

- **Frontend:** `fasterdepo-frontend/README.md`
- **Backend:** `fasterdepo-backend/README.md`
- **Contributing:** Each repo has `CONTRIBUTING.md`
- **Full Guide:** `MONOREPO_SPLIT_GUIDE.md`

---

## 🎯 Next Steps

1. ✅ Install both repos
2. ✅ Configure environment variables
3. ✅ Start backend server
4. ✅ Start frontend server
5. ✅ Open http://localhost:3000
6. 🚀 Start developing!

Happy coding! 🎉



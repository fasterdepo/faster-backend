# fasterdepo - Split Repository Setup

> Your fasterdepo project has been successfully split into two independent repositories!

## 🎯 What's New

The monorepo has been split into:

1. **fasterdepo-frontend/** - Next.js 14 web application (port 3000)
2. **fasterdepo-backend/** - Express.js API server (port 3001)

Both are fully independent and ready for development!

## 🚀 Quick Start (5 minutes)

### Terminal 1: Start Backend

```bash
cd fasterdepo-backend
npm install && npm --prefix app install
npm --prefix app run migrate
npm run dev
```

Backend will run on **http://localhost:3001**

### Terminal 2: Start Frontend

```bash
cd fasterdepo-frontend
npm install
npm run dev
```

Frontend will run on **http://localhost:3000**

### Open in Browser

- **Frontend:** http://localhost:3000
- **Backend API:** http://localhost:3001
- **Swagger Docs:** http://localhost:3001/api/docs

## 📚 Documentation

Read these files in order:

1. **[SETUP_INSTRUCTIONS.txt](SETUP_INSTRUCTIONS.txt)** ← Start here!
   - Quick reference format
   - 5-minute setup
   - Troubleshooting

2. **[SPLIT_REPOS_QUICK_START.md](SPLIT_REPOS_QUICK_START.md)**
   - Comprehensive quick start
   - Environment setup
   - Common commands

3. **[MONOREPO_SPLIT_GUIDE.md](MONOREPO_SPLIT_GUIDE.md)**
   - Full reference guide
   - Architecture details
   - Advanced topics

4. **[SPLIT_COMPLETE_SUMMARY.md](SPLIT_COMPLETE_SUMMARY.md)**
   - What was done
   - Migration details
   - Feature breakdown

5. **[MIGRATION_CHECKLIST.md](MIGRATION_CHECKLIST.md)**
   - Verification checklist
   - Pre-launch items

## 📁 Repository Structure

### Frontend

```
fasterdepo-frontend/
├── src/                      # React/Next.js source
├── e2e/                      # Playwright E2E tests
├── public/                   # Static files
├── .env.example              # Environment template
├── docker-compose.yml        # Docker setup
├── package.json              # Dependencies
├── README.md                 # Frontend docs
└── CONTRIBUTING.md           # Contributing guide
```

**Technologies:**
- Next.js 14
- React 18
- Tailwind CSS
- TanStack React Query
- React Hook Form + Zod
- Playwright (E2E)

### Backend

```
fasterdepo-backend/
├── app/                      # Backend application
│   ├── src/                  # TypeScript source
│   ├── tests/                # Test suite
│   ├── migrations/           # Database migrations
│   ├── .env.example          # Environment template
│   └── package.json          # Dependencies
├── docs/                     # Documentation
├── .kiro/                    # 8 Improvement specs
├── docker-compose.yml        # Full stack setup
├── package.json              # Root scripts
├── README.md                 # Backend docs
└── CONTRIBUTING.md           # Contributing guide
```

**Technologies:**
- Express.js
- PostgreSQL + Drizzle ORM
- Redis
- JWT Authentication
- Stellar SDK
- Pino Logging

## ⚙️ Environment Setup

### Frontend: `.env.local`

```env
NEXT_PUBLIC_API_URL=http://localhost:3001
NEXT_PUBLIC_STELLAR_NETWORK=testnet
NEXT_PUBLIC_HORIZON_URL=https://horizon-testnet.stellar.org
NEXT_PUBLIC_SOROBAN_RPC_URL=https://soroban-testnet.stellar.org
```

### Backend: `app/.env`

```env
DATABASE_URL=postgresql://fasterdepo:password@localhost:5432/fasterdepo
REDIS_URL=redis://localhost:6379
PORT=3001
NODE_ENV=development
JWT_SECRET=your-secret-key
STELLAR_NETWORK=testnet
```

## 🐳 Docker Setup

### Backend (with PostgreSQL + Redis)

```bash
cd fasterdepo-backend
docker-compose up -d
```

- Backend: http://localhost:3001
- PostgreSQL: localhost:5432
- Redis: localhost:6379

### Frontend

```bash
cd fasterdepo-frontend
docker-compose up -d
```

- Frontend: http://localhost:3000

## 📋 Common Commands

### Frontend

```bash
npm run dev              # Start dev server
npm run build            # Build for production
npm test                 # Run unit tests
npm run test:e2e         # E2E tests
npm run lint             # Check code quality
npm run format           # Format code
```

### Backend

```bash
npm run dev                      # Start dev server
npm run build                    # Build for production
npm --prefix app run test        # Unit tests
npm run test:integration         # Integration tests
npm run test:e2e                # E2E tests
npm --prefix app run migrate    # Database migrations
npm run lint                     # Check code quality
npm run format                   # Format code
```

### Database

```bash
npm --prefix app run migrate              # Apply migrations
npm --prefix app run migrate:generate     # Create migration
npm --prefix app run drizzle:studio       # Open database GUI
npm --prefix app run migrate:check        # Check status
```

## 🔗 Architecture

```
┌─────────────────────┐
│  Frontend (3000)    │
│  Next.js 14         │
│  React 18           │
└──────────┬──────────┘
           │ HTTP REST API
           │ (NEXT_PUBLIC_API_URL)
           ↓
┌─────────────────────┐
│  Backend (3001)     │
│  Express.js         │
│  PostgreSQL         │
│  Redis              │
└─────────────────────┘
```

**Benefits:**
- ✅ Independent deployment
- ✅ Separate scaling
- ✅ Clear API contract
- ✅ Different tech stacks possible
- ✅ Easier development

## 🆕 What's Included

### Frontend Repository
- ✅ All Next.js source code
- ✅ Complete component library
- ✅ E2E test suite
- ✅ Docker support
- ✅ Environment templates
- ✅ Contributing guidelines
- ✅ GitHub workflows

### Backend Repository
- ✅ All Express.js source code
- ✅ Database schema and migrations
- ✅ Complete test suite
- ✅ Docker full stack (API + DB + Redis)
- ✅ Environment templates
- ✅ Contributing guidelines
- ✅ GitHub workflows
- ✅ All 8 improvement specs with design & implementation tasks

## 🎯 Next Steps

### Immediate (Now)
1. Read [SETUP_INSTRUCTIONS.txt](SETUP_INSTRUCTIONS.txt)
2. Follow the 5-minute quick start above
3. Verify both services run locally

### Today
1. Configure environment variables
2. Test the API connection from frontend
3. Review the improvement specs

### This Week
1. Set up version control (git)
2. Configure CI/CD pipelines
3. Start implementing improvements
4. Add your own features

### This Sprint
1. Deploy to staging environment
2. Test in staging
3. Configure production deployment

## 🆘 Troubleshooting

### Frontend can't connect to backend
- Check `NEXT_PUBLIC_API_URL` in `.env.local`
- Verify backend is running: http://localhost:3001
- Check browser console for CORS errors

### Backend won't start
- Verify PostgreSQL is running
- Check `DATABASE_URL` in `app/.env`
- Run migrations: `npm --prefix app run migrate`

### Port conflicts
- Frontend: Edit `next.config.js` or use `-p` flag
- Backend: Change `PORT` in `app/.env`

### npm install fails
- Try: `npm install --legacy-peer-deps`
- Delete `node_modules` and `package-lock.json`
- Run: `npm install` again

## 📞 More Help

- **Frontend Issues**: See `fasterdepo-frontend/README.md`
- **Backend Issues**: See `fasterdepo-backend/README.md`
- **Contributing**: See respective `CONTRIBUTING.md` files
- **Architecture**: See `MONOREPO_SPLIT_GUIDE.md`

## ✨ Key Features

- 🚀 Two fully independent repositories
- 📦 Separate npm packages and dependencies
- 🐳 Complete Docker support
- 🧪 Comprehensive test suites
- 📚 Full documentation
- 🎯 8 improvement specs with tasks
- 🔄 Clear API communication
- 🌐 Production-ready setup

## 📊 Status

- ✅ Frontend repository: Ready
- ✅ Backend repository: Ready
- ✅ Documentation: Complete
- ✅ Environment setup: Complete
- ✅ Docker support: Complete
- ✅ Ready for development!

## 🎉 You're All Set!

Both repositories are fully independent and ready for development. Start with [SETUP_INSTRUCTIONS.txt](SETUP_INSTRUCTIONS.txt) and you'll be running in 5 minutes!

Happy coding! 🚀

---

**Created:** October 8, 2026  
**Status:** ✅ Complete and Ready  
**Support:** See documentation files above



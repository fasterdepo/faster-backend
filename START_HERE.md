# 🚀 FASTERDEPO - Start Here

Welcome to FASTERDEPO! Your project has been split into two independent repositories and renamed.

## 📦 What You Have

Two fully independent repositories:

1. **fasterdepo-frontend/** - Next.js 14 web application (port 3000)
2. **fasterdepo-backend/** - Express.js API server (port 3001)

## ⚡ Quick Start (5 minutes)

### Terminal 1: Backend

```bash
cd fasterdepo-backend
npm install && npm --prefix app install
npm --prefix app run migrate
npm run dev
```

Backend runs on: **http://localhost:3001**

### Terminal 2: Frontend

```bash
cd fasterdepo-frontend
npm install
npm run dev
```

Frontend runs on: **http://localhost:3000**

### Open Browser

- **Your App**: http://localhost:3000
- **API Docs**: http://localhost:3001/api/docs
- **Backend API**: http://localhost:3001

## 📚 Documentation

Read these in order:

| Document | Purpose |
|----------|---------|
| [PROJECT_RENAMED.md](PROJECT_RENAMED.md) | What was renamed |
| [SETUP_INSTRUCTIONS.txt](SETUP_INSTRUCTIONS.txt) | Quick reference |
| [SPLIT_REPOS_QUICK_START.md](SPLIT_REPOS_QUICK_START.md) | 5-minute guide |
| [MONOREPO_SPLIT_GUIDE.md](MONOREPO_SPLIT_GUIDE.md) | Comprehensive guide |
| [README_SPLIT.md](README_SPLIT.md) | Full overview |

## 📁 Project Structure

```
fasterdepo-frontend/
├── src/                 # React/Next.js code
├── .env.example         # Environment template
├── docker-compose.yml   # Docker setup
└── README.md

fasterdepo-backend/
├── app/                 # Express.js API
├── .kiro/specs/         # 8 improvement specs
├── docker-compose.yml   # Full stack (API + DB + Redis)
└── README.md
```

## 🐳 Using Docker

### Backend (with PostgreSQL + Redis)

```bash
cd fasterdepo-backend
docker-compose up -d
```

### Frontend

```bash
cd fasterdepo-frontend
docker-compose up -d
```

## 🔧 Environment Setup

### Frontend: `.env.local`

Create file: `fasterdepo-frontend/.env.local`

```env
NEXT_PUBLIC_API_URL=http://localhost:3001
```

### Backend: `app/.env`

Create file: `fasterdepo-backend/app/.env`

```env
DATABASE_URL=postgresql://fasterdepo:password@localhost:5432/fasterdepo
REDIS_URL=redis://localhost:6379
PORT=3001
NODE_ENV=development
JWT_SECRET=your-secret-key
STELLAR_NETWORK=testnet
```

Or copy from examples:
- `fasterdepo-frontend/.env.example` → `.env.local`
- `fasterdepo-backend/app/.env.example` → `app/.env`

## 📝 Common Commands

### Frontend

```bash
npm run dev          # Start dev server
npm run build        # Build for production
npm test             # Run tests
npm run test:e2e     # E2E tests
npm run lint         # Check code
npm run format       # Format code
```

### Backend

```bash
npm run dev                      # Start dev server
npm run build                    # Build for production
npm --prefix app run test        # Run tests
npm --prefix app run migrate     # Database migrations
npm --prefix app run migrate:generate  # Create migration
npm run lint                     # Check code
npm run format                   # Format code
```

## 🆘 Troubleshooting

**Frontend won't start?**
- Check Node.js: `node --version` (need 18+)
- Try: `npm install --legacy-peer-deps`

**Frontend can't connect to backend?**
- Verify backend running: http://localhost:3001
- Check `NEXT_PUBLIC_API_URL` in `.env.local`

**Backend won't start?**
- Verify PostgreSQL running
- Check `DATABASE_URL` in `app/.env`
- Run migrations: `npm --prefix app run migrate`

## 🎯 What's Next

1. ✅ Review the split repositories
2. ✅ Start both servers
3. ✅ Open http://localhost:3000
4. ✅ Review the 8 improvement specs in `fasterdepo-backend/.kiro/specs/`
5. ✅ Begin development!

## ✨ Key Features

- ✅ Two independent repositories
- ✅ Separate deployment
- ✅ Independent scaling
- ✅ Complete Docker support
- ✅ 8 improvement specs included
- ✅ Full documentation
- ✅ Production-ready

## 📞 More Information

- **Frontend specifics**: See `fasterdepo-frontend/README.md`
- **Backend specifics**: See `fasterdepo-backend/README.md`
- **Contributing**: See `CONTRIBUTING.md` in each repo
- **Full guide**: See `MONOREPO_SPLIT_GUIDE.md`

## 🎉 You're All Set!

Everything is ready to go. Start the backend and frontend servers, and you'll be running FASTERDEPO in minutes!

**Happy coding!** 🚀

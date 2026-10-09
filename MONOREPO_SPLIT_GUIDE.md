# FASTERDEPO Monorepo Split Guide

This document explains the split of the FASTERDEPO project into two independent repositories.

## Overview

The original monorepo has been split into:

1. **fasterdepo-frontend** - Next.js 14 frontend application (port 3000)
2. **fasterdepo-backend** - Node.js/Express API server (port 3001)

Both repositories are now independent and can be developed, deployed, and scaled separately.

## Repository Structure

### fasterdepo-frontend/
```
fasterdepo-frontend/
├── src/                      # React/Next.js source code
│   ├── app/                  # Next.js app directory
│   ├── components/           # React components
│   ├── hooks/                # Custom React hooks
│   ├── services/             # API integration services
│   ├── types/                # TypeScript types
│   └── utils/                # Utility functions
├── e2e/                      # Playwright E2E tests
├── public/                   # Static assets
├── .env.example              # Environment template
├── next.config.js            # Next.js configuration
├── tailwind.config.ts        # Tailwind CSS config
├── jest.config.js            # Jest testing config
├── playwright.config.ts      # Playwright config
└── package.json              # Frontend dependencies
```

**Key Dependencies:**
- Next.js 14
- React 18
- TanStack React Query
- Zustand (state management)
- React Hook Form + Zod (form validation)
- Tailwind CSS
- Playwright (E2E testing)

### fasterdepo-backend/
```
fasterdepo-backend/
├── app/                      # Backend application
│   ├── src/
│   │   ├── api/              # Express routes & controllers
│   │   ├── config/           # Configuration
│   │   ├── db/               # Drizzle ORM schema
│   │   ├── services/         # Business logic
│   │   ├── indexer/          # Stellar event indexing
│   │   ├── cron/             # Scheduled tasks
│   │   ├── email/            # Email templates
│   │   └── index.ts          # App entry point
│   ├── tests/                # Test suite
│   ├── migrations/           # Database migrations
│   ├── scripts/              # Utility scripts
│   ├── .env.example          # Environment template
│   ├── package.json          # Backend dependencies
│   └── Dockerfile            # Container image
├── docs/                     # Shared documentation
├── .kiro/                    # Kiro specs
├── docker-compose.yml        # Local development stack
├── package.json              # Root package scripts
└── README.md                 # Backend documentation
```

**Key Dependencies:**
- Express.js
- PostgreSQL (drizzle-orm)
- Redis (ioredis)
- JWT authentication
- Stellar SDK
- Pino (logging)

## Environment Configuration

### Frontend (.env.local)

```bash
# REQUIRED
NEXT_PUBLIC_API_URL=http://localhost:3001

# Optional (defaults to testnet)
NEXT_PUBLIC_STELLAR_NETWORK=testnet
NEXT_PUBLIC_HORIZON_URL=https://horizon-testnet.stellar.org
NEXT_PUBLIC_SOROBAN_RPC_URL=https://soroban-testnet.stellar.org
```

### Backend (app/.env)

```bash
# Database
DATABASE_URL=postgresql://user:password@localhost:5432/bankerchanger

# Redis
REDIS_URL=redis://localhost:6379

# Server
PORT=3001
NODE_ENV=development

# Authentication
JWT_SECRET=your-secret-key

# Stellar
STELLAR_NETWORK=testnet
```

## Development Workflow

### Option 1: Local Development (Both repos)

**Terminal 1: Backend**
```bash
cd bankerchanger-backend
npm install && npm --prefix app install
npm --prefix app run migrate
npm run dev
# API runs on http://localhost:3001
```

**Terminal 2: Frontend**
```bash
cd bankerchanger-frontend
npm install
npm run dev
# Frontend runs on http://localhost:3000
```

### Option 2: Docker Compose (Full Stack)

**Backend Stack** (includes PostgreSQL, Redis)
```bash
cd bankerchanger-backend
docker-compose up -d
# API: http://localhost:3001
# Postgres: localhost:5432
# Redis: localhost:6379
```

**Frontend Stack** (connects to backend)
```bash
cd bankerchanger-frontend
# Update docker-compose.yml to point to your backend
docker-compose up -d
# Frontend: http://localhost:3000
```

### Option 3: Docker Compose (Everything)

**From root directory** (if setting up both):
```bash
# Backend first
cd bankerchanger-backend && docker-compose up -d
# Then frontend
cd ../bankerchanger-frontend && docker-compose up -d
```

## Communication Between Repos

The frontend and backend communicate exclusively via HTTP REST API:

1. **Frontend** (port 3000) → makes HTTP requests to → **Backend** (port 3001)
2. Configuration via `NEXT_PUBLIC_API_URL` environment variable
3. No shared code or dependencies
4. Independent versioning and deployment

## API Integration

### Frontend API Calls

```typescript
// Uses NEXT_PUBLIC_API_URL environment variable
const response = await fetch(`${process.env.NEXT_PUBLIC_API_URL}/api/markets`);
```

### Backend API Endpoints

All endpoints available at: `http://localhost:3001/api/*`

**Swagger UI**: `http://localhost:3001/api/docs`

## Database & Cache

### PostgreSQL
- Used by backend only
- Schema defined in `bankerchanger-backend/app/src/db/schema.ts`
- Migrations in `bankerchanger-backend/app/migrations/`

### Redis
- Used by backend for caching
- Cache invalidation is backend responsibility
- Frontend gets fresh data via API

## Deployment

### Frontend Deployment

Can be deployed to any static hosting (Vercel, Netlify, AWS S3+CloudFront, etc.):

```bash
# Build
npm run build

# Output: .next/ directory
# Deploy the .next/ directory and public/ folder
```

Environment variables for deployment:
- `NEXT_PUBLIC_API_URL` - Backend API URL (required)

### Backend Deployment

Can be deployed as containerized service (Docker, Kubernetes, ECS, etc.):

```bash
# Build Docker image
docker build -t bankerchanger-backend:latest .

# Or deploy app/ directory directly with Node.js
npm --prefix app install --production
npm --prefix app run build:prod
npm --prefix app start
```

Environment variables for deployment:
- `DATABASE_URL` - PostgreSQL connection
- `REDIS_URL` - Redis connection
- `JWT_SECRET` - Secret key for JWT
- All other variables from `app/.env.example`

## Database Migrations

Migrations are in backend only:

```bash
# Create migration
npm --prefix app run migrate:generate

# Apply migration
npm --prefix app run migrate

# View migration history
npm --prefix app run migrate:check
```

## Testing

### Frontend Tests
```bash
cd bankerchanger-frontend
npm test              # Unit tests
npm run test:e2e      # E2E tests (requires backend)
```

### Backend Tests
```bash
cd bankerchanger-backend
npm --prefix app run test           # Unit tests
npm run test:integration           # Integration tests
npm run test:e2e                   # E2E tests
```

## CI/CD Pipelines

Each repository has independent GitHub Actions workflows:

### bankerchanger-frontend
- Lint & format check
- Unit tests
- Build verification
- E2E tests (requires backend)
- Deploy to hosting (if configured)

### bankerchanger-backend
- Lint & format check
- Unit & integration tests
- Build verification
- Database migration checks
- Deploy to hosting (if configured)

## Migration from Monorepo

If you have the old monorepo and want to use the split repos:

1. Keep old monorepo for reference
2. Clone new frontend repo: `git clone <frontend-url>`
3. Clone new backend repo: `git clone <backend-url>`
4. Update environment variables in both repos
5. Start development servers in separate terminals
6. Update any CI/CD pipelines

## Common Tasks

### Adding Dependencies

**Frontend:**
```bash
cd bankerchanger-frontend
npm install new-package
```

**Backend:**
```bash
cd bankerchanger-backend/app
npm install new-package
```

### Environment Configuration

**Frontend:**
- Copy `.env.example` to `.env.local`
- Update `NEXT_PUBLIC_API_URL`

**Backend:**
- Copy `app/.env.example` to `app/.env`
- Update all database/Redis/auth variables

### Database Schema Changes

```bash
cd bankerchanger-backend

# Edit schema in app/src/db/schema.ts

# Generate and apply migration
npm --prefix app run migrate:generate
npm --prefix app run migrate

# Verify in Drizzle Studio
npm --prefix app run drizzle:studio
```

## Troubleshooting

### Frontend can't connect to backend
- Check `NEXT_PUBLIC_API_URL` in frontend `.env.local`
- Ensure backend is running on configured port
- Check CORS configuration in backend

### Backend won't start
- Check database connection: `DATABASE_URL`
- Check Redis connection: `REDIS_URL`
- Run migrations: `npm --prefix app run migrate`

### Database migration issues
- Check PostgreSQL is running
- Verify `DATABASE_URL` is correct
- Review migration file for errors

### Port conflicts
- Frontend: change port in `next.config.js` or use `-p` flag
- Backend: change `PORT` in `.env`
- Postgres: change port in `docker-compose.yml`
- Redis: change port in `docker-compose.yml`

## Support

For issues specific to:
- **Frontend**: See `bankerchanger-frontend/CONTRIBUTING.md`
- **Backend**: See `bankerchanger-backend/CONTRIBUTING.md`

## Next Steps

1. Clone both repositories
2. Install dependencies in each
3. Configure environment variables
4. Start development servers
5. Begin development!

Happy coding! 🚀

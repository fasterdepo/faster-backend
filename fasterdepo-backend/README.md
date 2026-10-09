# FASTERDEPO Backend

A Node.js/TypeScript backend API for FASTERDEPO, a decentralized boxing prediction market built on Stellar Soroban.

## Quick Start

### Prerequisites
- Node.js 18+ and npm 9+
- PostgreSQL 14+
- Redis 6+
- Docker and Docker Compose (for containerized setup)

### Installation

```bash
# Install root dependencies
npm install

# Install app dependencies
npm --prefix app install

# Copy environment configuration
cp app/.env.example app/.env

# Update app/.env with your configuration
```

### Development

```bash
# Start development server (with auto-reload)
npm run dev

# The API will be available at http://localhost:3001
```

### Building & Production

```bash
# Build TypeScript
npm run build

# Start production server
npm --prefix app run start
```

### Database

```bash
# Run migrations
npm --prefix app run migrate

# Generate migrations from schema changes
npm --prefix app run migrate:generate

# Check migration status
npm --prefix app run migrate:check

# Open Drizzle Studio (GUI for database)
npm --prefix app run drizzle:studio
```

### Testing

```bash
# Run all tests
npm --prefix app run test

# Run integration tests
npm run test:integration

# Run E2E tests
npm run test:e2e
```

### Linting & Formatting

```bash
# Check code quality
npm run lint

# Format code
npm run format
```

## Architecture

- **Runtime**: Node.js with TypeScript
- **Framework**: Express.js
- **Database**: PostgreSQL with Drizzle ORM
- **Cache**: Redis with ioredis
- **Authentication**: JWT with refresh tokens
- **Blockchain**: Stellar Soroban via stellar-sdk
- **API Documentation**: OpenAPI/Swagger

## API Documentation

- **Swagger UI**: http://localhost:3001/api/docs
- **OpenAPI Spec**: http://localhost:3001/api/openapi.json
- **Postman Collection**: `postman/BANKERCHANGER.postman_collection.json`

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `DATABASE_URL` | Yes | PostgreSQL connection string |
| `REDIS_URL` | Yes | Redis connection string |
| `PORT` | No | Server port (default: 3001) |
| `JWT_SECRET` | Yes | Secret for JWT signing |
| `STELLAR_NETWORK` | No | Stellar network (testnet/public) |
| `NODE_ENV` | No | Environment (development/staging/production) |

See `app/.env.example` for a complete list of configuration options.

## Docker

```bash
# Build image
docker build -t bankerchanger-backend:latest .

# Run with docker-compose
docker-compose up -d
```

## Project Structure

```
app/
├── src/
│   ├── api/              # API routes and controllers
│   ├── config/           # Configuration files
│   ├── db/               # Database schema and queries
│   ├── services/         # Business logic
│   ├── middleware/       # Express middleware
│   ├── indexer/          # Stellar event indexer
│   ├── cron/             # Scheduled tasks
│   ├── email/            # Email templates and sending
│   └── index.ts          # Application entry point
├── tests/                # Test files
│   ├── integration/      # Integration tests
│   └── e2e/              # End-to-end tests
├── migrations/           # Database migrations
└── scripts/              # Utility scripts
```

## Key Features

- RESTful API for prediction market management
- Stellar Soroban blockchain integration
- Real-time event indexing
- JWT-based authentication
- Redis caching
- PostgreSQL data persistence
- Email notifications
- 2FA/TOTP support
- Admin audit logging
- Engagement system (XP, achievements, streaks)

## Frontend Integration

This backend serves the [FASTERDEPO Frontend](https://github.com/your-org/fasterdepo-frontend).

The frontend expects the API to be available at the URL configured in its `NEXT_PUBLIC_API_URL` environment variable.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for contribution guidelines.

## License

[MIT](LICENSE)

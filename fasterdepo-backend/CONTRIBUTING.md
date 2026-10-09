# Contributing to FASTERDEPO Backend

## Getting Started

1. Clone the repository
2. Install dependencies: `npm install && npm --prefix app install`
3. Copy `app/.env.example` to `app/.env` and configure
4. Set up database: `npm --prefix app run migrate`
5. Start development: `npm run dev`

## Development Workflow

### Before Committing

```bash
# Format code
npm run format

# Run linter
npm run lint

# Run tests
npm --prefix app run test
```

### Code Standards

- **TypeScript**: All code must be fully typed
- **Error Handling**: Use `AppError` class for consistency
- **Logging**: Use Pino logger (not console.log)
- **Controllers**: Validate input with Zod
- **Services**: Keep business logic separate
- **Database**: Use Drizzle ORM for queries
- **Testing**: Aim for >80% coverage

### Database Changes

When modifying the database schema:

```bash
# Update schema in app/src/db/schema.ts

# Generate migration
npm --prefix app run migrate:generate

# Review generated migration in app/migrations/

# Apply migration
npm --prefix app run migrate

# Verify with Drizzle Studio
npm --prefix app run drizzle:studio
```

### Commit Messages

Follow conventional commits:
```
feat: add new API endpoint
fix: fix bug in market resolver
docs: update API documentation
style: format code
refactor: refactor service logic
test: add unit tests
db: add database migration
ci: update CI pipeline
```

### Pull Requests

1. Create a feature branch: `git checkout -b feature/your-feature`
2. Make your changes
3. Add tests for new functionality
4. Update API documentation if needed
5. Push and create a pull request
6. Ensure CI passes
7. Request code review

## Testing

### Unit Tests
```bash
npm --prefix app run test
```

### Integration Tests
```bash
npm run test:integration
```

### E2E Tests
```bash
npm run test:e2e
```

## API Development

### Adding New Endpoints

1. Create controller in `app/src/api/controllers/`
2. Add routes in `app/src/api/routes/`
3. Add Zod schema for validation in `app/src/dto/`
4. Update OpenAPI spec in `app/openapi.yaml`
5. Add integration tests

### OpenAPI Documentation

Update `app/openapi.yaml` for all API changes:

```bash
# Validate OpenAPI spec
npm --prefix app run openapi:validate

# Generate Postman collection
npm --prefix app run openapi:postman
```

## Database Schema

Schema is defined in `app/src/db/schema.ts` using Drizzle ORM.

### Adding Tables

1. Define table in `schema.ts`
2. Generate migration: `npm --prefix app run migrate:generate`
3. Review and apply migration

### Modifying Tables

1. Update `schema.ts`
2. Generate migration
3. Review migration (edit if needed)
4. Apply migration

## Performance

- Use database indexes for frequently queried columns
- Implement Redis caching for repeated queries
- Monitor query performance with `app/scripts/test-index-queries.sql`
- Add indexes for sorting/filtering operations

## Security

- Always validate user input with Zod
- Use parameterized queries (Drizzle ORM handles this)
- Sanitize error messages in responses
- Never expose sensitive data in logs
- Use environment variables for secrets
- Implement rate limiting for API endpoints

## Blockchain Integration

### Stellar Events

The indexer monitors Stellar Soroban events and processes them. Event processing logic is in `app/src/indexer/`.

### Testing with Local Network

1. Set up Stellar Docker environment
2. Deploy contracts locally
3. Update contract addresses in `.env`
4. Test indexer with local events

## Environment Setup

### Local Development

`app/.env`:
```
DATABASE_URL=postgresql://user:password@localhost:5432/bankerchanger
REDIS_URL=redis://localhost:6379
PORT=3001
JWT_SECRET=your-secret-key
STELLAR_NETWORK=testnet
NODE_ENV=development
```

### Staging

`app/.env.staging`:
```
DATABASE_URL=postgresql://user:password@staging-db:5432/bankerchanger
REDIS_URL=redis://staging-redis:6379
PORT=3001
JWT_SECRET=your-staging-secret
STELLAR_NETWORK=testnet
NODE_ENV=staging
```

### Production

Environment variables managed in deployment system (Kubernetes/ECS secrets).

## Troubleshooting

### Database Connection Issues
```bash
# Check connection string
echo $DATABASE_URL

# Test connection
psql $DATABASE_URL -c "SELECT 1"
```

### Redis Issues
```bash
# Check Redis connection
redis-cli ping

# Check REDIS_URL
echo $REDIS_URL
```

### Build Issues
```bash
# Clean build
npm --prefix app run build -- --clean

# Check TypeScript
npm --prefix app run build
```

## Need Help?

- Check existing issues and PRs
- Read the README
- Check `app/.env.example` for configuration
- Review code comments
- Ask in discussions

Thank you for contributing!

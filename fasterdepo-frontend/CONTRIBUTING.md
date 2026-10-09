# Contributing to FASTERDEPO Frontend

## Getting Started

1. Clone the repository
2. Install dependencies: `npm install`
3. Copy `.env.example` to `.env.local` and configure
4. Ensure the backend API is running
5. Start development: `npm run dev`

## Development Workflow

### Before Committing

```bash
# Format code
npm run format

# Run linter
npm run lint

# Run tests
npm test
```

### Code Standards

- **TypeScript**: All code must be fully typed
- **Components**: Use functional components with hooks
- **State**: Use Zustand for global state
- **Styling**: Use Tailwind CSS classes
- **Forms**: Use React Hook Form + Zod validation
- **Testing**: Aim for >80% coverage

### Commit Messages

Follow conventional commits:
```
feat: add new feature
fix: fix bug
docs: update documentation
style: format code
refactor: refactor code
test: add tests
ci: update CI config
```

### Pull Requests

1. Create a feature branch: `git checkout -b feature/your-feature`
2. Make your changes
3. Add tests for new functionality
4. Update documentation if needed
5. Push and create a pull request
6. Ensure CI passes
7. Request code review

## Testing

### Unit Tests
```bash
npm test
```

### E2E Tests
```bash
# Requires backend running
npm run test:e2e
```

## Environment Setup

### For Local Development

`.env.local`:
```
NEXT_PUBLIC_API_URL=http://localhost:3001
NEXT_PUBLIC_STELLAR_NETWORK=testnet
```

### For Staging

`.env.staging`:
```
NEXT_PUBLIC_API_URL=https://staging-api.bankerchanger.app
NEXT_PUBLIC_STELLAR_NETWORK=testnet
```

### For Production

`.env.production`:
```
NEXT_PUBLIC_API_URL=https://api.bankerchanger.app
NEXT_PUBLIC_STELLAR_NETWORK=public
```

## Performance

- Aim for Lighthouse scores >90
- Keep bundle size <250KB (gzipped)
- Optimize images (use next/image)
- Lazy load components when possible
- Monitor Core Web Vitals

## Accessibility

- All interactive elements must be keyboard accessible
- Use semantic HTML
- Maintain WCAG 2.1 AA compliance
- Test with screen readers
- Use `aria-*` attributes appropriately

## Need Help?

- Check existing issues
- Read the README
- Review backend API documentation
- Ask in discussions

Thank you for contributing!

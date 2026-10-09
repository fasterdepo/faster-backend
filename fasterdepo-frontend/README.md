# FASTERDEPO Frontend

A Next.js 14 frontend for FASTERDEPO, a decentralized boxing prediction market built on Stellar Soroban.

## Quick Start

### Prerequisites
- Node.js 18+ and npm 9+
- A running BANKERCHANGER backend API server

### Installation

```bash
# Install dependencies
npm install

# Copy environment configuration
cp .env.example .env.local

# Update NEXT_PUBLIC_API_URL in .env.local to match your backend URL
# (default: http://localhost:3001 for local dev)
```

### Development

```bash
# Start frontend development server (runs on port 3000)
npm run dev

# Open http://localhost:3000 in your browser
```

The frontend will automatically connect to your backend API based on the `NEXT_PUBLIC_API_URL` environment variable.

### Building

```bash
# Build for production
npm run build

# Start production server
npm start
```

### Testing

```bash
# Run unit tests
npm test

# Run E2E tests (requires backend running)
npm run test:e2e

# Run linter
npm run lint

# Format code
npm run format
```

## Environment Variables

All environment variables must be prefixed with `NEXT_PUBLIC_` to be accessible in the browser.

| Variable | Required | Description |
|----------|----------|-------------|
| `NEXT_PUBLIC_API_URL` | Yes | Backend API URL (e.g., `http://localhost:3001`) |
| `NEXT_PUBLIC_STELLAR_NETWORK` | No | Stellar network (testnet/public, default: testnet) |
| `NEXT_PUBLIC_HORIZON_URL` | No | Horizon API URL for Stellar network |
| `NEXT_PUBLIC_SOROBAN_RPC_URL` | No | Soroban RPC URL |
| `NEXT_PUBLIC_MARKET_FACTORY_ADDRESS` | No | Market factory contract address |
| `NEXT_PUBLIC_XLM_TOKEN_ADDRESS` | No | XLM token contract address |
| `NEXT_PUBLIC_ADMIN_ADDRESSES` | No | Comma-separated list of admin wallet addresses |

## Architecture

- **Framework**: Next.js 14
- **Styling**: Tailwind CSS
- **State Management**: Zustand
- **Forms**: React Hook Form with Zod validation
- **Data Fetching**: TanStack React Query
- **Testing**: Jest + Playwright

## Key Features

- Decentralized prediction market UI
- Stellar wallet integration (Freighter)
- Real-time market updates
- Responsive design with Tailwind CSS
- Accessibility compliant (WCAG 2.1)
- E2E testing with Playwright

## Deployment

### Docker

```bash
# Build Docker image
docker build -t bankerchanger-frontend:latest .

# Run container
docker run -p 3000:3000 \
  -e NEXT_PUBLIC_API_URL=<backend-url> \
  bankerchanger-frontend:latest
```

### Standalone Deployment

The frontend is a fully independent SPA that connects to the backend API. Deploy to any static hosting (Vercel, Netlify, AWS S3, etc.) with the backend API URL configured.

## Backend Integration

This frontend connects to the [FASTERDEPO Backend API](https://github.com/your-org/fasterdepo-backend).

**Required Backend Version**: 0.1.0+

The backend must be running and accessible at the URL specified in `NEXT_PUBLIC_API_URL`.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for contribution guidelines.

## License

[MIT](LICENSE)

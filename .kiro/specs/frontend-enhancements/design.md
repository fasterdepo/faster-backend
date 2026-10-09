# Design Document: Frontend Enhancements

**Feature:** Frontend Enhancements for BANKERCHANGER  
**Status:** Design Phase  
**Tech Stack:** Next.js 14, React 19, TypeScript, TailwindCSS, React Hook Form, TanStack Query

---

## Overview

This design document outlines comprehensive frontend enhancements for BANKERCHANGER: error handling resilience, WCAG 2.1 Level AA accessibility, Core Web Vitals optimization, real-time updates via WebSocket, dark mode with system preferences, and structured analytics.

---

## Architecture

### Multi-Level Error Boundaries

**Three-Level Defense:**
1. **App-Level** (`RootLayout`): Catches critical failures, logs to monitoring service
2. **Route-Level** (each `layout.tsx`): Route-specific recovery (e.g., re-login, retry)
3. **Component-Level**: Granular boundaries for async components, prevents cascade

**Error Context Structure:**
```typescript
interface ErrorContext {
  errorCode?: string;
  userMessage: string;
  technicalMessage: string;
  timestamp: Date;
  requestId?: string;
  recoveryOptions: RecoveryOption[];
}
```

**Error Classification:**
- Network: Timeout, failed fetch → Retry, offline mode
- Validation: Invalid input → Show hints, clear form
- Authorization: 401, 403 → Refresh token, re-login
- Rate Limit: 429 → Exponential backoff, show wait
- Server: 500+ → Retry, show status page
- Unknown: Unhandled → Reload, reset state

---

## Components and Interfaces

### 1. Error Boundary System

**Enhanced ErrorBoundary Component**
- Add request tracking and error reporting
- Implement recovery actions (retry, navigate, refresh token)
- Support multiple fallback UI per error type
- Integrate with client-side error monitoring

**Route Error Handlers**
- Each route implements `error.tsx` with appropriate fallback
- Routes: `/markets`, `/portfolio`, `/governance`, `/leaderboard`
- Graceful degradation with retry mechanisms

### 2. Loading States and Skeleton Loaders

**Skeleton Component Spec:**
- `BetPanelSkeleton`, `MarketCardSkeleton`, `MarketDetailSkeleton`, `StatsRowSkeleton`
- Preserve layout exactly to prevent CLS (Cumulative Layout Shift)
- Use `animate-pulse` for visual feedback (2s opacity cycle)
- Fade-in content with 300ms transition when ready

**Implementation Pattern:**
```typescript
const { data, isLoading } = useQuery({...});
return (
  <SkeletonLoader isLoading={isLoading}>
    <Content data={data} />
  </SkeletonLoader>
);
```

### 3. Accessibility (WCAG 2.1 Level AA)

**Implementation Checklist:**
- Semantic HTML: Use `<button>`, `<header>`, `<main>`, `<nav>`, `<section>`
- ARIA Labels: `aria-label`, `aria-labelledby`, `aria-describedby` on interactive elements
- Keyboard Navigation: Tab order, Enter/Space on buttons, Escape to close modals
- Focus Management: Visible focus indicators, focus trap in modals
- Color Contrast: 4.5:1 for text, 3:1 for graphics (WCAG AA)
- Screen Reader: Announce state changes, loading states, form errors
- Form Labels: All inputs have associated `<label>` elements
- Headings: Proper hierarchy (h1→h2→h3), no skipping levels

**Audit & Remediation:**
- Run `@axe-core/playwright` in test suite for automated checks
- Manual testing with screen reader (NVDA on Windows, VoiceOver on Mac)
- Keyboard-only navigation testing
- Focus visibility validation

### 4. SEO Implementation

**Meta Tags Strategy:**
- Dynamic metadata for market pages (title, description, image)
- Open Graph tags for social sharing
- Canonical URLs to prevent duplicate content
- Structured data (`schema:Movie` for fights, `schema:Event` for matches)

**Implementation:**
- Next.js `generateMetadata()` for dynamic pages
- Structured data via `next/script` with JSON-LD
- Sitemap generation with `generateSitemapUrl()`
- robots.txt configuration

**Canonical URL Pattern:**
```typescript
// pages/markets/[id]/page.tsx
export async function generateMetadata({ params }) {
  const market = await fetchMarket(params.id);
  return {
    title: market.title,
    description: market.description,
    openGraph: {
      images: [{ url: market.image }],
    },
    alternates: {
      canonical: `https://bankerchanger.io/markets/${params.id}`,
    },
  };
}
```

### 5. Core Web Vitals Optimization

**Largest Contentful Paint (LCP):**
- Defer non-critical JS (e.g., analytics, ads)
- Preload critical fonts and images
- Optimize API response times (<200ms)
- Use skeleton loaders for perceived performance

**First Input Delay (FID):**
- Minimize main thread blocking JS
- Use `requestIdleCallback()` for non-urgent work
- Code-split routes with Next.js dynamic imports
- Optimize React Hook Form validation

**Cumulative Layout Shift (CLS):**
- Use skeleton loaders matching exact content dimensions
- Reserve space for images with `width`/`height` attributes
- Avoid injecting content above fold after load
- Use fixed dimensions for dynamic components

**Implementation:**
```typescript
// Defer non-critical JS
if (typeof window !== 'undefined') {
  if ('requestIdleCallback' in window) {
    requestIdleCallback(() => loadAnalytics());
  } else {
    setTimeout(loadAnalytics, 2000);
  }
}

// Dynamic imports for code splitting
const MarketDetail = dynamic(() => import('@/components/market/MarketDetail'), {
  loading: () => <MarketDetailSkeleton />,
  ssr: false,
});
```

### 6. Dark Mode Implementation

**Theme Architecture:**
- **Storage:** `localStorage` key `bankerchanger_theme` ('light' | 'dark')
- **System Detection:** `window.matchMedia('(prefers-color-scheme: dark)')`
- **Fallback:** System preference if localStorage not set
- **Flash Prevention:** Inline script in `<head>` sets class before paint

**Theme Provider:**
```typescript
export function ThemeProvider({ children }: { children: ReactNode }) {
  const [theme, setTheme] = useState<'light' | 'dark'>('light');
  const [mounted, setMounted] = useState(false);

  useEffect(() => {
    const stored = localStorage.getItem('bankerchanger_theme') as 'light' | 'dark' | null;
    const system = window.matchMedia('(prefers-color-scheme: dark)').matches;
    setTheme(stored ?? (system ? 'dark' : 'light'));
    setMounted(true);
  }, []);

  const toggleTheme = () => {
    const newTheme = theme === 'light' ? 'dark' : 'light';
    setTheme(newTheme);
    localStorage.setItem('bankerchanger_theme', newTheme);
    document.documentElement.classList.toggle('dark');
  };

  if (!mounted) return null;
  return (
    <ThemeContext.Provider value={{ theme, toggleTheme }}>
      {children}
    </ThemeContext.Provider>
  );
}
```

### 7. Real-Time Updates (WebSocket)

**WebSocket Architecture:**

```
┌─────────────────┐
│   Frontend      │
└────────┬────────┘
         │ WebSocket
         ↓
┌─────────────────────────────────────────┐
│  WebSocket Server (Node.js/StellarHub)  │
│  - Market price updates                 │
│  - Bet confirmations                    │
│  - Dispute notifications                │
│  - Portfolio changes                    │
└──────────────┬──────────────────────────┘
               │
         ┌─────┴─────┐
         ↓           ↓
    Backend API   Event Stream
```

**Subscription Manager Pattern:**

```typescript
interface Subscription {
  marketId?: string;
  userId?: string;
  eventType: 'market' | 'portfolio' | 'bet' | 'dispute';
  callback: (data: any) => void;
}

export class WebSocketManager {
  private ws: WebSocket | null = null;
  private subscriptions: Map<string, Subscription> = new Map();
  private reconnectAttempts = 0;
  private maxReconnectAttempts = 5;

  connect(url: string) {
    this.ws = new WebSocket(url);
    this.ws.onmessage = (event) => this.handleMessage(event.data);
    this.ws.onclose = () => this.reconnect(url);
  }

  subscribe(id: string, subscription: Subscription) {
    this.subscriptions.set(id, subscription);
    this.ws?.send(JSON.stringify({ type: 'subscribe', ...subscription }));
  }

  unsubscribe(id: string) {
    this.subscriptions.delete(id);
    this.ws?.send(JSON.stringify({ type: 'unsubscribe', id }));
  }

  private reconnect(url: string) {
    if (this.reconnectAttempts < this.maxReconnectAttempts) {
      const backoff = Math.pow(2, this.reconnectAttempts) * 1000;
      setTimeout(() => this.connect(url), backoff);
      this.reconnectAttempts++;
    }
  }
}
```

**Connection Status Hook:**
```typescript
export function useWebSocketStatus() {
  const [status, setStatus] = useState<'connecting' | 'connected' | 'disconnected'>('connecting');
  const [lastUpdate, setLastUpdate] = useState<Date | null>(null);

  return { status, lastUpdate };
}
```

### 8. Responsive Design

**Mobile-First Breakpoints:**
```typescript
const breakpoints = {
  sm: '640px',   // Mobile
  md: '768px',   // Tablet portrait
  lg: '1024px',  // Tablet landscape
  xl: '1280px',  // Desktop
  '2xl': '1536px', // Large desktop
};
```

**Layout Patterns:**
- Mobile (default): Single column, full-width
- Tablet: Two columns, optimized spacing
- Desktop: Three+ columns, sidebar navigation
- Large: Full-featured layout with extended features

**Touch-Friendly Specs:**
- Minimum button size: 44×44px (mobile)
- Touch target padding: 8px minimum
- Avoid hover-only interactions
- Swipe gestures for mobile (optional)

### 9. Form Handling with React Hook Form

**Enhanced Form Pattern:**
```typescript
type BetFormData = z.infer<typeof betSchema>;

export function BetForm() {
  const form = useForm<BetFormData>({
    resolver: zodResolver(betSchema),
    mode: 'onChange',
  });

  const onSubmit = async (data: BetFormData) => {
    try {
      await submitBet(data);
    } catch (error) {
      form.setError('root', { message: error.message });
    }
  };

  return (
    <form onSubmit={form.handleSubmit(onSubmit)} noValidate>
      <input
        {...form.register('amount')}
        aria-invalid={!!form.formState.errors.amount}
        aria-describedby="amount-error"
      />
      {form.formState.errors.amount && (
        <span id="amount-error" role="alert">
          {form.formState.errors.amount.message}
        </span>
      )}
    </form>
  );
}
```

### 10. Analytics Event Taxonomy

**Event Structure:**
```typescript
interface AnalyticsEvent {
  eventName: string;
  category: 'user' | 'market' | 'bet' | 'transaction' | 'engagement';
  userId: string;
  sessionId: string;
  timestamp: Date;
  properties: Record<string, any>;
  context: {
    page: string;
    referrer?: string;
    userAgent: string;
  };
}
```

**Event Categories:**

| Category | Events |
|----------|--------|
| **User** | signup, login, logout, profile_update, wallet_connect |
| **Market** | market_viewed, market_filtered, market_searched |
| **Bet** | bet_created, bet_confirmed, bet_cancelled, winnings_claimed |
| **Transaction** | deposit_initiated, withdrawal_initiated, fee_paid |
| **Engagement** | form_started, form_abandoned, help_clicked, feedback_submitted |

**Implementation:**
```typescript
export class AnalyticsService {
  static track(event: AnalyticsEvent) {
    const enriched = {
      ...event,
      sessionId: getSessionId(),
      timestamp: new Date(),
    };
    // Send to analytics provider (Mixpanel, Segment, etc.)
  }
}

// Usage in components
useEffect(() => {
  AnalyticsService.track({
    eventName: 'market_viewed',
    category: 'market',
    userId: user.id,
    properties: { marketId: market.id },
    context: { page: '/markets/[id]' },
  });
}, [market.id]);
```

---

## Error Handling

**Global Error Handler:**
- Catches unhandled promise rejections
- Logs with context (page, user, action)
- Shows user-friendly message
- Offers recovery options

**Recovery Strategies:**
1. **Retry**: Re-attempt failed operation with exponential backoff
2. **Fallback**: Use cached data or previous state
3. **Refresh**: Re-authenticate, refresh token, reconnect
4. **Reset**: Clear local state, return to safe state
5. **Report**: Send to support with error context

---

## Testing Strategy

**Unit Tests:**
- Error boundary rendering and recovery
- Component skeleton loading states
- ARIA label presence and correctness
- Form validation with React Hook Form
- Analytics event generation

**Integration Tests:**
- Multi-level error boundary propagation
- WebSocket subscription and message handling
- Real-time update display
- Dark mode persistence across sessions
- Form submission with API error handling

**E2E Tests:**
- Market page accessibility (keyboard navigation, screen reader)
- Bet placement error recovery flow
- WebSocket reconnection behavior
- Responsive design at breakpoints
- Analytics event tracking accuracy

**Performance Tests:**
- Core Web Vitals (LCP, FID, CLS)
- Bundle size analysis with Next.js analyzer
- Skeleton loader render performance

---

## Deployment Considerations

- Environment variables for error monitoring service and analytics provider
- Feature flags for gradual rollout of WebSocket and analytics
- Database migration for any schema changes (if needed)
- Cache invalidation strategy for static assets
- Monitoring dashboard for Core Web Vitals


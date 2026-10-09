# Design Document: Performance Optimization

## Overview

This design implements comprehensive performance optimization across database, caching, query patterns, and frontend layers. The strategy uses iterative measurement-driven optimization to reduce API latency from current baselines to target p95 < 200ms, achieve cache hit ratio > 80%, and optimize frontend load times to < 3s.

**Technology Stack:**
- Backend: Node.js, Express, TypeScript, PostgreSQL, Redis (ioredis)
- Frontend: Next.js, React, TanStack Query
- Monitoring: Prometheus, Grafana, OpenTelemetry
- Query Optimization: Query analysis, indexing, DataLoader, pagination
- Caching: Redis with LRU eviction, TTL-based invalidation, cache warming

**Key Metrics:**
- API response time p95: < 200ms (target)
- Database query p99: < 100ms
- Cache hit ratio: > 80%
- Frontend initial load: < 3s
- WebSocket latency p99: < 50ms

---

## 1. Database Query Optimization Strategy

### 1.1 Query Analysis and Profiling

**Current State:**
- PostgreSQL auto_explain enabled for slow queries (> 500ms threshold)
- Existing indexes on markets (market_id, status, scheduled_at) and bets (market_id, bettor_address, tx_hash)
- INDEX_OPTIMIZATION.md documents composite index on (market_id, claimed)
- No systematic profiling of query patterns

**Optimization Approach:**

1. **Query Performance Baselining:**
   - Run EXPLAIN ANALYZE on all high-frequency queries:
     - `listMarkets` with filters (status, weight_class, fighter)
     - `getMarket` detail view
     - `getBetsByAddress` for portfolio
     - `getMarketStats` for market details
     - `getPlatformStats` for dashboard
   - Capture current execution times and identify queries > 100ms

2. **Index Gap Analysis:**
   - Verify existing indexes are used (check pg_stat_user_indexes)
   - Identify missing indexes for WHERE, JOIN, ORDER BY, GROUP BY clauses
   - Priority indexes:
     - `markets(weight_class)` for filtered listing
     - `markets(created_at)` for trending/recent markets
     - `bets(placed_at)` for time-based queries
     - `users(created_at)` for user activity

3. **N+1 Query Elimination:**
   - Identify sequential queries in market listing (one market query + one odds calculation per market)
   - Replace with single JOIN or batch loading (DataLoader pattern)
   - Profile impact of batch loading on query time

### 1.2 Query Optimization Tactics

**Join Optimization:**
```sql
-- Current pattern (implicit N+1): fetch markets, then calculate odds per market
SELECT * FROM markets WHERE status = 'open' LIMIT 20;

-- Optimized pattern: single query with aggregated odds
SELECT 
  m.*,
  COALESCE(SUM(CASE WHEN b.side = 'A' THEN b.amount::float ELSE 0 END), 0) as pool_a,
  COALESCE(SUM(CASE WHEN b.side = 'B' THEN b.amount::float ELSE 0 END), 0) as pool_b,
  COALESCE(SUM(CASE WHEN b.side = 'draw' THEN b.amount::float ELSE 0 END), 0) as pool_draw
FROM markets m
LEFT JOIN bets b ON m.market_id = b.market_id
WHERE m.status = 'open'
GROUP BY m.id
ORDER BY m.created_at DESC
LIMIT 20;
```

**Pagination Efficiency:**
- Replace OFFSET/LIMIT with keyset pagination (cursor-based) for large datasets
- Keyset pagination avoids O(n) row skipping on large offsets
- Implementation: `WHERE market_id > :cursor ORDER BY market_id LIMIT 20`

**Query-Specific Optimizations:**
1. `listMarkets`: Add index on (status, created_at) for filtering + sorting
2. `getMarkets` with weight_class filter: Add index on (weight_class, status)
3. `calculateOdds`: Pre-aggregate odds calculation in indexed view or materialized view
4. `getPortfolioByAddress`: Use composite index (bettor_address, claimed) for efficient filtering

### 1.3 Caching at Query Layer

**Query Result Caching:**
- Cache expensive aggregations: market pool totals, odds calculations
- TTL: 60 seconds for market data (fresh odds on each page refresh)
- Invalidation: Clear cache on bet placement, market status change

---

## 2. PostgreSQL Connection Pooling Configuration

### 2.1 Current Configuration

```typescript
// BANKERCHANGER/backend/src/config/db.ts
export const pool = new Pool({
  connectionString: DATABASE_URL,
  max: parseNumber(process.env.DB_POOL_MAX, 10),           // Default: 10
  idleTimeoutMillis: parseNumber(process.env.DB_POOL_IDLE_TIMEOUT_MS, 30000), // 30s
  connectionTimeoutMillis: parseNumber(process.env.DB_POOL_CONNECTION_TIMEOUT_MS, 5000), // 5s
});
```

### 2.2 Recommended Configuration

**Production Settings:**
```typescript
export const poolConfig = {
  // Min connections: 5 (handle baseline traffic)
  // Max connections: 20 (handle peak traffic)
  max: 20,
  
  // Idle timeout: 30 seconds (balance: reuse vs. resource cleanup)
  idleTimeoutMillis: 30000,
  
  // Connection timeout: 5 seconds (fail fast on db unavailability)
  connectionTimeoutMillis: 5000,
  
  // Validation query on checkout (ensure connection is alive)
  validationQuery: 'SELECT 1',
  
  // Queue client timeout: 5 seconds (fail rather than queue indefinitely)
  max_client_idle_time: 5000,
};
```

**Environment-Based Tuning:**
```typescript
// Development: Smaller pool for resource efficiency
dev: { max: 5, idleTimeoutMillis: 60000 }

// Staging: Mid-size pool for testing
staging: { max: 15, idleTimeoutMillis: 30000 }

// Production: Larger pool for concurrency
production: { max: 20, idleTimeoutMillis: 30000 }

// Load testing: Stress-test configuration
loadtest: { max: 50, idleTimeoutMillis: 10000 }
```

### 2.3 Pool Health Monitoring

**Metrics to Track:**
```typescript
interface PoolMetrics {
  totalCount: number;      // Total connections in pool
  idleCount: number;       // Idle connections available
  waitingCount: number;    // Requests waiting for connection
  activeCount: number;     // Active connections (totalCount - idleCount)
  utilization: percentage; // (activeCount / totalCount) * 100
}
```

**Health Check Endpoint:**
```typescript
// GET /health/detailed (admin-only)
{
  status: "healthy",
  dbPool: {
    totalCount: 10,
    idleCount: 8,
    waitingCount: 0,
    utilization: "20%"  // 2 active / 10 total
  }
}
```

**Alerts:**
- `waitingCount > 5` for 2 consecutive minutes → scale up or investigate slow queries
- `utilization > 90%` for 5 minutes → increase pool size
- `idleCount = 0` and `waitingCount > 0` → connection leak or slow queries

---

## 3. Cache Architecture

### 3.1 Redis Cache Configuration

**Current Configuration:**
```typescript
// BANKERCHANGER/backend/src/config/redis.ts
export const redis = new Redis(REDIS_URL ?? 'redis://localhost:6379', {
  lazyConnect: true,
});
```

**Enhanced Configuration:**
```typescript
export const redis = new Redis({
  host: process.env.REDIS_HOST ?? 'localhost',
  port: parseInt(process.env.REDIS_PORT ?? '6379'),
  db: parseInt(process.env.REDIS_DB ?? '0'),
  
  // Connection management
  maxRetriesPerRequest: 3,
  enableReadyCheck: true,
  enableOfflineQueue: true,
  retryStrategy: (times) => {
    return Math.min(times * 50, 2000); // Exponential backoff, cap at 2s
  },
  
  // Performance
  lazyConnect: true,
  connectTimeout: 5000,
  commandTimeout: 5000,
  
  // Memory management
  maxmemory: process.env.REDIS_MAXMEMORY ?? '1gb',
  maxmemory_policy: 'allkeys-lru', // LRU eviction when memory limit reached
});

// Redis configuration settings
export const REDIS_CONFIG = {
  maxmemory: process.env.REDIS_MAXMEMORY ?? '1gb',
  maxmemory_policy: 'allkeys-lru',
};
```

### 3.2 Cache Key Strategy

**Cache Key Namespace:**
```
market:{marketId}           // Market detail (TTL: 60s)
market:odds:{marketId}      // Odds calculation (TTL: 30s)
market:stats:{marketId}     // Market statistics (TTL: 60s)
market:list:{hash}          // Paginated market list (TTL: 60s)
user:portfolio:{address}    // User portfolio (TTL: 120s)
user:bets:{address}         // User bets (TTL: 120s)
leaderboard:main            // Top 100 leaderboard (TTL: 300s)
leaderboard:weekly          // Weekly leaderboard (TTL: 300s)
platform:stats              // Platform statistics (TTL: 120s)
```

**Cache Key Hashing for Parameterized Lists:**
```typescript
// listMarkets with filters generates cache key
const cacheKey = `market:list:${hash({
  status, weight_class, fighter, dateFrom, dateTo, sort, page, limit
})}`;
```

### 3.3 TTL Strategy

| Category | TTL | Rationale |
|----------|-----|-----------|
| Market odds | 30s | Update frequently with bets |
| Market details | 60s | Update with status changes |
| Market stats | 60s | Aggregated data, stable |
| User portfolio | 120s | Changes on bet placement/claim |
| User profile | 120s | Static, infrequent changes |
| Leaderboard | 300s | Batch updates, can be stale |
| Platform stats | 120s | Aggregated, lower update frequency |
| Featured markets | 600s | Admin-controlled, stable |

### 3.4 Cache Invalidation Patterns

**Event-Based Invalidation:**
```typescript
// Market status change
async function updateMarketStatus(market_id, newStatus) {
  await db.updateMarketStatus(market_id, newStatus);
  
  // Invalidate related caches
  await redis.del(`market:${market_id}`);
  await redis.del(`market:odds:${market_id}`);
  await redis.del(`market:stats:${market_id}`);
  
  // Invalidate aggregates containing this market
  await invalidatePatternCache('market:list:*');
  await invalidatePatternCache('platform:stats');
}

// Bet placement
async function placeBet(marketId, amount, side) {
  await saveBet(marketId, amount, side);
  
  // Invalidate market odds and portfolio
  await redis.del(`market:odds:${marketId}`);
  await redis.del(`market:stats:${marketId}`);
  await redis.del(`user:portfolio:${bettor_address}`);
  
  // Invalidate leaderboard (assuming bets update ranks)
  await redis.del('leaderboard:main');
}
```

**Pattern-Based Invalidation:**
```typescript
// Efficient SCAN-based pattern invalidation
async function invalidatePatternCache(pattern: string): Promise<void> {
  const cursor = '0';
  const keys: string[] = [];
  
  // Use SCAN to avoid blocking on large key sets
  let result = await redis.scan(cursor, 'MATCH', pattern, 'COUNT', 100);
  keys.push(...result[1]);
  
  while (result[0] !== '0') {
    result = await redis.scan(result[0], 'MATCH', pattern, 'COUNT', 100);
    keys.push(...result[1]);
  }
  
  // Delete in batches
  for (let i = 0; i < keys.length; i += 100) {
    await redis.del(...keys.slice(i, i + 100));
  }
}
```

### 3.5 Cache Stampede Prevention

**Problem:** Cache expiration causes thundering herd (all requests hit DB simultaneously)

**Solution: Probabilistic Early Expiration**
```typescript
async function getWithStampedeProtection<T>(
  key: string,
  fetchFn: () => Promise<T>,
  ttl: number = 60,
): Promise<T> {
  const cached = await redis.get(key);
  
  if (cached) {
    const data = JSON.parse(cached);
    
    // 5% chance to refresh cache early if within last 20% of TTL
    if (Math.random() < 0.05) {
      redis.set(key, JSON.stringify(data), 'EX', ttl).catch(err => {
        logger.warn({ err, key }, 'Failed to refresh cache');
      });
    }
    
    return data;
  }
  
  // Cache miss - fetch fresh data
  const fresh = await fetchFn();
  await redis.set(key, JSON.stringify(fresh), 'EX', ttl);
  return fresh;
}
```

---

## 4. Query Batching with DataLoader

### 4.1 DataLoader Implementation

**Problem:** N+1 queries when fetching related data
```typescript
// Without DataLoader: N+1 queries
const markets = await getMarkets(filters);
const odds = await Promise.all(
  markets.map(m => calculateOdds(m.market_id)) // N separate queries
);
```

**Solution: DataLoader Pattern**
```typescript
// dataloader/marketOddsLoader.ts
import DataLoader from 'dataloader';

export class MarketOddsLoader {
  private loader: DataLoader<string, MarketOdds>;
  
  constructor(db: DbAdapter) {
    this.loader = new DataLoader(
      async (marketIds: string[]) => {
        // Batch query: single SQL query for all markets
        const results = await db.calculateOddsBatch(marketIds);
        
        // Return results in same order as input keys
        return marketIds.map(id => results[id]);
      },
      {
        batchScheduleFn: (callback) => {
          // Batch within same request only (not across requests)
          setImmediate(callback);
        },
      }
    );
  }
  
  async load(marketId: string): Promise<MarketOdds> {
    return this.loader.load(marketId);
  }
  
  clearAll(): void {
    this.loader.clearAll();
  }
}

// db adapter batch method
async function calculateOddsBatch(marketIds: string[]): Promise<Record<string, MarketOdds>> {
  // Single batched query
  const result = await db.query(`
    SELECT 
      m.market_id,
      COALESCE(SUM(CASE WHEN b.side = 'A' THEN b.amount::float ELSE 0 END), 0) as pool_a,
      COALESCE(SUM(CASE WHEN b.side = 'B' THEN b.amount::float ELSE 0 END), 0) as pool_b,
      COALESCE(SUM(CASE WHEN b.side = 'draw' THEN b.amount::float ELSE 0 END), 0) as pool_draw
    FROM markets m
    LEFT JOIN bets b ON m.market_id = b.market_id
    WHERE m.market_id = ANY($1)
    GROUP BY m.market_id
  `, [marketIds]);
  
  return result.rows.reduce((acc, row) => {
    acc[row.market_id] = { poolA: row.pool_a, poolB: row.pool_b, poolDraw: row.pool_draw };
    return acc;
  }, {});
}
```

**Request-Scoped DataLoader:**
```typescript
// middleware/dataloader.middleware.ts
export function dataLoaderMiddleware(req: Request, res: Response, next: NextFunction) {
  // Create fresh DataLoader instance per request
  req.loaders = {
    marketOdds: new MarketOddsLoader(db),
    userStats: new UserStatsLoader(db),
    marketStats: new MarketStatsLoader(db),
  };
  
  // Clear loaders after response sent
  res.on('finish', () => {
    req.loaders?.marketOdds.clearAll();
    req.loaders?.userStats.clearAll();
    req.loaders?.marketStats.clearAll();
  });
  
  next();
}

// Usage in controller
export async function listMarkets(req: Request, res: Response) {
  const markets = await getMarkets(filters);
  
  // DataLoader batches these calls within single request
  const marketsWithOdds = await Promise.all(
    markets.map(async m => ({
      ...m,
      odds: await req.loaders.marketOdds.load(m.market_id)
    }))
  );
  
  res.json(marketsWithOdds);
}
```

### 4.2 Batching Opportunities

| Query Pattern | Batch Size | Expected Improvement |
|---------------|-----------|----------------------|
| Market odds (listMarkets) | 20 | 20x faster |
| Market stats per market | 20 | 15x faster |
| User stats per portfolio | 1 | N/A (single) |
| Related markets | 50 | 25x faster |

---

## 5. Response Compression

### 5.1 gzip Compression Implementation

**Enable gzip Middleware:**
```typescript
// src/index.ts
import compression from 'compression';

app.use(compression({
  threshold: 1024,           // Only compress responses > 1KB
  level: 6,                  // Compression level (1-9, 6 is default)
  filter: (req, res) => {
    // Don't compress if client sends no-encoding header
    if (req.headers['x-no-compression']) {
      return false;
    }
    return compression.filter(req, res);
  }
}));
```

**Expected Compression Ratio:**
- JSON responses: 60-70% reduction (typical API responses)
- Large list endpoints: 70-80% reduction (repetitive data)
- Market data: 65% reduction
- Platform stats: 60% reduction

**Client Support:**
- Modern browsers: automatic gzip support via Accept-Encoding: gzip
- Mobile apps: configure HttpClient to request gzip
- Desktop apps: verify Accept-Encoding header

---

## 6. Frontend Optimization

### 6.1 Code Splitting Strategy

**Route-Based Splitting:**
```typescript
// pages/markets/[id].tsx
const MarketDetail = dynamic(
  () => import('../../components/MarketDetail'),
  { loading: () => <Skeleton /> }
);

export default function MarketPage({ marketId }) {
  return <MarketDetail marketId={marketId} />;
}
```

**Component Lazy Loading:**
```typescript
// Lazy load chart library (heavy dependency)
const TradingViewChart = dynamic(() => import('react-tradingview-widgets'), {
  ssr: false,
  loading: () => <div className="h-96 bg-gray-200 animate-pulse" />
});

// Lazy load comparison table (below fold)
const ComparisonTable = dynamic(() => import('./ComparisonTable'), {
  loading: () => <TableSkeleton rows={5} />
});
```

### 6.2 Image Optimization

**Next.js Image Component:**
```typescript
import Image from 'next/image';

// All images use Next.js Image for optimization
<Image
  src={fighter.imageUrl}
  alt={fighter.name}
  width={200}
  height={200}
  placeholder="blur"           // Blur placeholder while loading
  blurDataURL={blurredImage}   // Small blurred image
  priority={isAboveTheFold}    // Load immediately if above fold
  quality={80}                 // Optimize quality
/>
```

### 6.3 React Performance Optimization

**Memoization:**
```typescript
// Avoid unnecessary re-renders
const MarketCard = memo(({ market, onSelect }) => {
  return <div onClick={onSelect}>{market.title}</div>;
}, (prev, next) => prev.market.id === next.market.id);

// Memoize computed values
const odds = useMemo(() => calculateOdds(market), [market.id]);

// Memoize callbacks
const handleSelect = useCallback(
  (marketId) => { navigate(`/markets/${marketId}`); },
  [navigate]
);
```

**Virtual Scrolling for Large Lists:**
```typescript
// Use react-window for large market lists
import { FixedSizeList } from 'react-window';

<FixedSizeList
  height={600}
  itemCount={markets.length}
  itemSize={100}
  width="100%"
>
  {({ index, style }) => (
    <div style={style}>
      <MarketCard market={markets[index]} />
    </div>
  )}
</FixedSizeList>
```

### 6.4 Data Fetching Optimization

**React Query Stale-While-Revalidate:**
```typescript
const { data: markets } = useQuery({
  queryKey: ['markets', filters],
  queryFn: () => fetchMarkets(filters),
  staleTime: 60 * 1000,           // 60s: data is fresh
  gcTime: 10 * 60 * 1000,         // 10min: keep in cache
  refetchOnWindowFocus: false,
  refetchInterval: 5 * 60 * 1000, // Periodic refresh
});
```

---

## 7. Performance Monitoring Strategy

### 7.1 Backend Metrics (Prometheus)

**Key Metrics:**
```typescript
// Response time histograms
const httpRequestDuration = new Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request latency',
  labelNames: ['method', 'route', 'status_code'],
  buckets: [0.01, 0.05, 0.1, 0.5, 1, 2, 5],
});

// Database query time
const dbQueryDuration = new Histogram({
  name: 'db_query_duration_seconds',
  help: 'Database query latency',
  labelNames: ['query_type'],
  buckets: [0.001, 0.01, 0.05, 0.1, 0.5],
});

// Cache metrics
const cacheHitRate = new Counter({
  name: 'cache_hits_total',
  labelNames: ['cache_type'],
});

const cacheMissRate = new Counter({
  name: 'cache_misses_total',
  labelNames: ['cache_type'],
});

// Pool metrics
const poolSize = new Gauge({
  name: 'db_pool_size',
  help: 'Database connection pool size',
});

const poolWaiting = new Gauge({
  name: 'db_pool_waiting_requests',
  help: 'Requests waiting for DB connection',
});
```

### 7.2 Frontend Metrics (Web Vitals)

**Core Web Vitals Tracking:**
```typescript
// pages/_app.tsx
import { getCLS, getFID, getFCP, getLCP, getTTFB } from 'web-vitals';

function sendMetric(metric: any) {
  // Send to analytics/monitoring backend
  fetch('/api/metrics', {
    method: 'POST',
    body: JSON.stringify({
      name: metric.name,
      value: metric.value,
      rating: metric.rating,
    }),
  });
}

getCLS(sendMetric);   // Cumulative Layout Shift
getFID(sendMetric);   // First Input Delay
getFCP(sendMetric);   // First Contentful Paint
getLCP(sendMetric);   // Largest Contentful Paint
getTTFB(sendMetric);  // Time to First Byte
```

### 7.3 Grafana Dashboards

**Dashboard 1: API Performance**
- Response time: p50, p95, p99
- Request rate (RPS)
- Error rate
- Top slow endpoints

**Dashboard 2: Database Performance**
- Query execution time: p50, p95, p99
- Query count by type
- Connection pool utilization
- Slow query rate

**Dashboard 3: Cache Performance**
- Cache hit rate (%)
- Cache miss rate
- Eviction rate
- Memory usage

**Dashboard 4: Frontend Performance**
- Core Web Vitals (LCP, FID, CLS)
- Page load time
- Bundle size
- Error rate

### 7.4 Alerting Rules

```yaml
# Alert on slow API responses
- alert: APISlowResponse
  expr: histogram_quantile(0.95, http_request_duration_seconds) > 0.2
  for: 5m

# Alert on database query slowness
- alert: DBSlowQuery
  expr: histogram_quantile(0.99, db_query_duration_seconds) > 0.1
  for: 5m

# Alert on low cache hit ratio
- alert: LowCacheHitRate
  expr: (cache_hits / (cache_hits + cache_misses)) < 0.8
  for: 10m

# Alert on connection pool exhaustion
- alert: DBPoolExhausted
  expr: db_pool_waiting_requests > 5
  for: 2m
```

---

## 8. Testing Strategy

### 8.1 Database Query Testing

**Unit Tests:**
- Test individual query optimization (join vs N+1)
- Verify index usage with EXPLAIN ANALYZE
- Test pagination edge cases (empty results, single page, last page)

**Performance Tests:**
- Run queries against dataset with realistic scale (1M+ bets)
- Measure execution time distribution (p50, p95, p99)
- Verify query time remains < 100ms baseline

### 8.2 Cache Testing

**Unit Tests:**
- Test cache key generation consistency
- Test TTL expiration
- Test invalidation patterns

**Integration Tests:**
- Test cache hit/miss on repeated requests
- Test cache invalidation triggers (bet placement, market update)
- Test cache stampede prevention

### 8.3 Frontend Testing

**Performance Testing:**
- Lighthouse CI for bundle size and performance metrics
- Vitals Testing: core web vitals measurement on page load
- Load testing: measure performance under concurrent users

---

## 9. Rollout Strategy

### Phase 1: Database Optimization (Week 1)
1. Deploy index additions
2. Migrate to keyset pagination
3. Measure query performance improvement

### Phase 2: Connection Pooling & Caching (Week 2)
1. Deploy pool configuration tuning
2. Deploy Redis caching middleware
3. Implement cache warming cron job

### Phase 3: Query Batching (Week 3)
1. Deploy DataLoader middleware
2. Implement batch queries for N+1 patterns
3. Measure response time improvement

### Phase 4: Compression & Frontend (Week 4)
1. Deploy gzip middleware
2. Deploy code splitting and image optimization
3. Measure Core Web Vitals improvement

### Phase 5: Monitoring (Ongoing)
1. Deploy Prometheus metrics
2. Set up Grafana dashboards
3. Configure alerting rules

---

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do.*

### Property 1: Query Optimization Preserves Data Integrity

*For any* query optimization (join rewrite, index addition, or pagination implementation), the optimized query SHALL return identical result sets to the original unoptimized query, including all columns, rows, and sort order.

**Validates: Requirements 1.1, 1.2**

### Property 2: Cache Coherency

*For any* cached value and its source data, after data update, the cache SHALL be invalidated within 100ms, and subsequent queries SHALL reflect the updated value.

**Validates: Requirements 6.1, 6.2**

### Property 3: Connection Pool Bounds

*For any* duration of operation, the number of active database connections SHALL never exceed the configured `max` pool size, and the number of waiting requests SHALL remain below alert threshold.

**Validates: Requirements 2.1**

### Property 4: DataLoader Batching Correctness

*For any* set of DataLoader requests within a single request context, the loader SHALL batch them into a single database query and return results in the same order as the input keys.

**Validates: Requirements 9.1**

### Property 5: Response Compression Transparency

*For any* API response with gzip compression enabled, the decompressed response body SHALL be identical to the uncompressed response body.

**Validates: Requirements 8.1**

### Property 6: Pagination Round-Trip

*For any* paginated list, pagination parameters (page, limit) SHALL allow iterating through all results exactly once without duplication or omission.

**Validates: Requirements 3.1**

---

## Error Handling

**Database Query Failures:**
- Retry logic with exponential backoff (max 3 attempts)
- Fallback to stale cache if available
- Return 503 Service Unavailable on persistent failure

**Cache Failures:**
- Cache misses fall through to database query (not an error)
- Cache write failures logged but don't block request
- Redis connection loss logged; cache disabled until reconnected

**Connection Pool Exhaustion:**
- Requests queued with timeout (5 seconds)
- Timeout returns 503 Service Unavailable
- Alert triggered for investigation

**Performance Degradation:**
- Slow query alert (> 500ms) triggers investigation
- API latency alert (p95 > 200ms) triggers scaling review
- Cache hit ratio alert (< 80%) triggers cache strategy review

---

## Implementation Notes

1. All optimizations prioritize backward compatibility (no breaking API changes)
2. Metrics are additive (no removal of existing observability)
3. Configuration via environment variables (no code changes for tuning)
4. Gradual rollout with feature flags to enable/disable optimizations
5. Monitoring in place before and after each optimization phase
6. Load testing validates improvements before production deployment

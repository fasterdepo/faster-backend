# Implementation Plan: Performance Optimization

## Overview

This plan implements comprehensive performance optimization across database queries, connection pooling, caching, query batching, response compression, and monitoring. Work is organized in five phases with incremental validation.

## Tasks

- [ ] 1. Database Query Analysis and Index Optimization
  - [ ] 1.1 Profile and analyze slow queries
    - Run EXPLAIN ANALYZE on all high-frequency endpoints
    - Document current query execution times (getMarkets, getMarket, getBetsByAddress, getPlatformStats)
    - Identify queries exceeding 100ms baseline
    - _Requirements: 1.1, 1.2_
  
  - [ ] 1.2 Add missing database indexes
    - Create index on markets(weight_class) for filtered listing
    - Create index on markets(created_at) for trending queries
    - Create index on markets(weight_class, status) for combined filter
    - Create index on bets(placed_at) for time-based queries
    - Create migration file for index additions
    - Verify indexes are used with pg_stat_user_indexes
    - _Requirements: 1.1_
  
  - [ ] 1.3 Implement keyset (cursor-based) pagination
    - Replace OFFSET-based pagination with keyset pagination in MarketService.getMarkets
    - Add support for cursor parameter in listMarketsQuerySchema
    - Implement pagination helper that generates next cursor
    - Update API response to include pagination metadata (has_next, next_cursor)
    - _Requirements: 1.3_
  
  - [ ] 1.4 Optimize N+1 query patterns
    - Identify market odds calculation performed per-market in listMarkets
    - Rewrite market listing query with LEFT JOIN to aggregate odds in single query
    - Benchmark improvement over original N+1 pattern
    - _Requirements: 1.1_

- [ ] 2. Connection Pool Tuning and Monitoring
  - [ ] 2.1 Configure and tune PostgreSQL connection pooling
    - Update pool configuration in src/config/db.ts with recommended settings (max: 20, idle: 30s)
    - Add environment variables for pool tuning (DB_POOL_MAX, DB_POOL_IDLE_TIMEOUT_MS)
    - Add validation query on checkout to ensure connection health
    - Document pool configuration strategy in README
    - _Requirements: 2.1_
  
  - [ ] 2.2 Implement pool health monitoring
    - Add pool metrics to /health/detailed endpoint (totalCount, idleCount, waitingCount)
    - Create Prometheus gauge metrics for pool utilization
    - Add gauge for pool waiting requests
    - Update health check to report pool status
    - _Requirements: 2.1, 13.1_
  
  - [ ] 2.3 Set up pool exhaustion alerts
    - Add alerting rule for waiting requests > 5
    - Add alerting rule for utilization > 90%
    - Document alert runbook for troubleshooting
    - _Requirements: 13.1_

- [ ] 3. Redis Cache Architecture and Implementation
  - [ ] 3.1 Configure Redis with optimized settings
    - Update Redis client configuration in src/config/redis.ts
    - Add maxmemory policy (allkeys-lru) for eviction
    - Add retry strategy with exponential backoff
    - Add connection timeout and command timeout
    - Create REDIS_CONFIG constants for TTL values by cache type
    - _Requirements: 4.1_
  
  - [ ] 3.2 Implement cache key strategy
    - Create CacheKeys utility module with namespace constants
    - Implement cache key generation for: markets, odds, portfolio, leaderboards, stats
    - Add hash-based key generation for parameterized queries
    - Test cache key consistency across requests
    - _Requirements: 4.1_
  
  - [ ] 3.3 Implement TTL-based cache invalidation
    - Create cache invalidation helper functions (invalidate by key, invalidate by pattern)
    - Implement SCAN-based pattern invalidation to avoid blocking
    - Add invalidation hooks to: updateMarketStatus, placeBet, resolveMarket
    - Create batch invalidation for bulk operations
    - _Requirements: 6.1_
  
  - [ ] 3.4 Implement cache stampede prevention
    - Create getWithStampedeProtection helper function
    - Implement probabilistic early refresh (5% chance to refresh within last 20% of TTL)
    - Test under high concurrency to prevent thundering herd
    - _Requirements: 4.1_
  
  - [ ] 3.5 Implement cache warming
    - Create cron job to warm popular markets cache on startup
    - Create cron job to warm top 100 leaderboard entries on startup
    - Implement hourly leaderboard refresh
    - Log cache warming metrics (items cached, duration)
    - _Requirements: 5.1_

- [ ] 4. Query Batching with DataLoader
  - [ ] 4.1 Implement DataLoader infrastructure
    - Create dataloader/ directory with MarketOddsLoader class
    - Implement batch loading for market odds calculation
    - Create UserStatsLoader for user portfolio batching
    - Create MarketStatsLoader for market statistics batching
    - _Requirements: 9.1_
  
  - [ ] 4.2 Add DataLoader middleware
    - Create middleware/dataloader.middleware.ts
    - Initialize DataLoader instances per request
    - Clear loaders after response sent (prevent memory leaks)
    - Add loaders to Request type (TypeScript)
    - _Requirements: 9.1_
  
  - [ ] 4.3 Implement batch database queries
    - Add calculateOddsBatch method to DbAdapter
    - Add getUserStatsBatch method for portfolio queries
    - Add getMarketStatsBatch for market detail queries
    - Return results in same order as input keys for DataLoader compatibility
    - _Requirements: 9.1_
  
  - [ ] 4.4 Integrate DataLoader into controllers
    - Update MarketController.listMarkets to use marketOddsLoader
    - Update MarketController.getMarket to use loaders for nested data
    - Update BetController to use userStatsLoader for portfolio data
    - Benchmark improvement: compare with/without DataLoader
    - _Requirements: 9.1_

- [ ] 5. Response Compression and Frontend Optimization
  - [ ] 5.1 Enable gzip response compression
    - Install compression middleware (npm install compression)
    - Add compression middleware to src/index.ts
    - Configure threshold (1KB), level (6), and exclusions
    - Test compression ratio on real responses (expect 60-70% for JSON)
    - _Requirements: 8.1_
  
  - [ ] 5.2 Implement code splitting in frontend
    - Install dynamic import support in Next.js (already available)
    - Create route-based code splitting for major sections
    - Add dynamic imports for heavy components (charts, tables)
    - Use React.lazy for component splitting
    - Configure loading placeholders for split chunks
    - _Requirements: 10.1_
  
  - [ ] 5.3 Optimize images and implement lazy loading
    - Replace all <img> tags with Next.js Image component
    - Add placeholder blur strategy with blurDataURL
    - Set priority=true for above-fold images
    - Configure responsive image sizes
    - Test image load times in Lighthouse
    - _Requirements: 11.1_
  
  - [ ] 5.4 Optimize React rendering performance
    - Wrap memoizable components with React.memo
    - Replace computed values with useMemo in hot paths
    - Replace event handlers with useCallback
    - Implement virtual scrolling for large market lists
    - Profile React render times with DevTools
    - _Requirements: 12.1_

- [ ] 6. Performance Monitoring Setup
  - [ ] 6.1 Implement Prometheus metrics for backend
    - Create metrics for: http_request_duration_seconds, db_query_duration_seconds, cache_hits_total, cache_misses_total
    - Add histogram buckets appropriate for response times
    - Add gauge metrics for pool utilization
    - Wire metrics into request/response handlers
    - Test metrics output on /metrics endpoint
    - _Requirements: 13.1_
  
  - [ ] 6.2 Implement frontend performance metrics
    - Install web-vitals library (npm install web-vitals)
    - Create metrics tracking for Core Web Vitals (LCP, FID, CLS, FCP, TTFB)
    - Send metrics to backend /api/metrics endpoint
    - Aggregate metrics for dashboard visualization
    - _Requirements: 13.1, 12.1_
  
  - [ ] 6.3 Set up Grafana dashboards
    - Create dashboard for API Performance (response time p50/p95/p99, RPS, error rate)
    - Create dashboard for Database Performance (query time, pool utilization, slow queries)
    - Create dashboard for Cache Performance (hit rate, miss rate, eviction rate, memory)
    - Create dashboard for Frontend Performance (Core Web Vitals, page load time)
    - Create dashboard for System Overview (all metrics in one view)
    - _Requirements: 13.1_
  
  - [ ] 6.4 Configure alerting rules
    - Create alerting rule: APISlowResponse (p95 > 200ms for 5m)
    - Create alerting rule: DBSlowQuery (p99 > 100ms for 5m)
    - Create alerting rule: LowCacheHitRate (< 80% for 10m)
    - Create alerting rule: DBPoolExhausted (waiting > 5 for 2m)
    - Create alerting rule: HighErrorRate (> 1% for 5m)
    - _Requirements: 13.1_

- [ ] 7. Checkpoint - Integration and Load Testing
  - [ ] 7.1 Integration test all optimizations
    - Run full test suite to ensure no regressions
    - Test database queries return correct data with new indexes
    - Test cache invalidation triggers work end-to-end
    - Test DataLoader batching within request context
    - Test compression works with client requests
    - _Requirements: All_
  
  - [ ] 7.2 Load test performance improvements
    - Run load test with 100 concurrent users for 10 minutes
    - Measure response time distribution before/after optimizations
    - Verify cache hit ratio reaches > 80%
    - Verify API response time p95 < 200ms
    - Verify database query p99 < 100ms
    - Capture metrics in dashboard for baseline
    - _Requirements: All_
  
  - [ ] 7.3 Document performance baselines
    - Record baseline metrics before optimization deployment
    - Document expected improvements from each optimization
    - Create performance regression test (CI check for regressions)
    - Document performance tuning guide for ops team
    - _Requirements: 13.1_

- [ ] 8. Deploy to Staging and Monitor
  - [ ] 8.1 Deploy all optimizations to staging environment
    - Deploy database schema changes (indexes)
    - Deploy backend code (DataLoader, cache, compression)
    - Deploy frontend code (code splitting, images, React optimization)
    - Deploy monitoring (Prometheus, Grafana)
    - Verify all metrics are being collected
    - _Requirements: All_
  
  - [ ] 8.2 Monitor staging performance for 24 hours
    - Monitor API response times, query execution, cache hit ratio
    - Verify no errors from new code paths
    - Verify alerts are triggered appropriately
    - Check for memory leaks or connection issues
    - Document any issues or unexpected behaviors
    - _Requirements: 13.1_
  
  - [ ] 8.3 Compare staging to production metrics
    - Pull metrics for same traffic patterns
    - Calculate performance improvements (expected vs actual)
    - Verify improvements meet targets (p95 < 200ms, cache > 80%)
    - Document performance improvement report
    - _Requirements: All_

- [ ] 9. Final Performance Validation
  - [ ] 9.1 End-to-end performance verification
    - Execute full user journey (browse markets, place bet, view portfolio)
    - Measure end-to-end latency with all optimizations enabled
    - Verify WebSocket messages complete < 50ms
    - Verify frontend page load < 3 seconds on 4G
    - _Requirements: 7.1, 10.1, 12.1_
  
  - [ ] 9.2 Document performance tuning guide
    - Create guide for adjusting cache TTLs
    - Create guide for connection pool tuning by load
    - Create guide for monitoring and alerting
    - Create troubleshooting guide for common issues
    - _Requirements: 13.1_

## Notes

- All performance optimizations are backward compatible (no breaking API changes)
- Feature flags can enable/disable optimizations gradually
- Metrics must be collected before and after each optimization phase
- Load testing should be run in staging before production deployment
- All database changes require migration files and can be rolled back
- Cache invalidation is designed for consistency (prefers freshness over speed)
- DataLoader batching happens per-request, not across requests

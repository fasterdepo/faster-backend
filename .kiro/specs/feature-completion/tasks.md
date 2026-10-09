# Implementation Plan: Feature Completion

## Overview

Implement the engagement system, admin features, and rewards system for BANKERCHANGER. The work is structured in 7 major phases: services for core logic, API endpoints, background jobs, real-time features, notification system, admin tooling, and testing/validation.

## Tasks

### Phase 1: Core Service Implementation

- [ ] 1. Implement StreakService and database operations
  - Create StreakService class with recordResult, getStreak, getStreakStats
  - Implement streak calculation logic (increment on win, reset on loss)
  - Add database queries: insert/update user_streaks, fetch by user_id
  - Add transaction locking for concurrent win/loss updates
  - Test streak logic with edge cases (loss, tie scenarios)
  - _Requirements: 1.1, 1.2, 1.3_

- [ ] 2. Implement AchievementService and award triggers
  - Create AchievementService with evaluateUserForAchievements, awardAchievement
  - Load all achievements from database during initialization
  - Implement achievement criteria evaluation (streak, volume, referral, xp)
  - Add idempotent award logic (check for duplicate awards)
  - Implement trigger points: bet result, daily job, level up
  - _Requirements: 2.1, 2.2, 2.3_

- [ ] 3. Implement LeaderboardService with caching
  - Create LeaderboardService with getLeaderboard, getUserRank, getLocalLeaderboard
  - Set up Redis caching with 5-minute TTL for leaderboard pages
  - Implement ranking algorithm (win %, total wins, volume, timestamp)
  - Add cache invalidation on score changes
  - Implement database queries for leaderboard rankings
  - _Requirements: 3.1, 3.2, 3.3_

- [ ] 4. Implement XPService for level progression
  - Create XPService with awardXP, getLevel, getXPLeaderboard
  - Implement XP calculation (1000 XP per level)
  - Add XP event tracking with transaction hash de-duplication
  - Implement level-up detection and milestone checks
  - Create XP leaderboard with proper sorting
  - _Requirements: 9.1, 9.2_

- [ ] 5. Implement ReferralService with payout calculation
  - Create ReferralService with multi-tier payout logic
  - Implement calculatePayoutsForBet for tier 1, 2, 3 calculation
  - Add getReferralSummary for referrer statistics
  - Implement payout recording (pending state)
  - Add referral code generation and tracking
  - _Requirements: 6.1, 6.2, 7.1_

- [ ] 6. Expand AuditLogService with comprehensive logging
  - Enhance auditLog service with query filters (date, admin, action, target)
  - Add getSummary for admin action statistics
  - Implement append-only enforcement (no updates)
  - Add context capture (IP address, user agent, before/after state)
  - Create database indexes for fast queries
  - _Requirements: 4.1, 4.2_

- [ ] 7. Implement NotificationService with queueing
  - Create NotificationService with notify, getNotifications, markAsRead
  - Set up Redis pub/sub for notification queue
  - Implement notification payload serialization by type
  - Add batch notification sending capability
  - Implement notification retention and cleanup
  - _Requirements: 10.1, 10.2_

### Phase 2: API Endpoints

- [ ] 8. Create EngagementController endpoints
  - [ ] 8.1 Implement GET /api/engagement/streaks/:userId
    - Return current_streak, longest_streak, total_wins, win_percentage
    - _Requirements: 1.1_
  
  - [ ] 8.2 Implement GET /api/engagement/achievements (list all)
    - Return all achievements with category and threshold
    - _Requirements: 2.1_
  
  - [ ] 8.3 Implement GET /api/engagement/achievements/:userId (user's earned)
    - Return achievements earned by user with earned_at timestamp
    - _Requirements: 2.1_
  
  - [ ] 8.4 Implement GET /api/engagement/xp (user's current XP and level)
    - Return total_xp, level, xp_progress, last_level_up_at
    - _Requirements: 9.1_

- [ ] 9. Create LeaderboardController endpoints
  - [ ] 9.1 Implement GET /api/engagement/leaderboard
    - Query params: page (default 1), limit (default 100)
    - Return ranked entries with position, user info, stats
    - Add pagination metadata (total_users, has_next)
    - _Requirements: 3.1, 3.2_
  
  - [ ] 9.2 Implement GET /api/engagement/leaderboard/local/:userId
    - Return leaderboard centered on user (±10 positions)
    - _Requirements: 3.1_
  
  - [ ] 9.3 Implement GET /api/engagement/leaderboard/xp
    - Return XP leaderboard with different sorting
    - _Requirements: 9.2_

- [ ] 10. Create ReferralController endpoints
  - [ ] 10.1 Implement GET /api/referrals
    - Return user's referral code and summary stats
    - Return direct_referrals, pending_payouts, paid_payouts
    - _Requirements: 6.1, 6.2_
  
  - [ ] 10.2 Implement GET /api/referrals/payouts
    - Return pending and paid payouts with details
    - _Requirements: 7.1_
  
  - [ ] 10.3 Implement POST /api/referrals/payouts/claim
    - Claim pending payouts and trigger distribution
    - _Requirements: 7.1_

- [ ] 11. Create NotificationController endpoints
  - [ ] 11.1 Implement GET /api/notifications
    - Query params: limit (default 50), unread_only (default false)
    - Return notifications with read status and payload
    - _Requirements: 10.1_
  
  - [ ] 11.2 Implement POST /api/notifications/:id/read
    - Mark notification as read, return updated notification
    - _Requirements: 10.1_
  
  - [ ] 11.3 Implement DELETE /api/notifications/:id
    - Delete notification (soft delete with logical deletion)
    - _Requirements: 10.1_

- [ ] 12. Extend AdminController with audit and dispute endpoints
  - [ ] 12.1 Implement GET /api/admin/audit-log
    - Query params: start_date, end_date, admin_id, action, limit
    - Return filtered audit log entries with pagination
    - _Requirements: 4.1, 4.2_
  
  - [ ] 12.2 Implement GET /api/admin/disputes
    - Query params: status (open, pending, resolved), limit, page
    - Return open disputes with user info and market details
    - _Requirements: 5.1, 5.2_
  
  - [ ] 12.3 Implement PATCH /api/admin/disputes/:id
    - Update dispute status, add admin notes, set final outcome
    - Log action in audit trail
    - _Requirements: 5.1, 5.2_

- [ ] 13. Extend MarketController with category filtering
  - [ ] 13.1 Implement GET /api/categories (list all)
    - Return all categories with name, slug, icon_url, description
    - _Requirements: 8.1_
  
  - [ ] 13.2 Implement GET /api/markets with category filter
    - Add query param: category (slug or id)
    - Filter markets by category_id
    - _Requirements: 8.1_
  
  - [ ] 13.3 Implement category association on market creation
    - Accept category_id in POST /api/markets request
    - Store market-category relationship
    - _Requirements: 8.1_

### Phase 3: Background Jobs & Event Hooks

- [ ] 14. Implement ReferralPayoutJob (nightly distribution)
  - Create cron job running at 02:00 UTC
  - Batch process referrers with pending payouts (100 at a time)
  - Sum payouts by referrer, filter by minimum threshold ($10 USD)
  - Convert USD to XLM using current exchange rate
  - Create Stellar payment transactions via StellarService
  - Mark payouts as paid, store tx_hash
  - Send email confirmations to referrers
  - Log to audit trail
  - _Requirements: 7.1_

- [ ] 15. Implement LeaderboardSnapshotJob (weekly)
  - Create cron job running every Sunday at 00:00 UTC
  - Capture current leaderboard state
  - Insert snapshot records with rank, user stats, date
  - Use for historical rank tracking
  - _Requirements: 3.2_

- [ ] 16. Implement XPCalculationJob (post-market resolution)
  - Hook into market resolution event
  - Award bet placed XP (if not already awarded by tx hash)
  - Award win/loss XP based on bet outcome
  - Check for streak milestones (5, 10, 25 wins)
  - Award streak milestone XP
  - Trigger level-up check
  - _Requirements: 9.1_

- [ ] 17. Implement AchievementAwardJob (post-events)
  - Hook into streak updates
  - Evaluate user for streak achievements (5, 10, 25, 50)
  - Hook into daily job for volume achievements
  - Hook into referral conversion for referral achievements
  - Award achievements and send notifications
  - _Requirements: 2.1, 2.2_

- [ ] 18. Implement NotificationBatchJob (hourly)
  - Process notification queue from Redis
  - Group notifications by user and type (deduplicate)
  - Send batch notifications to in-app system
  - Send email for significant events (achievement, rank up, payout)
  - Mark as sent, handle failures with retry
  - _Requirements: 10.1, 10.2_

- [ ] 19. Hook events into bet result recording
  - After market resolution, trigger StreakService.recordResult
  - Trigger AchievementAwardJob evaluation
  - Trigger XPCalculationJob
  - Trigger LeaderboardService cache invalidation
  - Trigger notification generation
  - _Requirements: 1.1, 2.1, 3.1, 9.1_

### Phase 4: Real-time Features

- [ ] 20. Implement WebSocket broadcasting for leaderboard
  - Create WebSocket handler for rank change events
  - Broadcast rank updates to all connected clients
  - Send achievement earned notifications in real-time
  - Send streak milestone notifications in real-time
  - Handle client subscription/unsubscription
  - _Requirements: 3.1, 3.3_

- [ ] 21. Implement WebSocket broadcasting for achievements
  - Broadcast achievement earned event to user's WebSocket
  - Include achievement details and reward info
  - Handle offline users (store in notification queue)
  - _Requirements: 2.1, 2.3_

- [ ] 22. Implement WebSocket broadcasting for notifications
  - Create notification subscription endpoint
  - Broadcast new notifications to subscribers
  - Support notification filtering by type
  - Handle connection/disconnection gracefully
  - _Requirements: 10.1_

### Phase 5: Admin Features

- [ ] 23. Implement Dispute Resolution Workflow
  - Extend disputes controller with full CRUD
  - Allow admin to override market outcome with evidence
  - Log all changes to audit trail
  - Send notifications to affected users
  - Update bet payouts if outcome changes
  - _Requirements: 5.1, 5.2_

- [ ] 24. Implement Admin Audit Log UI data endpoints
  - Enhance audit log queries with aggregations
  - Add endpoint for audit summary by admin (actions count, types)
  - Add endpoint for change history of specific resource
  - Add export audit log to CSV (date range)
  - _Requirements: 4.1, 4.2_

- [ ] 25. Implement Category Management endpoints
  - [ ] 25.1 Implement POST /api/admin/categories (create)
    - Accept name, slug, sport_type, icon_url, description
    - Validate slug uniqueness
    - _Requirements: 8.1_
  
  - [ ] 25.2 Implement PUT /api/admin/categories/:slug (update)
    - Update category details
    - Log changes to audit trail
    - _Requirements: 8.1_
  
  - [ ] 25.3 Implement DELETE /api/admin/categories/:slug
    - Soft delete category (no markets affected)
    - Log deletion to audit trail
    - _Requirements: 8.1_

### Phase 6: Email & Communication

- [ ] 26. Implement email notifications
  - [ ] 26.1 Add achievement earned email template
    - Include achievement name, description, reward
    - _Requirements: 2.1_
  
  - [ ] 26.2 Add rank change email template
    - Include old rank, new rank, current position
    - _Requirements: 3.1_
  
  - [ ] 26.3 Add referral payout email template
    - Include payout amount, tier breakdown, payment tx
    - _Requirements: 7.1_
  
  - [ ] 26.4 Add level up email template
    - Include new level, total XP, unlocked rewards
    - _Requirements: 9.1_

- [ ] 27. Enhance NotificationService with email delivery
  - Integrate with existing email service
  - Send emails for significant events
  - Track email delivery status
  - Handle email failures with retry
  - _Requirements: 10.1, 10.2_

### Phase 7: Testing & Validation

- [ ] 28. Checkpoint - Ensure core services pass tests
  - Run unit tests for StreakService, AchievementService, XPService
  - Verify streak logic with various win/loss sequences
  - Verify achievement criteria evaluation
  - Verify XP calculations and level progression
  - Ask the user if questions arise.

- [ ] 29. Checkpoint - Ensure API endpoints working
  - Test all endpoints with Postman/curl
  - Verify endpoint responses match design specs
  - Check error handling and validation
  - Verify pagination on list endpoints
  - Ask the user if questions arise.

- [ ] 30. Checkpoint - Ensure background jobs operational
  - Test ReferralPayoutJob in staging
  - Test LeaderboardSnapshotJob
  - Test AchievementAwardJob
  - Test NotificationBatchJob
  - Verify job scheduling and execution times
  - Ask the user if questions arise.

- [ ] 31. Checkpoint - Ensure real-time features working
  - Test WebSocket connections for leaderboard
  - Test achievement broadcast
  - Test notification subscriptions
  - Verify message delivery and ordering
  - Ask the user if questions arise.

- [ ] 32. Checkpoint - Ensure admin features complete
  - Test audit log queries and filtering
  - Test dispute resolution workflow
  - Test category management
  - Verify all admin actions logged
  - Ask the user if questions arise.

- [ ] 33. Final checkpoint - All integration tests passing
  - Run integration test suite
  - Verify end-to-end workflows (bet → streak → achievement → notification)
  - Verify data consistency across features
  - Verify performance meets requirements (< 500ms for leaderboard)
  - Ask the user if questions arise.

## Notes

- All features use existing database infrastructure (PostgreSQL)
- Caching uses Redis (already configured)
- Email delivery uses existing email service
- Admin actions are logged to audit trail automatically
- Services are designed to be independently testable
- Background jobs use node-cron for scheduling
- WebSocket events use existing ws configuration
- Notification payloads are extensible via JSONB columns


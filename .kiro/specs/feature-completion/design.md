# Design Document: Feature Completion

## Overview

This document outlines the detailed design for completing BANKERCHANGER's engagement features. The architecture builds on existing infrastructure (Node.js/Express, PostgreSQL, Redis) and extends with leaderboard caching, real-time notifications, batch job processing, and comprehensive audit logging.

## Architecture

### System Components

```
┌─────────────────────────────────────────────────────────┐
│                    API Layer (Express)                  │
├─────────────────────────────────────────────────────────┤
│ • EngagementController (streaks, achievements, XP)      │
│ • LeaderboardController (rankings, real-time updates)   │
│ • AdminController (audit, disputes, categories)         │
│ • ReferralController (codes, payouts, claims)           │
│ • NotificationController (list, read, subscribe)        │
└─────────────────────────────────────────────────────────┘
         │                                   │
         ▼                                   ▼
┌─────────────────────────┐    ┌──────────────────────────┐
│  Service Layer          │    │  Background Jobs (cron)  │
├─────────────────────────┤    ├──────────────────────────┤
│ • StreakService         │    │ • ReferralPayoutJob      │
│ • AchievementService    │    │ • LeaderboardSnapshotJob │
│ • LeaderboardService    │    │ • XPCalculationJob       │
│ • ReferralService       │    │ • NotificationJob        │
│ • AuditLogService       │    └──────────────────────────┘
│ • XPService             │
│ • NotificationService   │
└─────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────┐
│           Cache & Data Layer                            │
├─────────────────────────────────────────────────────────┤
│ • Redis: leaderboard cache (5min TTL)                   │
│ • Redis: XP aggregations (real-time)                    │
│ • Redis: notification queue                             │
│ • PostgreSQL: all persistent data                       │
└─────────────────────────────────────────────────────────┘
```

## Components and Interfaces

### 1. User Streaks Data Model

**Database Schema (user_streaks table)**
```typescript
interface UserStreak {
  id: number;
  user_id: text;
  current_streak: number;        // Consecutive wins
  longest_streak: number;        // Historical best
  total_wins: number;
  total_losses: number;
  last_result: 'win' | 'loss';   // Last prediction result
  last_resolved_at: timestamp;   // When last market resolved
  updated_at: timestamp;
}
```

**Streak Service Operations**
```typescript
interface StreakService {
  // Record a bet result (win/loss) after market resolution
  recordResult(userId: string, won: boolean): Promise<StreakUpdate>;
  
  // Get user's current streak info
  getStreak(userId: string): Promise<StreakInfo>;
  
  // Get streak statistics for leaderboard
  getStreakStats(userId: string): Promise<StreakStatistics>;
  
  // Get users with active streaks (for notifications)
  getActiveStreaks(minLength: number = 5): Promise<StreakInfo[]>;
  
  // Reset streak on loss (internal)
  _resetStreak(userId: string): Promise<void>;
}

interface StreakUpdate {
  userId: string;
  currentStreak: number;
  longestStreak: number;
  won: boolean;
  streakBreak?: boolean;
  achievementUnlocked?: string;
}

interface StreakInfo {
  userId: string;
  currentStreak: number;
  longestStreak: number;
  totalWins: number;
  totalLosses: number;
  lastResult: 'win' | 'loss';
  lastResolvedAt: Date;
  winPercentage: number;
}
```

**Calculation Logic**
- On bet win: `current_streak++`, check if `longest_streak` needs update
- On bet loss: Store `current_streak` to history, reset `current_streak = 0`
- Win percentage: `(total_wins / (total_wins + total_losses)) * 100`
- Streak milestone: Achievements trigger at 5, 10, 25, 50 consecutive wins

---

### 2. Achievements System

**Database Schema (achievements table)**
```typescript
interface Achievement {
  id: number;
  code: string;              // Unique: 'first_bet', 'streak_5', 'volume_1000'
  name: string;              // Display name
  description: string;       // What to do to earn
  category: string;          // 'streak' | 'volume' | 'referral' | 'general'
  criteria: {                // Rules for awarding
    type: 'streak' | 'volume' | 'xp' | 'referral';
    threshold: number;
    condition?: string;
  };
  created_at: timestamp;
}

interface UserAchievement {
  id: number;
  user_id: string;
  achievement_code: string;
  awarded_at: timestamp;
}
```

**Achievement Triggers**
```typescript
enum AchievementType {
  FIRST_BET = 'first_bet',           // Place any bet
  STREAK_5 = 'streak_5',             // 5 consecutive wins
  STREAK_10 = 'streak_10',           // 10 consecutive wins
  STREAK_25 = 'streak_25',           // 25 consecutive wins
  VOLUME_1K = 'volume_1k',           // $1000 total bet volume
  VOLUME_10K = 'volume_10k',         // $10,000 total bet volume
  VOLUME_100K = 'volume_100k',       // $100,000 total bet volume
  LEVEL_5 = 'level_5',               // Reach XP level 5
  LEVEL_10 = 'level_10',             // Reach XP level 10
  REFERRAL_3 = 'referral_3',         // Refer 3 active users
  REFERRAL_10 = 'referral_10',       // Refer 10 active users
}

interface AchievementService {
  // Check if user qualifies for any new achievements
  evaluateUserForAchievements(userId: string): Promise<Achievement[]>;
  
  // Award achievement (idempotent)
  awardAchievement(userId: string, code: string): Promise<UserAchievement>;
  
  // Get all achievements
  listAchievements(): Promise<Achievement[]>;
  
  // Get user's earned achievements
  getUserAchievements(userId: string): Promise<UserAchievement[]>;
  
  // Get remaining achievements for user
  getRemainingAchievements(userId: string): Promise<Achievement[]>;
}
```

**Award Trigger Points**
- After market resolution → check streak achievements
- Daily → check volume and referral achievements
- On XP level up → check level achievements
- On user registration with referral code → check referral achievements

---

### 3. Leaderboard Algorithm & Real-time Updates

**Leaderboard Ranking**
```typescript
interface LeaderboardEntry {
  rank: number;
  userId: string;
  displayName: string;
  totalWins: number;
  totalLosses: number;
  winPercentage: number;
  currentStreak: number;
  longestStreak: number;
  totalVolume: number;
  xpTotal: number;
  level: number;
  avatarUrl: string;
}

// Ranking Algorithm (priority order):
// 1. Sort by win_percentage DESC (primary)
// 2. Then by total_wins DESC (secondary)
// 3. Then by total_volume DESC (tertiary)
// 4. Then by updated_at ASC (newest changes rank higher)
```

**Caching Strategy**
```typescript
interface LeaderboardService {
  // Get leaderboard page (cached for 5 minutes)
  getLeaderboard(page: number = 1, limit: number = 100): Promise<LeaderboardPage>;
  
  // Get user's current rank (always fresh)
  getUserRank(userId: string): Promise<number>;
  
  // Get users near a specific user (±10 positions)
  getLocalLeaderboard(userId: string, range: number = 10): Promise<LeaderboardEntry[]>;
  
  // Invalidate cache (called after score changes)
  invalidateCache(): Promise<void>;
  
  // Refresh leaderboard from database
  _refreshLeaderboard(): Promise<void>;
}

// Redis Cache Structure:
// Key: "leaderboard:page:{pageNum}"
// Value: JSON array of LeaderboardEntry
// TTL: 5 minutes
// On score change: delete "leaderboard:*" pattern

interface RankUpdate {
  userId: string;
  oldRank: number;
  newRank: number;
  timestamp: Date;
}
```

**Real-time Update via WebSocket**
```typescript
interface WebSocketMessage {
  type: 'rank_change' | 'leaderboard_update' | 'achievement_earned';
  payload: {
    userId: string;
    previousRank?: number;
    newRank?: number;
    achievement?: {
      code: string;
      name: string;
    };
  };
  timestamp: Date;
}

// Broadcast rank changes when:
// - Win percentage changes
// - Total wins increases
// - New achievement earned
// - Total volume surpasses threshold
```

**Weekly Snapshot**
```typescript
interface LeaderboardSnapshot {
  id: number;
  snapshot_date: date;
  rank: number;
  user_id: string;
  win_percentage: number;
  total_wins: number;
  total_volume: number;
  created_at: timestamp;
}

// Create snapshot every Sunday at 00:00 UTC
// Preserve historical rankings for comparison
```

---

### 4. Admin Audit Trail Schema & Queries

**Audit Log Schema**
```typescript
interface AdminAuditLog {
  id: number;
  admin_id: string;           // Admin performing action
  action: string;             // Action code
  target_id: string;          // Resource affected (market_id, user_id, dispute_id)
  before_state: JSONB;        // Data before change
  after_state: JSONB;         // Data after change
  details: JSONB;             // Additional context
  created_at: timestamp;
  ip_address?: string;
  user_agent?: string;
}

enum AuditAction {
  MARKET_CREATE = 'market_create',
  MARKET_UPDATE = 'market_update',
  MARKET_RESOLVE = 'market_resolve',
  DISPUTE_FILED = 'dispute_filed',
  DISPUTE_RESOLVED = 'dispute_resolved',
  USER_BANNED = 'user_banned',
  USER_UNBANNED = 'user_unbanned',
  CONTENT_REMOVED = 'content_removed',
  CATEGORY_CREATED = 'category_created',
  CATEGORY_UPDATED = 'category_updated',
}
```

**Audit Service Interface**
```typescript
interface AuditLogService {
  // Log an admin action (immutable)
  log(entry: {
    adminId: string;
    action: AuditAction;
    targetId: string;
    beforeState?: unknown;
    afterState?: unknown;
    details?: unknown;
    ipAddress?: string;
    userAgent?: string;
  }): Promise<void>;
  
  // Query audit logs with filters
  query(filters: {
    startDate?: Date;
    endDate?: Date;
    adminId?: string;
    action?: AuditAction;
    targetId?: string;
    limit?: number;
  }): Promise<AdminAuditLog[]>;
  
  // Get summary for admin
  getSummary(adminId: string, daysBack?: number): Promise<ActionSummary>;
}

interface ActionSummary {
  totalActions: number;
  byType: Map<AuditAction, number>;
  lastAction?: AdminAuditLog;
}
```

**Query Indexes**
- `(admin_id, created_at)` - Find actions by admin
- `(action, created_at)` - Find actions by type
- `(target_id, created_at)` - Find all changes to resource
- `(created_at)` - Retention queries for archival

**Data Retention**
- Keep for 2 years
- Immutable (no updates, only inserts)
- Quarterly archival to separate table for compliance

---

### 5. Referral Program: Payout Calculation & Distribution

**Referral Payout Schema**
```typescript
interface ReferralRecord {
  id: number;
  referrer_id: string;
  referred_id: string;
  referral_code: string;
  status: 'pending' | 'active';        // pending = not yet converted
  bonus_rate_bps: number;              // Basis points (500 = 5%)
  created_at: timestamp;
  converted_at?: timestamp;
}

interface ReferralPayout {
  id: number;
  referrer_id: string;
  referred_id: string;
  tier: number;                        // 1 = direct, 2 = indirect
  source_amount: numeric;              // Base bet amount
  payout_amount: numeric;              // Calculated payout
  rate_bps: number;                    // Applied rate (in basis points)
  status: 'pending' | 'paid';
  tx_hash?: string;                    // Stellar tx for proof
  created_at: timestamp;
  paid_at?: timestamp;
}
```

**Multi-level Payout Structure**
```typescript
const REFERRAL_RATES = {
  tier1: 500,   // 5% on direct referral's volume
  tier2: 250,   // 2.5% on tier1 referral's volume
  tier3: 125,   // 1.25% on tier2 referral's volume
};

const MINIMUM_PAYOUT = 10; // USD equivalent (≈50 XLM at $0.20/XLM)

interface ReferralService {
  // Calculate payouts for a referred user's bet
  calculatePayoutsForBet(
    referredUserId: string,
    betAmount: number,
    marketId: string
  ): Promise<PayoutCalculation[]>;
  
  interface PayoutCalculation {
    tier: number;
    referrerId: string;
    payoutAmount: number;
    rateBps: number;
  }
  
  // Record payout in pending state
  recordPendingPayout(payout: PayoutCalculation): Promise<ReferralPayout>;
  
  // Calculate and batch payouts (nightly job)
  calculateNightlyPayouts(): Promise<void>;
  
  // Distribute payouts to Stellar wallets
  distributePayouts(minAmount?: number): Promise<DistributionResult>;
  
  // Get referral earnings summary
  getReferralSummary(referrerId: string): Promise<{
    referralCode: string;
    directReferrals: number;
    tier1Payouts: number;
    tier2Payouts: number;
    tier3Payouts: number;
    totalEarnings: number;
    pendingPayouts: number;
    paidPayouts: number;
  }>;
}
```

**Payout Distribution Job**
```typescript
interface PayoutDistributionJob {
  // Runs nightly at 02:00 UTC
  // 1. Sum all pending payouts by referrer
  // 2. Filter those >= MINIMUM_PAYOUT
  // 3. Convert USD amount to XLM
  // 4. Create Stellar payment transactions
  // 5. Mark as paid
  // 6. Email confirmations to referrers
  
  batchSize: 100;           // Process 100 referrers per batch
  maxRetries: 3;
  retryDelayMs: 5000;
}
```

---

### 6. XP System Progression Mechanics

**XP Events Schema**
```typescript
interface XPEvent {
  id: number;
  transaction_hash: string;  // De-duplicate by tx
  event_type: XPEventType;
  user_id: string;
  points: number;
  market_id?: string;
  created_at: timestamp;
}

enum XPEventType {
  BET_PLACED = 'bet_placed',           // +10 XP
  BET_WON = 'bet_won',                 // +50 XP
  BET_LOST = 'bet_lost',               // +0 XP
  STREAK_5 = 'streak_5',               // +100 XP
  STREAK_10 = 'streak_10',             // +250 XP
  ACHIEVEMENT_EARNED = 'achievement_earned',  // +50 XP
  REFERRAL_CONVERTED = 'referral_converted',  // +25 XP
}

interface UserLevel {
  user_id: string;
  total_xp: number;
  level: number;              // Calculated: total_xp / 1000
  xp_progress: number;        // Remaining XP to next level
  last_level_up?: timestamp;
}
```

**XP Calculation**
```typescript
// Level progression (1000 XP per level)
const getLevel = (totalXp: number): number => Math.floor(totalXp / 1000) + 1;
const getXpForLevel = (level: number): number => (level - 1) * 1000;
const getXpProgress = (totalXp: number): number => totalXp % 1000;

interface XPService {
  // Award XP on event
  awardXP(userId: string, type: XPEventType, amount: number, txHash: string): Promise<UserLevel>;
  
  // Get user's current level
  getLevel(userId: string): Promise<UserLevel>;
  
  // Get leaderboard by XP
  getXPLeaderboard(page: number = 1, limit: number = 100): Promise<XPLeaderboardEntry[]>;
  
  // Check for level-up achievements
  checkLevelMilestones(userId: string): Promise<string[]>; // achievement codes
}

interface XPLeaderboardEntry {
  rank: number;
  userId: string;
  displayName: string;
  totalXp: number;
  level: number;
  lastLevelUpAt: Date;
}
```

---

### 7. Notification Types & Delivery Strategy

**Notification Schema**
```typescript
interface UserNotification {
  id: number;
  user_id: string;
  type: NotificationType;
  title: string;
  body: string;
  payload: JSONB;            // Event-specific data
  read: boolean;
  created_at: timestamp;
}

enum NotificationType {
  STREAK_MILESTONE = 'streak_milestone',       // 5, 10, 25 wins
  ACHIEVEMENT_EARNED = 'achievement_earned',
  REFERRAL_CONVERTED = 'referral_converted',
  REFERRAL_PAYOUT = 'referral_payout',
  LEADERBOARD_RANK_CHANGE = 'rank_change',
  LEVEL_UP = 'level_up',
  MARKET_RESOLVED = 'market_resolved',
  BET_WON = 'bet_won',
  DISPUTE_RESOLVED = 'dispute_resolved',
}

// Notification Payloads
interface StreakNotification {
  streak: number;
  achievementUnlocked: boolean;
  nextMilestone: number;
}

interface AchievementNotification {
  achievementCode: string;
  achievementName: string;
  reward?: string;
}

interface RankChangeNotification {
  oldRank: number;
  newRank: number;
  direction: 'up' | 'down';
}

interface LevelUpNotification {
  newLevel: number;
  totalXp: number;
  rewards: string[];
}
```

**Notification Service**
```typescript
interface NotificationService {
  // Create notification (queued)
  notify(
    userId: string,
    type: NotificationType,
    title: string,
    body: string,
    payload: unknown
  ): Promise<void>;
  
  // Get user's notifications
  getNotifications(userId: string, limit: number = 50): Promise<UserNotification[]>;
  
  // Mark as read
  markAsRead(userId: string, notificationId: number): Promise<void>;
  
  // Delete notification
  delete(notificationId: number): Promise<void>;
  
  // Batch send (for background jobs)
  sendBatch(notifications: NotificationRecord[]): Promise<void>;
}

interface NotificationRecord {
  userId: string;
  type: NotificationType;
  title: string;
  body: string;
  payload: unknown;
}
```

**Delivery Channels**
```
┌─────────────────────────────────────────┐
│  In-App Notifications                   │
│  • Stored in user_notifications table    │
│  • Fetched via REST API                  │
│  • Real-time delivery via WebSocket      │
└─────────────────────────────────────────┘
                   │
                   ▼
         ┌─────────────────────┐
         │ Redis Notification  │
         │ Queue               │
         │ (Pub/Sub)           │
         └─────────────────────┘
           │                 │
           ▼                 ▼
    ┌──────────────┐  ┌──────────────┐
    │ Email Svc    │  │ WebSocket    │
    │ (SignalR or  │  │ Broadcast    │
    │  async mail) │  │              │
    └──────────────┘  └──────────────┘
```

---

## Data Models

### Complete Entity Relationship Diagram

```
Users (existing)
├── user_streaks
├── user_achievements
├── user_notifications
├── user_xp_events
├── referrals (as referrer_id)
└── referral_payouts (as referrer_id)

Markets (existing)
├── bets (existing)
├── market_categories (new)
└── leaderboard_snapshots (new)

Achievements
├── user_achievements (earned_by)
└── achievement_triggers (rules)

Admin
├── admin_audit_log
└── disputes (existing)
```

### Key Relationships

- **User → UserStreak**: 1:1 relationship
- **User → UserAchievement**: 1:M (many achievements earned)
- **Achievement → UserAchievement**: 1:M (earned by many users)
- **User → ReferralPayout**: 1:M (tier 1, 2, 3 payouts)
- **User → UserNotification**: 1:M (notifications received)
- **Market → Category**: M:1 (one category per market)
- **Admin → AdminAuditLog**: 1:M (admin actions)

## Error Handling

**Streak Calculation Errors**
- Concurrent updates: Use `SELECT FOR UPDATE` to lock row during calculation
- Market resolution delay: Cache streak until resolution confirmed
- Data inconsistency: Recalculate on read if mismatch detected

**Achievement Award Errors**
- Duplicate awards: Idempotent operation using unique constraint
- Missing criteria: Fail silently and retry in daily job
- Notification failure: Decouple from award transaction

**Leaderboard Errors**
- Cache miss: Query database and rebuild cache (max 2 seconds)
- Rank tie-breaking: Use timestamp ordering (first wins)
- Real-time lag: Accept eventual consistency (5-min delay acceptable)

**Payout Errors**
- Minimum threshold: Queue payouts and retry next nightly job
- Stellar tx failure: Retry with exponential backoff, max 3 retries
- Duplicate payout: Hash (referrer_id, referred_id, amount) for idempotence

**Notification Errors**
- Queue overflow: Prioritize achievement/rank notifications
- Email delivery failure: Retry up to 3 times, log for manual review
- User not found: Discard notification after 7 days

## Testing Strategy

### Unit Tests
- Streak calculation logic (win/loss sequences)
- Achievement criteria evaluation (thresholds)
- Payout calculation (multi-level rates)
- XP progression (level calculations)
- Notification formatting

### Integration Tests
- End-to-end market resolution → streak update → achievement check
- Referral conversion → payout calculation → distribution
- Admin audit logging during dispute resolution
- Real-time leaderboard updates via WebSocket
- Category filtering on market queries

### Load Tests
- Leaderboard query performance (< 500ms for 100k users)
- Streak updates during concurrent bet resolution
- Nightly payout job with 50k referrers
- WebSocket broadcast to 1000+ concurrent users

### Data Consistency Tests
- Audit log immutability
- Referral payout idempotence
- Streak calculation consistency across db replicas
- XP event de-duplication by transaction hash


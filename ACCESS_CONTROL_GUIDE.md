# Access Control & Player Management Guide

**Date**: November 2, 2025  
**System**: Infinity Storm Casino Server

---

## 📋 Overview

Your casino system uses a **Player Status System** instead of traditional whitelist/blacklist. This provides more flexibility with statuses: `active`, `suspended`, and `banned`.

---

## 🎯 Player Status System

### Status Types

| Status | Description | Can Login? | Can Play? | Can Transact? |
|--------|-------------|------------|-----------|---------------|
| **active** | Normal player | ✅ Yes | ✅ Yes | ✅ Yes |
| **suspended** | Temporarily restricted | ✅ Yes | ❌ No | ❌ No |
| **banned** | Permanently blocked | ❌ No | ❌ No | ❌ No |

---

## 🔧 How to Manage Players

### Via Admin Panel (Recommended)

**Access**: `http://127.0.0.1:3000/admin`

#### 1. Suspend a Player
**Endpoint**: `POST /admin/players/:id/suspend`

**Use When**: 
- Suspicious activity detected
- Temporary account freeze pending investigation
- Cooling-off period for problem gambling

**Effect**:
- Player can still login
- Cannot place bets or spin
- Cannot withdraw funds
- Audit log created

**How To**:
```bash
# Via API
POST http://127.0.0.1:3000/admin/players/{player_id}/suspend
Headers:
  Authorization: Bearer {admin_token}
  Content-Type: application/json
Body:
{
  "reason": "Suspicious betting patterns detected"
}
```

**Via Admin Panel**:
1. Login to admin panel: `http://127.0.0.1:3000/admin/login`
2. Navigate to Players section
3. Find player by username/ID
4. Click "Suspend Account"
5. Enter suspension reason
6. Confirm

---

#### 2. Ban a Player (Permanent)
**Endpoint**: `POST /admin/players/:id/ban`

**Use When**:
- Terms of service violation
- Fraud/cheating confirmed
- Multiple account abuse
- Underage gambling

**Effect**:
- Player cannot login
- All sessions terminated
- Cannot create new accounts (if IP tracked)
- Permanent audit log entry

**How To**:
```bash
# Via API
POST http://127.0.0.1:3000/admin/players/{player_id}/ban
Headers:
  Authorization: Bearer {admin_token}
  Content-Type: application/json
Body:
{
  "reason": "Terms of service violation - multiple accounts"
}
```

**Via Admin Panel**:
1. Login to admin panel
2. Navigate to Players section
3. Find player
4. Click "Ban Account"
5. Enter ban reason (required for audit)
6. Confirm (irreversible without admin action)

---

#### 3. Reactivate a Player
**Endpoint**: `PUT /admin/players/:id/activate`

**Use When**:
- Investigation cleared
- Suspension period ended
- Appeal approved
- Temporary ban lifted

**Effect**:
- Player status set to `active`
- Can login and play again
- Full account access restored
- Reactivation logged

**How To**:
```bash
# Via API
PUT http://127.0.0.1:3000/admin/players/{player_id}/activate
Headers:
  Authorization: Bearer {admin_token}
```

**Via Admin Panel**:
1. Login to admin panel
2. Navigate to Players section
3. Filter by "Suspended" or "Banned" status
4. Select player
5. Click "Activate Account"
6. Confirm

---

### Via Database (Direct Access)

**Use With Caution!** Bypasses audit logging.

#### Check Player Status
```sql
SELECT id, username, status, is_demo, credits, created_at 
FROM players 
WHERE username = 'suspicious_player';
```

#### Suspend Player
```sql
UPDATE players 
SET status = 'suspended', updated_at = NOW() 
WHERE username = 'player_to_suspend';
```

#### Ban Player
```sql
UPDATE players 
SET status = 'banned', updated_at = NOW() 
WHERE username = 'player_to_ban';
```

#### Reactivate Player
```sql
UPDATE players 
SET status = 'active', updated_at = NOW() 
WHERE username = 'player_to_reactivate';
```

#### View All Suspended/Banned Players
```sql
SELECT username, status, credits, created_at, updated_at 
FROM players 
WHERE status IN ('suspended', 'banned') 
ORDER BY updated_at DESC;
```

---

## 🚫 Rate Limiting (IP-Based Throttling)

Your system has **NO traditional IP whitelist/blacklist**, but uses sophisticated **rate limiting** instead.

### Rate Limits by Endpoint

| Endpoint Type | Window | Max Requests | Applies To |
|--------------|--------|--------------|------------|
| **Global** | 15 min | 100 | All endpoints |
| **API** | 1 min | 10 | /api/* endpoints |
| **Spin** | 1 min | 10 | /api/spin |
| **Demo Spin** | 1 min | 30 | /api/demo-spin |
| **Authentication** | 15 min | 5 | /auth/login |
| **Wallet Operations** | 1 min | 30 | Wallet endpoints |

### How Rate Limiting Works

**By IP Address**:
- Tracks requests per IP
- Blocks excessive requests
- Returns 429 (Too Many Requests)
- Auto-resets after time window

**By Player ID** (when authenticated):
- Tracks per player
- Prevents automation/bots
- More accurate than IP alone
- Logged in admin_logs

---

### Rate Limit Configuration

**File**: `src/middleware/security.js`

**Development vs Production**:
```javascript
// Development: Very permissive (for testing)
max: IS_PRODUCTION ? 100 : 10000

// Production: Strict limits
max: IS_PRODUCTION ? 10 : 1000
```

**To Adjust Limits**:
```javascript
// src/middleware/security.js

// Change global rate limit
const globalRateLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,  // 15 minutes
  max: 200,  // Increase from 100 to 200
  // ...
});

// Change spin rate limit
const spinRateLimiter = rateLimit({
  windowMs: 1 * 60 * 1000,  // 1 minute
  max: 20,  // Increase from 10 to 20
  // ...
});
```

**Restart Required**: Yes, after modifying rate limits.

---

## 📊 Monitoring Access Control

### 1. View Suspended/Banned Players

**Database Query**:
```sql
SELECT 
  username, 
  status, 
  credits, 
  created_at, 
  updated_at,
  CASE 
    WHEN status = 'active' THEN '✅ Active'
    WHEN status = 'suspended' THEN '⚠️ Suspended'
    WHEN status = 'banned' THEN '🚫 Banned'
  END as display_status
FROM players 
WHERE status != 'active'
ORDER BY updated_at DESC;
```

---

### 2. View Admin Actions (Audit Trail)

**Recent Suspensions/Bans**:
```sql
SELECT 
  al.action_type,
  al.created_at,
  admin.username as admin_username,
  target.username as target_username,
  al.details->'reason' as reason,
  al.severity,
  al.result
FROM admin_logs al
LEFT JOIN players admin ON al.admin_id = admin.id
LEFT JOIN players target ON al.target_player_id = target.id
WHERE al.action_type IN ('account_suspension', 'account_ban', 'account_activation')
ORDER BY al.created_at DESC
LIMIT 20;
```

---

### 3. Check Rate Limit Violations

**View in Logs**:
```bash
# Check server logs for rate limit hits
cd infinity-storm-server
cat logs/combined.log | grep "rate limit"
```

**Recent Rate Limit Violations**:
```sql
-- If you're logging rate limits to admin_logs
SELECT 
  action_type,
  ip_address,
  created_at,
  details
FROM admin_logs
WHERE action_type = 'security_event'
  AND details->>'type' = 'rate_limit_exceeded'
ORDER BY created_at DESC
LIMIT 20;
```

---

## 🔐 Admin Access Requirements

### To Suspend/Ban Players:

1. **Admin Account Required**
   - Field: `is_admin = true` in `players` table
   - Field: `status = 'active'`

2. **Admin Authentication**
   - Login at `/admin/login`
   - JWT token with admin privileges
   - Session timeout: 1 hour (configurable)

3. **Rate Limiting**
   - Suspend/Ban operations: 10 per 5 minutes
   - Protects against accidental mass actions

---

## 🚀 Quick Access Commands

### Check if Player is Banned/Suspended
```bash
# Via Database
docker exec -i supabase_db_infinity-storm-server psql -U postgres -d postgres \
  -c "SELECT username, status FROM players WHERE username = 'playername';"
```

### List All Problem Players
```bash
docker exec -i supabase_db_infinity-storm-server psql -U postgres -d postgres \
  -c "SELECT username, status, updated_at FROM players WHERE status != 'active' ORDER BY updated_at DESC;"
```

### Count Players by Status
```bash
docker exec -i supabase_db_infinity-storm-server psql -U postgres -d postgres \
  -c "SELECT status, COUNT(*) as count FROM players GROUP BY status;"
```

---

## ⚙️ Player Model Methods

**File**: `src/models/Player.js`

### Available Methods

```javascript
// Suspend a player
await player.suspend('Reason for suspension');

// Ban a player
await player.ban('Reason for ban');

// Reactivate a player
await player.reactivate();

// Check if player can play
player.isActive();  // Returns true if status === 'active'

// Check if player can place a bet
player.canPlaceBet(betAmount);  // Checks status and credits
```

### Usage Example

```javascript
const { Player } = require('./src/models');

// Find player
const player = await Player.findOne({ 
  where: { username: 'suspicious_player' } 
});

// Suspend player
if (player) {
  await player.suspend('Multiple failed authentication attempts');
  console.log(`✅ Player ${player.username} suspended`);
}

// Ban player
if (player) {
  await player.ban('Terms of service violation');
  console.log(`🚫 Player ${player.username} banned permanently`);
}

// Reactivate player
if (player) {
  await player.reactivate();
  console.log(`✅ Player ${player.username} reactivated`);
}
```

---

## 📋 Admin Panel Routes

| Action | Method | Endpoint | Authentication |
|--------|--------|----------|----------------|
| **View Players** | GET | `/admin/players` | Required |
| **View Player Details** | GET | `/admin/players/:id` | Required |
| **Suspend Player** | POST | `/admin/players/:id/suspend` | Required + Rate Limited |
| **Ban Player** | POST | `/admin/players/:id/ban` | Required + Rate Limited |
| **Activate Player** | PUT | `/admin/players/:id/activate` | Required |
| **View Admin Logs** | GET | `/admin/logs` | Required |

---

## 🛡️ Security Best Practices

### 1. Always Log Actions
✅ Use admin panel (auto-logs)  
❌ Don't use direct DB updates (no audit trail)

### 2. Provide Reasons
✅ Include detailed reason for ban/suspension  
❌ Don't ban without documenting why

### 3. Review Regularly
- Check `admin_logs` weekly for patterns
- Review suspended players monthly
- Investigate multiple bans from same admin

### 4. Separation of Duties
- Different admins for different actions
- Require approval for permanent bans
- Log all admin access

---

## 🎯 Common Scenarios

### Scenario 1: Detected Bot Activity

**Steps**:
1. Check player's spin patterns in `spin_results`
2. Review `admin_logs` for rate limit violations
3. Suspend account immediately
4. Investigate spin timing, bet patterns
5. If confirmed: Ban account
6. If false alarm: Reactivate with apology

**Commands**:
```sql
-- Check spin patterns
SELECT 
  COUNT(*) as total_spins,
  MIN(EXTRACT(EPOCH FROM (created_at - LAG(created_at) OVER (ORDER BY created_at)))) as min_seconds_between_spins,
  AVG(EXTRACT(EPOCH FROM (created_at - LAG(created_at) OVER (ORDER BY created_at)))) as avg_seconds_between_spins
FROM spin_results
WHERE player_id = 'player_uuid'
  AND created_at > NOW() - INTERVAL '1 hour';
```

---

### Scenario 2: Multiple Account Abuse

**Steps**:
1. Check if same IP has multiple accounts
2. Review transaction patterns
3. Ban all related accounts
4. Flag IP in monitoring

**Commands**:
```sql
-- Find accounts from same IP (requires IP tracking)
SELECT DISTINCT player_id, COUNT(*) as login_count
FROM sessions
WHERE ip_address = '192.168.1.1'
GROUP BY player_id;
```

---

### Scenario 3: Player Requests Account Closure

**Steps**:
1. Suspend account (not ban - allows future recovery)
2. Withdraw remaining balance
3. Log closure reason
4. Mark in notes

**Commands**:
```javascript
await player.suspend('Self-exclusion requested by player');
// Process withdrawal
// Log in admin_logs
```

---

## 📖 Summary

### Your Access Control System:

✅ **Player Status**: active, suspended, banned  
✅ **Admin Panel**: Full UI for player management  
✅ **Rate Limiting**: IP-based throttling (no hard blacklist)  
✅ **Audit Logging**: All actions tracked in `admin_logs`  
✅ **API Access**: RESTful endpoints for automation  
✅ **Database Access**: Direct SQL for emergencies  

### No Traditional Whitelist/Blacklist:

❌ No IP whitelist table  
❌ No IP blacklist table  
✅ Instead: Sophisticated rate limiting  
✅ Instead: Player status system  
✅ Better: More flexible and auditable  

---

## 🚀 Quick Reference

### Suspend Player
```bash
POST /admin/players/{id}/suspend
Body: {"reason": "Suspicious activity"}
```

### Ban Player
```bash
POST /admin/players/{id}/ban
Body: {"reason": "ToS violation"}
```

### Reactivate Player
```bash
PUT /admin/players/{id}/activate
```

### Check Player Status
```sql
SELECT username, status FROM players WHERE username = 'player';
```

### View Recent Admin Actions
```sql
SELECT * FROM admin_logs 
WHERE action_type IN ('account_suspension', 'account_ban') 
ORDER BY created_at DESC LIMIT 10;
```

---

**Your access control system is production-ready with comprehensive auditing!** 🎰✨

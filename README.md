# Subscription & Payment System Documentation

This document provides a comprehensive guide to the subscription and payment system, including all edge cases, webhook handling, and state management logic.

## Table of Contents

1. [Subscription States](#subscription-states)
2. [Plan Hierarchy](#plan-hierarchy)
3. [Upgrade Flow](#upgrade-flow)
4. [Downgrade Flow](#downgrade-flow)
5. [Trial Management](#trial-management)
6. [Webhook Processing](#webhook-processing)
7. [Credit Webhook Detection](#credit-webhook-detection)
8. [Scheduled Downgrades](#scheduled-downgrades)
9. [State Machine Logic](#state-machine-logic)
10. [Edge Cases & Solutions](#edge-cases--solutions)

---

## Subscription States

The system supports the following subscription states:

- **`active`**: Active paid subscription
- **`trialing`**: User is on a trial period (no charge)
- **`cancelled_at_period_end`**: Subscription is cancelled but remains active until period ends
- **`free`**: No active subscription (revoked or never subscribed)
- **`expired`**: Subscription has expired

### State Transitions

```
free → trialing → active (trial conversion)
free → active (direct subscription)
active → cancelled_at_period_end → free (cancellation)
active → active (plan change)
trialing → free (trial revoked)
```

---

## Plan Hierarchy

Plans are ordered by tier for upgrade/downgrade detection:

```javascript
const PLAN_HIERARCHY = {
  free: 0,
  pro: 1,
  plus: 2,
  agency: 3
};
```

**Key Rules:**
- Higher tier = more expensive plan
- Upgrades: `targetTier > currentTier`
- Downgrades: `targetTier < currentTier`
- Same tier: Interval change (monthly ↔ yearly)

---

## Upgrade Flow

### Paid-to-Paid Upgrade (e.g., Pro → Plus)

**Immediate Upgrade with Proration:**
1. User requests upgrade via `/api/subscription/upgrade`
2. System calls Polar API to update subscription immediately
3. Polar applies proration (charges for new plan, credits for old plan)
4. Webhooks arrive:
   - `subscription.updated` → Updates subscription details
   - `order.paid` (charge) → New plan charge → **Updates plan to Plus**
   - `order.paid` (credit) → Old plan refund → **Must be ignored** (see Credit Detection)

**Key Points:**
- Upgrade happens immediately
- Proration is handled by Polar
- Credit webhook must NOT downgrade the plan

### Trial → Paid Upgrade

**Different Plan:**
1. User on trial (e.g., Pro trial) upgrades to different plan (e.g., Plus)
2. System revokes the trial subscription
3. Sets user to `free` status
4. Creates new subscription via checkout
5. User completes checkout → New paid subscription created

**Same Plan:**
- User cannot "upgrade" to the same plan they're trialing
- System returns error: "You are already on this plan. Your trial will automatically convert to paid when it ends."

---

## Downgrade Flow

### Paid-to-Paid Downgrade (e.g., Plus → Pro)

**Scheduled Downgrade (No Proration):**
1. User requests downgrade via `/api/subscription/upgrade`
2. System does NOT call Polar API immediately
3. System stores `nextPlan` in database
4. System schedules a job to run at `currentPeriodEnd`
5. User keeps current plan until period ends
6. At period end, scheduled job:
   - Updates Polar subscription to new plan
   - Updates local database
   - Clears `nextPlan`

**Key Points:**
- No immediate plan change
- No proration (user keeps current plan until period ends)
- Downgrade is deferred to end of billing period

### Free Plan (Revocation)

- "Free" means revoking the subscription
- Happens immediately (no scheduling)
- Sets `polarSubscriptionId` to `null`
- Sets status to `free`

---

## Trial Management

### Trial States

**Active Trial:**
- Status: `trialing`
- Price: `0`
- `trialingEndsAt`: End date of trial
- `polarSubscriptionId`: Present

**Cancelled Trial:**
- Status: `cancelled_at_period_end` (if before trial ends)
- Status: `free` (if after trial ends)
- `trialingEndsAt`: Preserved if trial hasn't ended

### Trial Conversion

**Natural Conversion:**
- Trial ends → `subscription.updated` webhook
- Status changes: `trialing` → `active`
- Price updates to plan price
- `trialingEndsAt` cleared

**Trial Cancellation:**
- User cancels during trial
- Status: `cancelled_at_period_end`
- Trial continues until `trialingEndsAt`
- After trial ends → Status: `free`

**Trial Resume:**
- User cancels trial, then resumes before trial ends
- Status must restore to `trialing` (not `active`)
- `trialingEndsAt` preserved
- Price: `0`

**Trial Plan Change:**
- User on trial changes to different plan
- Trial is revoked immediately
- User set to `free`
- New subscription created via checkout

---

## Webhook Processing

### Webhook Event Types

The system processes the following Polar.sh webhooks:

1. **`order.paid`**: Payment recorded
2. **`subscription.created`**: New subscription created
3. **`subscription.updated`**: Subscription updated
4. **`subscription.active`**: Subscription activated
5. **`subscription.canceled`**: Subscription cancelled (at period end)
6. **`subscription.uncanceled`**: Cancellation reversed
7. **`subscription.revoked`**: Subscription revoked

### Webhook Processing Flow

```
1. Verify webhook signature
2. Check idempotency (prevent duplicate processing)
3. Extract user ID from metadata or customer lookup
4. Get current subscription state from database
5. Build next state using state machine
6. Validate state invariants
7. Update database atomically
8. Mark webhook as processed
```

### State Machine

The `buildNextSubscriptionState()` function is the single source of truth for state transitions. It takes:
- Current state (from database)
- Event type (webhook type)
- Event data (webhook payload)
- Derived values (plan, price, interval, etc.)

And returns the next valid state.

---

## Credit Webhook Detection

### Problem

When upgrading (e.g., Pro → Plus), Polar sends multiple `order.paid` webhooks:
1. **Charge webhook**: New plan (Plus) with positive amount
2. **Credit webhook**: Old plan (Pro) with negative amount OR positive amount for refund

Without detection, the credit webhook would incorrectly downgrade the plan back to Pro.

### Solution

Credit webhooks are detected using two methods:

```javascript
const isCredit = 
  (webhookPrice !== null && webhookPrice < 0) ||  // Negative amount
  (isDowngrade && existingPlanName !== null);     // Downgrade webhook
```

**Detection Logic:**
1. **Negative Amount**: `webhookPrice < 0` → Credit/refund
2. **Downgrade Webhook**: Incoming plan tier < current plan tier → Credit for old plan

**When Credit Detected:**
- Plan is NOT updated (preserved from current state)
- Price is NOT updated (preserved from current state)
- Only `polarTransactionId` and other metadata are updated

### Example

**User upgrades Pro → Plus:**
- Charge webhook: Plus (tier 2), amount: +$79 → Updates plan to Plus ✅
- Credit webhook: Pro (tier 1), amount: -$39 → Detected as credit, plan stays Plus ✅

---

## Scheduled Downgrades

### How It Works

1. **Scheduling:**
   - User requests downgrade
   - System stores `nextPlan` in database
   - System schedules BullMQ job with `executeAt = currentPeriodEnd`

2. **Job Processing:**
   - Job runs at period end
   - Verifies period has actually ended (fetches from Polar if needed)
   - If period not ended, reschedules job
   - Updates Polar subscription to new plan
   - Updates local database
   - Clears `nextPlan`

3. **Webhook Protection:**
   - `order.paid` webhooks check for `nextPlan`
   - If `nextPlan` exists and period hasn't ended, plan is preserved
   - Prevents premature downgrade from webhooks

### Job ID Format

```javascript
// Format: scheduled-downgrade-{userId}
// Note: No colons (BullMQ requirement)
const jobId = `scheduled-downgrade-${userId}`;
```

### Cancellation

Scheduled downgrade jobs are cancelled when:
- User upgrades to a different plan
- User resumes subscription
- Subscription is revoked
- User cancels subscription (soft cancel)

---

## State Machine Logic

### `buildNextSubscriptionState()` Function

This function deterministically calculates the next subscription state based on:
- Current state
- Webhook event type
- Webhook payload data

**Key Principles:**
1. **Idempotency**: Same input → Same output
2. **Validation**: Invalid states are rejected
3. **Preservation**: Fields not explicitly changed are preserved

### State Transitions by Event

#### `order.paid`

**For Paid Plans:**
- Updates plan (unless credit or scheduled downgrade)
- Updates price (unless credit or trial)
- Updates `polarTransactionId`
- Updates `currentPeriodEnd`

**For Trials:**
- Price remains `0`
- Status remains `trialing`
- `trialingEndsAt` preserved

**Credit Detection:**
- Negative amount OR downgrade webhook → Preserve plan/price

**Scheduled Downgrade:**
- If `nextPlan` exists and period not ended → Preserve current plan

#### `subscription.created`

- Sets `polarSubscriptionId`
- Sets plan from webhook
- Sets status: `trialing` if trial detected, else `active`
- Sets `trialingEndsAt` if trial

#### `subscription.updated`

**For Trialing:**
- Preserves `trialingEndsAt`
- Updates plan if changed
- Status remains `trialing`

**For Active:**
- Updates plan if changed
- Updates price
- Status remains `active`

#### `subscription.canceled`

- Status: `cancelled_at_period_end`
- Plan preserved
- `nextPlan` set to `free` (if not already set)

#### `subscription.uncanceled`

**For Trials:**
- If trial hasn't ended → Status: `trialing`, price: `0`
- If trial ended → Status: `active`, price: plan price

**For Active:**
- Status: `active`
- Price: plan price

#### `subscription.revoked`

- Status: `free`
- Plan: `free`
- `polarSubscriptionId`: `null`
- All subscription fields cleared
- Cancels scheduled downgrade jobs

---

## Edge Cases & Solutions

### 1. Credit Webhook Downgrading Plan

**Problem:** Credit webhooks during upgrades were downgrading the plan.

**Solution:** Detect credits by:
- Negative amount, OR
- Downgrade webhook (incoming tier < current tier)

**Location:** `webhook.controller.js` - `buildNextSubscriptionState()`

### 2. Scheduled Downgrade Overridden by Webhook

**Problem:** `order.paid` webhooks were immediately applying scheduled downgrades.

**Solution:** Check for `nextPlan` and preserve current plan if period hasn't ended.

**Location:** `webhook.controller.js` - `buildNextSubscriptionState()`

### 3. Trial Resume Setting Wrong Status

**Problem:** Resuming cancelled trial set status to `active` instead of `trialing`.

**Solution:** Check if `trialingEndsAt` is in future, restore to `trialing` if so.

**Location:** 
- `subscription.controller.js` - `resume()`
- `webhook.controller.js` - `subscription.uncanceled` handler

### 4. Stale `order.paid` Reactivating Revoked Subscription

**Problem:** Delayed `order.paid` webhooks were reactivating revoked subscriptions.

**Solution:** Check webhook event history for recent `subscription.revoked` events.

**Location:** `webhook.controller.js` - `order.paid` handler

### 5. BullMQ Job ID Invalid Character

**Problem:** Job ID contained colon (`:`) which BullMQ doesn't allow.

**Solution:** Changed format from `scheduled-downgrade:${userId}` to `scheduled-downgrade-${userId}`.

**Location:** `services/scheduledDowngrade.js`

### 6. Trial Plan Change Not Ending Trial

**Problem:** Changing plan during trial was updating in-place instead of ending trial.

**Solution:** Revoke trial, set to `free`, create new subscription via checkout.

**Location:** `subscription.controller.js` - `upgrade()`

### 7. Same Plan Trial "Upgrade"

**Problem:** User could "upgrade" to same plan they're trialing.

**Solution:** Check if current plan matches requested plan, return error.

**Location:** `subscription.controller.js` - `upgrade()`

### 8. Proration Behavior Not Supported

**Problem:** Using `proration_behavior: "none"` which Polar doesn't support.

**Solution:** Removed parameter, let Polar handle proration automatically.

**Location:** `queues/workers.js` - `apply-scheduled-downgrade` worker

### 9. Scheduled Downgrade Not Cancelled on Revocation

**Problem:** Scheduled downgrade jobs weren't cancelled when subscription revoked.

**Solution:** Call `cancelScheduledDowngradeJob()` in `subscription.revoked` handler.

**Location:** `webhook.controller.js` - `subscription.revoked` handler

### 10. Period End Stale in Database

**Problem:** `currentPeriodEnd` in database could be stale or in past.

**Solution:** Fetch from Polar API if database value is missing or invalid.

**Location:** 
- `subscription.controller.js` - `upgrade()` (downgrade scheduling)
- `queues/workers.js` - `apply-scheduled-downgrade` worker

---

## Database Schema

### Subscription Model

```javascript
{
  userId: ObjectId,
  plan: {
    name: "pro" | "plus" | "agency" | "free",
    productId: String  // Polar product ID
  },
  status: "active" | "trialing" | "cancelled_at_period_end" | "free" | "expired",
  nextPlan: {
    name: String,
    productId: String
  } | null,
  interval: "monthly" | "yearly",
  price: Number,  // In dollars
  currentPeriodEnd: Date,
  trialingEndsAt: Date | null,
  polarSubscriptionId: String | null,
  polarTransactionId: String | null
}
```

### Key Fields

- **`plan`**: Current active plan (object with name and productId)
- **`nextPlan`**: Scheduled plan change (for downgrades)
- **`status`**: Current subscription state
- **`trialingEndsAt`**: Trial end date (null if not trialing)
- **`currentPeriodEnd`**: End of current billing period
- **`polarSubscriptionId`**: Polar.sh subscription ID (null if revoked)

---

## API Endpoints

### `POST /api/subscription/upgrade`

**Request:**
```json
{
  "plan": "pro" | "plus" | "agency",
  "interval": "monthly" | "yearly"
}
```

**Behavior:**
- **Upgrade (paid-to-paid)**: Immediate update via Polar API
- **Downgrade (paid-to-paid)**: Schedule for period end
- **Trial → Different Plan**: Revoke trial, create checkout
- **Trial → Same Plan**: Return error
- **Free → Paid**: Create checkout

**Response:**
```json
{
  "checkoutUrl": "https://..."  // For new subscriptions
}
// OR
{
  "currentPlan": "plus",
  "nextPlan": "pro"  // For scheduled downgrades
}
```

### `POST /api/subscription/cancel`

**Behavior:**
- Soft cancel: Sets `cancel_at_period_end: true` in Polar
- Status: `cancelled_at_period_end`
- `nextPlan`: `free`
- Subscription remains active until period ends

### `POST /api/subscription/resume`

**Behavior:**
- Removes `cancel_at_period_end` in Polar
- Restores status:
  - `trialing` if trial hasn't ended
  - `active` if trial ended or was active
- Clears `nextPlan`
- Cancels scheduled downgrade jobs

---

## Webhook Endpoints

### `POST /api/webhooks/polar`

**Processing:**
1. Verify signature using Polar webhook secret
2. Check idempotency (prevent duplicates)
3. Extract user ID
4. Build next state using state machine
5. Validate state
6. Update database atomically
7. Mark webhook as processed

**Idempotency:**
- Uses `event.id` or `event.timestamp` as primary key
- Falls back to payload hash if no ID
- Prevents duplicate processing

---

## Job Queue

### Scheduled Downgrade Job

**Queue:** `scheduled-downgrade`
**Job ID Format:** `scheduled-downgrade-{userId}`

**Process:**
1. Fetch subscription from database
2. Verify `nextPlan` exists
3. Fetch `currentPeriodEnd` from Polar (if DB value stale)
4. If period not ended, reschedule job
5. Update Polar subscription to `nextPlan`
6. Update local database
7. Clear `nextPlan`

**Error Handling:**
- If subscription not found → Log and complete
- If period not ended → Reschedule
- If Polar API error → Retry with exponential backoff

---

## Best Practices

### 1. Always Use State Machine

Never update subscription state directly. Always use `buildNextSubscriptionState()` to ensure consistency.

### 2. Validate Before Writing

Always validate state using `validateSubscriptionState()` before writing to database.

### 3. Handle Webhook Idempotency

Always check if webhook was already processed before updating state.

### 4. Preserve Fields Not Changed

When building next state, preserve fields that aren't explicitly updated.

### 5. Fetch from Polar When Needed

If database value is stale or missing, fetch from Polar API as fallback.

### 6. Log State Transitions

Always log state transitions for debugging and audit trails.

### 7. Handle Edge Cases

- Check for revoked subscriptions
- Handle stale webhooks
- Verify period end dates
- Check for scheduled downgrades

---

## Testing Checklist

When testing subscription flows, verify:

- [ ] Upgrade immediately updates plan
- [ ] Downgrade schedules for period end
- [ ] Credit webhooks don't downgrade plan
- [ ] Scheduled downgrade applies at period end
- [ ] Trial resume restores correct status
- [ ] Trial plan change revokes trial
- [ ] Stale webhooks don't reactivate revoked subscriptions
- [ ] Scheduled downgrade cancelled on upgrade/resume/revocation
- [ ] Period end fetched from Polar if stale
- [ ] Same plan trial upgrade returns error

---

## Common Pitfalls

1. **Not checking for credits**: Always detect credit webhooks to prevent downgrades
2. **Ignoring scheduled downgrades**: Check `nextPlan` before updating plan
3. **Not preserving trial status**: Check `trialingEndsAt` when resuming
4. **Stale period end dates**: Always verify with Polar API
5. **Not cancelling jobs**: Cancel scheduled jobs on revocation/upgrade/resume
6. **Direct state updates**: Always use state machine function
7. **Missing idempotency checks**: Always check if webhook was processed

---

## Future Enhancements

- Discount/coupon system integration
- Proration calculation display
- Subscription history/audit log
- Multi-currency support
- Subscription pause/resume
- Plan change preview (showing proration)

---

## Support

For issues or questions about the subscription system:
1. Check this documentation
2. Review webhook logs
3. Check state transition logs
4. Verify Polar.sh subscription status
5. Check database state vs Polar state

---

**Last Updated:** February 2026
**Version:** 1.0

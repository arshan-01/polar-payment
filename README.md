# Subscription & Payment System Documentation

This is the maintained reference for current subscription behavior and code structure.
It reflects the refactored implementation used in `sample-backend` and is intended for future updates.

## Table of Contents

- [Subscription \& Payment System Documentation](#subscription--payment-system-documentation)
  - [Table of Contents](#table-of-contents)
  - [Canonical Code Layout](#canonical-code-layout)
  - [Client Architecture and Structure](#client-architecture-and-structure)
    - [Main client file](#main-client-file)
    - [Client state model](#client-state-model)
    - [Client request flow](#client-request-flow)
  - [Subscription States](#subscription-states)
  - [Behavior Matrix](#behavior-matrix)
  - [Upgrade Flow](#upgrade-flow)
    - [High-level path](#high-level-path)
    - [Notes](#notes)
  - [Cancel and Resume Flows](#cancel-and-resume-flows)
    - [Cancel (`POST /api/subscription/cancel`)](#cancel-post-apisubscriptioncancel)
    - [Resume (`POST /api/subscription/resume`)](#resume-post-apisubscriptionresume)
  - [Webhook Processing](#webhook-processing)
  - [Scheduled Downgrade Job](#scheduled-downgrade-job)
  - [Discount System](#discount-system)
    - [Discount Code Layout](#discount-code-layout)
    - [Discount States](#discount-states)
    - [Discount Creation Flow](#discount-creation-flow)
    - [Discount Attachment to Checkout](#discount-attachment-to-checkout)
    - [Discount Webhook Reconciliation](#discount-webhook-reconciliation)
    - [Discount Cleanup and Reconciliation Jobs](#discount-cleanup-and-reconciliation-jobs)
    - [Complete Discount Code Sections](#complete-discount-code-sections)
      - [1) Discount creation orchestration](#1-discount-creation-orchestration)
      - [2) Discount checkout attachment](#2-discount-checkout-attachment)
      - [3) Discount lifecycle transitions](#3-discount-lifecycle-transitions)
      - [4) Discount webhook reconciliation](#4-discount-webhook-reconciliation)
      - [5) Discount integration in checkout creation](#5-discount-integration-in-checkout-creation)
      - [6) Discount webhook integration](#6-discount-webhook-integration)
  - [Complete Code Sections](#complete-code-sections)
    - [1) Upgrade controller orchestration](#1-upgrade-controller-orchestration)
    - [2) Trial change logic (service)](#2-trial-change-logic-service)
    - [3) Paid downgrade scheduling (service)](#3-paid-downgrade-scheduling-service)
    - [4) Soft cancel and resume (service)](#4-soft-cancel-and-resume-service)
    - [5) Webhook transition service (extracted domain state machine)](#5-webhook-transition-service-extracted-domain-state-machine)
    - [6) Webhook idempotency service (extracted replay guard)](#6-webhook-idempotency-service-extracted-replay-guard)
  - [Complete Client Code Sections](#complete-client-code-sections)
    - [1) Client query and mutation setup](#1-client-query-and-mutation-setup)
    - [2) Client derived subscription state](#2-client-derived-subscription-state)
    - [3) Client upgrade action handling](#3-client-upgrade-action-handling)
    - [4) Client cancel-resume-billing interactions](#4-client-cancel-resume-billing-interactions)
  - [Invariants and Guarantees](#invariants-and-guarantees)
  - [Testing Checklist](#testing-checklist)
  - [Known Pitfalls](#known-pitfalls)

---

## Canonical Code Layout

Use these files as the source of truth:

- `sample-backend/src/modules/subscription/subscription.controller.js`
- `sample-backend/src/modules/subscription/subscription.payment.service.js`
- `sample-backend/src/modules/subscription/subscription.persistence.js`
- `sample-backend/src/modules/subscription/subscription.constants.js`
- `sample-backend/src/modules/webhook/webhook.controller.js`
- `sample-backend/src/modules/webhook/webhook.transition.service.js`
- `sample-backend/src/modules/webhook/webhook.idempotency.service.js`
- `sample-backend/src/modules/webhook/webhookEvent.model.js`
- `sample-backend/src/config/env.schema.js` (for `WEBHOOK_EVENT_TTL_SECONDS`)
- `sample-backend/src/config/env.js` (mapped runtime config value)
- `sample-backend/src/services/scheduledDowngrade.js`

**Discount system files:**
- `sample-backend/src/modules/discount/discount.constants.js`
- `sample-backend/src/modules/discount/discount.lifecycle.js`
- `sample-backend/src/modules/discount/discount.eligibility.js`
- `sample-backend/src/modules/discount/discount.polar.js`
- `sample-backend/src/modules/discount/discount.persistence.js`
- `sample-backend/src/modules/discount/discount.orchestrator.js`
- `sample-backend/src/modules/discount/discount.checkout.js`
- `sample-backend/src/modules/discount/discount.reconciliation.js`
- `sample-backend/src/modules/discount/discount.reconciliation.job.js`
- `sample-backend/src/services/discountService.js`

If `legacy-backend/` also exists in this repo, treat it as legacy/mirror unless explicitly chosen as runtime target.

---

## Client Architecture and Structure

The subscription UI is driven from a single page component and a small subscription API client.

### Main client file

- `sample-frontend/src/pages/Subscription.tsx`
- API methods imported from `@/api/subscription`:
  - `fetchSubscriptionPlans`
  - `fetchCurrentSubscription`
  - `fetchSubscriptionUsage`
  - `upgradeSubscription`
  - `createBillingPortal`
  - `resumeSubscription`

### Client state model

The client normalizes backend payloads to support both canonical and legacy shapes:

- `current_plan` or `plan`
- `subscription_status` or `status`
- `next_plan` or `nextPlan`
- `current_period_end` or `currentPeriodEnd`
- `billing_interval` or `interval`
- `trialing_ends_at`
- `trial_used_at`
- `active_discount`

### Client request flow

1. Load plans (`subscription/plans`), current state (`subscription/current`), and usage (`subscription/usage`) via React Query.
2. User action on plan card triggers `upgradeSubscription(plan, interval)`.
3. Client branches on response:
   - `checkoutUrl` -> redirect to checkout
   - `nextPlan/currentPlan` -> show scheduled downgrade success message
   - otherwise -> show generic success
4. Resume action calls `resumeSubscription`.
5. Billing actions call `createBillingPortal` and redirect to returned `url`.
6. On successful mutations, invalidate `["subscription"]` and refresh auth session.

---

## Subscription States

Supported status values:

- `active`: paid subscription is active
- `trialing`: trial is active and not charged yet
- `cancelled_at_period_end`: cancellation scheduled, access still active
- `free`: no effective paid subscription
- `expired`: subscription ended

Common transitions:

File: `documentation-only (conceptual state flow)`
```text
free -> trialing -> active
free -> active
active -> cancelled_at_period_end -> free
trialing -> free (trial revoked)
active -> active (paid upgrade/same-tier change)
```

---

## Behavior Matrix

Current backend behavior:

| Scenario | Expected Behavior | Proration |
|---|---|---|
| Trialing -> Upgrade (different plan) | Trial revoked immediately, user reset to free, new checkout created | No proration call |
| Trialing -> Downgrade (different plan) | Same as above: revoke + checkout | No proration call |
| Trialing -> Same Plan | Rejected with validation error | N/A |
| Paid -> Upgrade | Immediate Polar subscription update | Yes (`proration_behavior: "invoice"`) |
| Paid -> Downgrade | Deferred: store `nextPlan`, schedule job for period end | No immediate proration |
| Paid -> Same Tier (interval switch) | Immediate Polar update | Yes (`invoice`) |
| Any Paid/Trial -> Free | Immediate revoke/reset to free | No scheduling |
| Cancel Endpoint | Soft cancel at next billing period | No immediate plan removal |
| Resume Endpoint | Uncancel, restore `active` or `trialing` | N/A |

---

## Upgrade Flow

### High-level path

1. Validate billing context and ownership.
2. Validate requested plan and interval.
3. Load current subscription.
4. Determine change direction (upgrade/downgrade/same-tier).
5. Handle free plan switch immediately if requested.
6. Resolve product ID for paid plans.
7. If user is trialing and changing to a different plan:
   - revoke trial immediately
   - reset local state to free
   - create new checkout
8. If user has effective paid subscription:
   - paid upgrade: immediate Polar update with proration
   - paid downgrade: set `nextPlan`, schedule job at period end
   - same-tier change: immediate Polar update with proration
9. If no effective paid subscription: create checkout.

### Notes

- Trial transitions do not use in-place `updatePolarSubscription`.
- Paid downgrade does not update Polar immediately.
- Paid upgrade and same-tier change use immediate Polar PATCH.

---

## Cancel and Resume Flows

### Cancel (`POST /api/subscription/cancel`)

1. Validate owner context.
2. Require active `polarSubscriptionId`.
3. Reject if already `cancelled_at_period_end`.
4. Call Polar cancel with `effectiveFrom: "next_billing_period"`.
5. Update local status to `cancelled_at_period_end`, set `nextPlan` to free.
6. Cancel any scheduled downgrade job.

### Resume (`POST /api/subscription/resume`)

1. Validate owner context.
2. Require `polarSubscriptionId`.
3. Require local status `cancelled_at_period_end`.
4. Uncancel in Polar (`cancel_at_period_end: false`), optionally restore product ID.
5. Restore local state:
   - `trialing` if trial end is still in the future
   - otherwise `active`
6. Clear `nextPlan`.
7. Cancel scheduled downgrade job.

---

## Webhook Processing

Handled event families:

- `order.paid`
- `subscription.created`
- `subscription.updated`
- `subscription.active`
- `subscription.canceled` / `subscription.cancelled`
- `subscription.uncanceled`
- `subscription.revoked`

Core processing pipeline:

1. Verify signature.
2. Apply idempotency gate.
3. Resolve user and derive plan/price/interval/period end.
4. Build next state with `buildNextSubscriptionState`.
5. Validate invariants with `validateSubscriptionState`.
6. Write state atomically.
7. Mark event processed.

Idempotency retention:

- Processed webhook events are stored in `WebhookEvent`.
- A TTL index expires old idempotency records using `processedAt`.
- Retention is configured via `WEBHOOK_EVENT_TTL_SECONDS` (default: 90 days).

Credit/rollback guard during upgrades:

- `order.paid` refunds/credits must not roll plan backward.
- Downgrade-like webhook during an upgrade path is treated as credit and does not overwrite upgraded state.

Revoked subscription guard:

- If local state is already revoked (`free`) and webhook refers to old subscription IDs, event is ignored.

---

## Scheduled Downgrade Job

Scheduling utility:

- `sample-backend/src/services/scheduledDowngrade.js`
- Job ID format: `scheduled-downgrade-{userId}`

When scheduling:

1. Save `nextPlan` in DB.
2. Schedule job for `currentPeriodEnd`.

Worker behavior:

1. Re-read current subscription and period end.
2. If period has not ended yet, re-schedule.
3. If ended, update Polar to next plan.
4. Update local plan and clear `nextPlan`.

Cancellation triggers:

- immediate revoke to free
- soft cancel
- resume
- upgrade path that invalidates previously scheduled downgrade

---

## Discount System

The discount system provides user-specific, single-use discounts that are automatically applied at checkout. Discounts are created in Polar.sh and tracked locally for eligibility, lifecycle management, and reconciliation.

### Discount Code Layout

Core discount modules:

- `sample-backend/src/modules/discount/discount.constants.js` - Status constants and transition rules
- `sample-backend/src/modules/discount/discount.lifecycle.js` - Idempotent status transitions
- `sample-backend/src/modules/discount/discount.eligibility.js` - Eligibility checks
- `sample-backend/src/modules/discount/discount.polar.js` - Polar API adapter
- `sample-backend/src/modules/discount/discount.persistence.js` - Database operations
- `sample-backend/src/modules/discount/discount.orchestrator.js` - Main orchestration
- `sample-backend/src/modules/discount/discount.checkout.js` - Checkout attachment helpers
- `sample-backend/src/modules/discount/discount.reconciliation.js` - Webhook reconciliation
- `sample-backend/src/modules/discount/discount.reconciliation.job.js` - Periodic reconciliation
- `sample-backend/src/services/discountService.js` - Main entry point (backward compatible)

### Discount States

Supported discount status values:

- `active`: Discount is valid and can be used
- `used`: Discount was applied to a payment
- `expired`: Discount passed its expiration date
- `revoked`: Discount was manually revoked
- `failed`: Discount creation in Polar failed

**Final states** (cannot transition to other states):
- `used`, `expired`, `revoked`, `failed`

**Valid transitions:**

File: `documentation-only (conceptual state flow)`
```text
active -> used (via order.paid or reconciliation)
active -> expired (via timeout or reconciliation)
active -> revoked (via manual revoke or reconciliation)
active -> failed (via Polar creation failure)
```

**Transition sources:**
- `checkout_attach`: Discount attached to checkout
- `order_paid`: Payment completed with discount
- `timeout_expiry`: Discount expired naturally
- `manual_revoke`: Manually revoked
- `polar_failure`: Polar API failure
- `reconciliation`: Periodic reconciliation job

### Discount Creation Flow

**High-level path:**

1. Check eligibility (user not on target plan, marketing consent, no active/recent discount, cooldown period respected)
2. Create Polar discount first (with `max_redemptions: 1`, `duration: "once"`, `restrict_to` product IDs)
3. Create local database record
4. Send email notification (if enabled)
5. Handle compensation: if Polar succeeds but DB fails, log for reconciliation

**Eligibility checks:**
- User must not already be on target plan
- User must have marketing consent (or be legacy user)
- No active discount exists
- No recent discount within cooldown period (default: 30 days)
- Business rules (e.g., minimum usage for `usage_limit_reached` trigger)

**Discount triggers:**
- `usage_limit_reached`: User hit a feature limit
- `locked_feature_accessed`: User accessed locked feature
- `upgrade_flow_abandoned`: User started but didn't complete upgrade
- `inactivity_after_usage`: User was active last period, inactive this period

### Discount Attachment to Checkout

**Flow:**

1. Resolve applicable discount for selected plan using `resolveApplicableDiscount()`
2. Check if discount applies to selected plan (via `restrict_to` in Polar)
3. Build checkout metadata with `discountId` mapping
4. Attach discount to Polar checkout via `discount_id` parameter
5. Handle exhausted discount retry: if Polar returns `discount_usage_limit_exceeded`, mark discount as used and retry without discount

**Deterministic mapping:**
- Internal `discountId` (MongoDB ObjectId) stored in checkout `metadata.discountId`
- Polar `discount_id` passed to `createPolarCheckout()`
- Both paths reconciled in webhook processing

### Discount Webhook Reconciliation

**When `order.paid` webhook is received:**

1. Extract discount identifiers:
   - Internal `discountId` from `metadata.discountId`
   - Polar `discount_id` from `data.discount_id`
2. Reconcile via `reconcileDiscountUsage()`:
   - Try internal `discountId` first (most reliable)
   - Fallback to Polar `discount_id` if needed
   - Idempotent: if discount already finalized, return success without action
3. Mark discount as used via lifecycle manager
4. Track analytics event

**Idempotency guarantees:**
- Safe to call multiple times with same inputs
- Already-finalized discounts return success without state change
- Race conditions handled via atomic status transitions

### Discount Cleanup and Reconciliation Jobs

**Expired discount cleanup:**
- Scheduled: Daily at 2 AM
- Task: `discount-expiry`
- Function: `markExpiredDiscounts()`
- Behavior: Finds active discounts past `expiresAt`, transitions to `expired` status

**Periodic reconciliation:**
- Scheduled: Weekly on Sundays
- Task: `discount-reconciliation`
- Function: `runDiscountReconciliation()`
- Behavior: Detects drift between local and Polar state, reconciles expired discounts

**Note:** Discounts are not deleted when expired — they remain in database with `expired` status for audit trail. Only deleted when user account is permanently deleted.

### Complete Discount Code Sections

#### 1) Discount creation orchestration

File: `sample-backend/src/modules/discount/discount.orchestrator.js`
```javascript
export async function evaluateAndCreateDiscount(userId, trigger, context = {}) {
  if (!config.discount?.enabled) {
    return { created: false, reason: "system_disabled" };
  }

  // Check eligibility
  const eligibilityResult = await checkEligibility(userId, trigger, context);
  if (!eligibilityResult.eligible) {
    return { created: false, reason: eligibilityResult.reason };
  }

  // Check for existing active discount (race condition guard)
  const existingActive = await getActiveDiscount(userId);
  if (existingActive) {
    return { created: false, reason: "already_exists", discount: existingActive };
  }

  const rule = getDiscountRule(trigger);
  const targetPlan = rule.targetPlan || "pro";
  const percentOff = rule.percentOff ?? 20;
  const expiresAt = calculateExpirationDate(DISCOUNT_VALIDITY_DAYS);

  // Step 1: Create Polar discount first
  let paymentDiscountId;
  try {
    paymentDiscountId = await createPolarDiscountWithMetadata({
      percentOff,
      targetPlan,
      expiresAt,
      description: `User discount: ${trigger} (${percentOff}% off ${targetPlan})`,
      userId,
      trigger,
      version: "1.0"
    });
  } catch (err) {
    return { created: false, reason: "polar_discount_failed" };
  }

  // Step 2: Create DB record
  let discount;
  try {
    discount = await createDiscountRecord({
      userId,
      paymentProvider: "polar",
      paymentDiscountId,
      trigger,
      targetPlan,
      discountType: rule.percentOff ? "percent_off" : "amount_off",
      discountValue: percentOff,
      expiresAt,
      status: DISCOUNT_STATUS_ACTIVE,
      metadata: context
    });
  } catch (err) {
    // Compensation: Polar discount exists but DB failed
    logger.error({ userId, trigger, paymentDiscountId, err }, "DB creation failed after Polar success");
    return { created: false, reason: "creation_failed", polarDiscountId: paymentDiscountId };
  }

  // Step 3: Send email notification
  if (paymentDiscountId && discount.status === DISCOUNT_STATUS_ACTIVE) {
    try {
      const user = await User.findById(userId).select("email name").lean();
      if (user?.email) {
        const { html, text, subject } = discountOfferEmail(user, trigger, rule, context);
        await enqueueEmailEvent({
          eventId: `discount-offer:${userId}:${discount._id}`,
          type: "discount-offer",
          to: user.email,
          subject,
          text,
          html
        });
      }
    } catch (err) {
      logger.warn({ userId, discountId: discount._id, err }, "Failed to send discount email");
    }
  }

  return { created: true, discount };
}
```

#### 2) Discount checkout attachment

File: `sample-backend/src/modules/discount/discount.checkout.js`
```javascript
export async function resolveApplicableDiscount(userId, selectedPlan) {
  const activeDiscount = await getActiveDiscount(userId);

  if (!activeDiscount) {
    return { discount: null, discountId: null, shouldAttach: false };
  }

  // Discount is restricted to its target plan in Polar (restrict_to)
  const discountAppliesToSelectedPlan = String(activeDiscount.targetPlan) === String(selectedPlan);

  if (!discountAppliesToSelectedPlan) {
    return { discount: activeDiscount, discountId: null, shouldAttach: false };
  }

  const paymentDiscountId = activeDiscount.paymentDiscountId || null;
  if (!paymentDiscountId) {
    return { discount: activeDiscount, discountId: null, shouldAttach: false };
  }

  return {
    discount: activeDiscount,
    discountId: paymentDiscountId,
    shouldAttach: true
  };
}

export async function handleExhaustedDiscountRetry(err, userId, discount) {
  const code = err?.payload?.error?.code;
  if (code === "discount_usage_limit_exceeded" && discount?._id) {
    await markDiscountUsedSafe(userId, discount._id.toString(), DISCOUNT_TRANSITION_SOURCE.CHECKOUT_ATTACH);
    return { shouldRetry: true, retryDiscountId: null };
  }
  return { shouldRetry: false, retryDiscountId: null };
}
```

#### 3) Discount lifecycle transitions

File: `sample-backend/src/modules/discount/discount.lifecycle.js`
```javascript
export async function transitionDiscountStatus(discountId, toStatus, source, options = {}) {
  const discount = await UserDiscount.findById(discountId);
  if (!discount) return null;

  const fromStatus = discount.status;

  // No-op if already in target status
  if (fromStatus === toStatus) return discount;

  // Check if transition is valid
  if (!isValidTransition(fromStatus, toStatus, source)) {
    logger.warn({ discountId, fromStatus, toStatus, source }, "Invalid discount transition");
    return null;
  }

  // Prevent transitions from final states
  if (FINAL_DISCOUNT_STATUSES.includes(fromStatus)) {
    return null;
  }

  const update = {
    status: toStatus,
    lastTransitionSource: source,
    lastTransitionAt: new Date()
  };

  if (toStatus === DISCOUNT_STATUS_USED && !discount.usedAt) {
    update.usedAt = new Date();
  }

  // Atomic update with status check
  const updated = await UserDiscount.findOneAndUpdate(
    { _id: discountId, status: fromStatus },
    update,
    { new: true }
  );

  return updated;
}
```

#### 4) Discount webhook reconciliation

File: `sample-backend/src/modules/discount/discount.reconciliation.js`
```javascript
export async function reconcileDiscountUsage(userId, discountId, polarDiscountId) {
  // Try internal discountId first (most reliable)
  if (discountId) {
    try {
      const marked = await markDiscountUsedSafe(
        userId,
        discountId,
        DISCOUNT_TRANSITION_SOURCE.ORDER_PAID
      );
      if (marked) {
        return { marked: true, discountId };
      }
    } catch (err) {
      logger.warn({ userId, discountId, err }, "Failed to mark discount via internal ID");
    }
  }

  // Fallback to Polar discount_id
  if (polarDiscountId) {
    try {
      const existing = await getDiscountByPaymentId(userId, polarDiscountId);
      if (existing) {
        if (existing.status === DISCOUNT_STATUS_USED || existing.status === DISCOUNT_STATUS_EXPIRED) {
          return { marked: false, discountId: existing._id.toString(), reason: "already_finalized" };
        }
      }

      const marked = await markDiscountUsedByPaymentIdSafe(
        userId,
        polarDiscountId,
        DISCOUNT_TRANSITION_SOURCE.ORDER_PAID
      );
      if (marked) {
        const discount = await getDiscountByPaymentId(userId, polarDiscountId);
        return { marked: true, discountId: discount?._id?.toString() || null };
      }
    } catch (err) {
      logger.warn({ userId, polarDiscountId, err }, "Failed to mark discount via Polar ID");
    }
  }

  return { marked: false, discountId: null, reason: discountId || polarDiscountId ? "reconciliation_failed" : "no_discount_ids" };
}
```

#### 5) Discount integration in checkout creation

File: `sample-backend/src/modules/subscription/subscription.payment.service.js`
```javascript
export async function createCheckoutForNewSubscription(
  billingUserId,
  customerId,
  paidProductId,
  targetPlanId,
  interval,
  userEmail,
  userName
) {
  // Resolve applicable discount using centralized helper
  const { discount: activeDiscount, discountId, shouldAttach } = await resolveApplicableDiscount(
    billingUserId,
    targetPlanId
  );

  // Build checkout metadata with discount ID mapping
  const customData = buildCheckoutMetadata(billingUserId, targetPlanId, interval, activeDiscount);

  const allowTrial = billingUser?.trialUsedAt == null;

  let transaction;
  try {
    transaction = await createPolarCheckout({
      customerId,
      productId: paidProductId,
      metadata: customData,
      discountId: shouldAttach ? discountId : null,
      allowTrial,
      successUrl: `${config.app.frontendBaseUrl}/subscription?success=1`,
      cancelUrl: `${config.app.frontendBaseUrl}/subscription?canceled=1`
    });
  } catch (err) {
    // Handle exhausted discount retry using centralized helper
    const retryResult = await handleExhaustedDiscountRetry(err, billingUserId, activeDiscount);
    if (retryResult.shouldRetry) {
      delete customData.discountId;
      transaction = await createPolarCheckout({
        customerId,
        productId: paidProductId,
        metadata: customData,
        discountId: null,
        allowTrial,
        successUrl: `${config.app.frontendBaseUrl}/subscription?success=1`,
        cancelUrl: `${config.app.frontendBaseUrl}/subscription?canceled=1`
      });
    } else {
      throw err;
    }
  }

  return { checkoutUrl: transaction?.checkout_url || transaction?.url };
}
```

#### 6) Discount webhook integration

File: `sample-backend/src/modules/webhook/webhook.controller.js`
```javascript
// In order.paid handler:
const discountId = customData?.discountId ?? null;
const polarDiscountId = data?.discount_id ?? null;

// Reconcile discount usage using unified helper (idempotent)
const reconciliationResult = await reconcileDiscountUsage(userId, discountId, polarDiscountId);
if (reconciliationResult.marked) {
  analyticsTrack(userId, "coupon_used", {
    discountId: reconciliationResult.discountId,
    plan
  });
}
```

---

## Complete Code Sections

These code sections are intentionally complete for core flow maintenance.

### 1) Upgrade controller orchestration

File: `sample-backend/src/modules/subscription/subscription.controller.js`
```javascript
const upgrade = asyncHandler(async (req, res) => {
  const { billingUserId, isSharedContext } = await resolveBillingContext(req);
  if (isSharedContext || String(billingUserId) !== String(req.user._id)) {
    throw new AppError("Only the workspace owner can change subscription", StatusCodes.FORBIDDEN);
  }

  const { interval } = validatePlanChangeInput(req.body.plan, req.body.interval, plans);

  const existingSubscription = await Subscription.findOne({ userId: billingUserId }).lean();
  const currentPlanName = getPlanName(existingSubscription?.plan) || "free";
  const currentStatus = existingSubscription?.status || "active";

  const { isUpgrade, isDowngrade } = computePlanChangeDirection(currentPlanName, req.body.plan);
  const hasEffectivePolarSub = hasEffectivePolarSubscription(existingSubscription, currentStatus, currentPlanName);

  if (req.body.plan === "free") {
    const result = await handleSwitchToFree(billingUserId, existingSubscription, hasEffectivePolarSub);
    return ok(res, result.data, result.message);
  }

  const paidProductId = getPaidProductId(req.body.plan, interval);
  if (!paidProductId) {
    throw new AppError("Polar product is not configured", StatusCodes.INTERNAL_SERVER_ERROR);
  }

  const { trialingRevoked } = await handleTrialToPaidTransition(
    billingUserId,
    currentStatus,
    currentPlanName,
    req.body.plan,
    existingSubscription
  );

  if (!trialingRevoked && hasEffectivePolarSub && existingSubscription?.polarSubscriptionId) {
    const subId = existingSubscription.polarSubscriptionId;

    if (String(currentStatus) === "active" && isUpgrade) {
      const result = await handlePaidUpgrade(billingUserId, subId, paidProductId, req.body.plan, interval);
      return ok(res, result.data, result.message);
    }

    if (String(currentStatus) === "active" && isDowngrade) {
      const result = await handlePaidDowngrade(
        billingUserId,
        existingSubscription,
        paidProductId,
        req.body.plan,
        interval,
        currentPlanName
      );
      return ok(res, result.data, result.message);
    }

    const result = await handleSameTierChange(billingUserId, subId, paidProductId, req.body.plan, interval);
    return ok(res, result.data, result.message);
  }

  const customerId = await ensurePolarCustomer(billingUserId, req.user.email, req.user.name);
  const { checkoutUrl } = await createCheckoutForNewSubscription(
    billingUserId,
    customerId,
    paidProductId,
    req.body.plan,
    interval,
    req.user.email,
    req.user.name
  );

  return ok(res, { checkoutUrl }, "Checkout session created");
});
```

### 2) Trial change logic (service)

File: `sample-backend/src/modules/subscription/subscription.payment.service.js`
```javascript
export async function handleTrialToPaidTransition(
  billingUserId,
  currentStatus,
  currentPlanName,
  targetPlanId,
  existingSubscription
) {
  if (String(currentStatus) !== "trialing" || !existingSubscription?.polarSubscriptionId) {
    return { trialingRevoked: false };
  }

  if (String(currentPlanName) === String(targetPlanId)) {
    throw new AppError(
      "You are already on this plan. Your trial will automatically convert to paid when it ends.",
      StatusCodes.BAD_REQUEST
    );
  }

  const subId = existingSubscription.polarSubscriptionId;
  try {
    await revokePolarSubscription(subId);
  } catch (err) {
    if (err?.status !== 404) throw err;
    logger.warn({ userId: billingUserId, subscriptionId: subId }, "Subscription not found in Polar when revoking trial");
  }

  await resetToFreePlan(billingUserId, { clearPolarIds: true });
  await cancelScheduledDowngradeJob(billingUserId);
  return { trialingRevoked: true };
}
```

### 3) Paid downgrade scheduling (service)

File: `sample-backend/src/modules/subscription/subscription.payment.service.js`
```javascript
export async function handlePaidDowngrade(
  billingUserId,
  existingSubscription,
  paidProductId,
  targetPlanId,
  interval,
  currentPlanName
) {
  let periodEnd = existingSubscription?.currentPeriodEnd ? new Date(existingSubscription.currentPeriodEnd) : null;
  const now = new Date();

  if (!periodEnd || periodEnd <= now) {
    if (!existingSubscription?.polarSubscriptionId) {
      throw new AppError("Cannot schedule downgrade without active subscription", StatusCodes.BAD_REQUEST);
    }

    try {
      const polarSub = await getPolarSubscription(existingSubscription.polarSubscriptionId);
      const polarPeriodEnd = polarSub?.current_period_end ? new Date(polarSub.current_period_end) : null;
      if (polarPeriodEnd && polarPeriodEnd > now) {
        periodEnd = polarPeriodEnd;
        await updatePeriodEnd(billingUserId, polarPeriodEnd);
      }
    } catch (err) {
      logger.warn(
        { userId: billingUserId, subscriptionId: existingSubscription.polarSubscriptionId, err },
        "Failed to fetch period end from Polar, using DB value"
      );
    }
  }

  if (!periodEnd || periodEnd <= now) {
    throw new AppError("Cannot schedule downgrade: current billing period has ended or is invalid", StatusCodes.BAD_REQUEST);
  }

  const nextPlan = normalizePlan(targetPlanId, paidProductId, interval, getProductIdForPlan);
  await setScheduledDowngrade(billingUserId, nextPlan, periodEnd);
  return {
    success: true,
    message: "Downgrade scheduled for next billing cycle",
    data: { currentPlan: currentPlanName, nextPlan: targetPlanId }
  };
}
```

### 4) Soft cancel and resume (service)

File: `sample-backend/src/modules/subscription/subscription.payment.service.js`
```javascript
export async function handleSoftCancel(billingUserId, subscriptionId) {
  try {
    await cancelPolarSubscription(subscriptionId, { effectiveFrom: "next_billing_period" });
  } catch (err) {
    if (err?.status === 404) {
      logger.warn({ userId: billingUserId, subscriptionId }, "Subscription not found in Polar when canceling");
      throw new AppError("Subscription not found", StatusCodes.NOT_FOUND);
    }
    throw err;
  }

  await updateSubscriptionStatus(billingUserId, {
    status: SUBSCRIPTION_STATUS_CANCELLED_AT_PERIOD_END,
    nextPlan: normalizePlan("free")
  });
  await cancelScheduledDowngradeJob(billingUserId);
  return { success: true, message: "Subscription scheduled to cancel at period end" };
}

export async function handleResumeSubscription(billingUserId, subscription) {
  const currentPlanName = getPlanName(subscription.plan);
  const currentPlanProductId = getPlanProductId(subscription.plan);
  const interval = subscription.interval || "monthly";

  const now = new Date();
  const trialingEndsAt = subscription.trialingEndsAt ? new Date(subscription.trialingEndsAt) : null;
  const shouldRestoreToTrialing = !!trialingEndsAt && trialingEndsAt > now;

  const productIdToRestore = currentPlanProductId || getProductIdForPlan(currentPlanName, interval);

  if (productIdToRestore && currentPlanName !== "free") {
    await updatePolarSubscription(subscription.polarSubscriptionId, {
      cancel_at_period_end: false,
      product_id: productIdToRestore
    });
  } else {
    await updatePolarSubscription(subscription.polarSubscriptionId, { cancel_at_period_end: false });
  }

  const update = {
    status: shouldRestoreToTrialing ? SUBSCRIPTION_STATUS_TRIALING : SUBSCRIPTION_STATUS_ACTIVE,
    nextPlan: null,
    trialingEndsAt: shouldRestoreToTrialing ? trialingEndsAt : null
  };

  if (currentPlanName !== "free") {
    update.plan = currentPlanProductId
      ? normalizePlan(currentPlanName, currentPlanProductId)
      : normalizePlan(currentPlanName, productIdToRestore, interval, getProductIdForPlan);
  }

  await Subscription.findOneAndUpdate({ userId: billingUserId }, update, { new: true });
  await cancelScheduledDowngradeJob(billingUserId);
  return { success: true, message: "Subscription resumed" };
}
```

### 5) Webhook transition service (extracted domain state machine)

File: `sample-backend/src/modules/webhook/webhook.transition.service.js`
```javascript
/**
 * Validate subscription state invariants before write
 * @param {object} state - Subscription state to validate
 * @returns {object} { valid: boolean, error: string|null }
 */
function validateSubscriptionState(state, deps) {
  const { getPlanName } = deps;
  const planName = state.plan ? getPlanName(state.plan) : "free";
  const status = state.status;
  const polarSubscriptionId = state.polarSubscriptionId;
  const price = state.price;

  if (status === "active" && planName !== "free" && !polarSubscriptionId) {
    return {
      valid: false,
      error: `Invalid state: active paid plan (${planName}) must have polarSubscriptionId`
    };
  }

  if (planName === "free" || status === "free") {
    if (price !== 0 && price !== null && price !== undefined) {
      return {
        valid: false,
        error: `Invalid state: free plan must have price = 0, got ${price}`
      };
    }
    if (polarSubscriptionId !== null && polarSubscriptionId !== undefined) {
      return {
        valid: false,
        error: "Invalid state: free plan must have polarSubscriptionId = null"
      };
    }
  }

  if (status === "cancelled_at_period_end") {
    if (!polarSubscriptionId) {
      return {
        valid: false,
        error: "Invalid state: cancelled_at_period_end must have polarSubscriptionId"
      };
    }
    if (!state.currentPeriodEnd) {
      return {
        valid: false,
        error: "Invalid state: cancelled_at_period_end must have currentPeriodEnd"
      };
    }
  }

  return { valid: true, error: null };
}

/**
 * Build next subscription state from current state and webhook event
 * This is the single source of truth for state transitions
 * @param {object} currentState - Current subscription state from DB
 * @param {string} eventType - Webhook event type
 * @param {object} eventData - Webhook event data
 * @param {object} derived - Derived values (plan, interval, price, etc.)
 * @returns {object} Next subscription state
 */
function buildNextSubscriptionState(currentState, eventType, eventData, derived, deps) {
  const {
    getPlanName,
    getProductIdForPlan,
    normalizePlan,
    getEffectivePrice
  } = deps;
  const {
    plan,
    interval,
    productId,
    price: webhookPrice,
    currentPeriodEnd,
    trialingEndsAt,
    incomingPolarSubscriptionId
  } = derived;

  const existingPlanName = currentState?.plan ? getPlanName(currentState.plan) : null;
  const existingStatus = currentState?.status;
  const existingNextPlanName = currentState?.nextPlan ? getPlanName(currentState.nextPlan) : null;

  const nextState = {
    ...(currentState || {}),
    _id: undefined,
    __v: undefined
  };

  if (eventType === "order.paid") {
    const planObject = normalizePlan(plan, productId, interval, getProductIdForPlan);
    const subscriptionStatusFromOrder = eventData?.subscription?.status ?? null;
    const isTrialOrder = subscriptionStatusFromOrder === "trialing";
    const orderTrialEnd =
      eventData?.subscription?.trial_end != null ? new Date(eventData.subscription.trial_end) : null;
    const isZeroPrice = webhookPrice === null || webhookPrice === 0;

    const PLAN_HIERARCHY = { free: 0, pro: 1, plus: 2, agency: 3 };
    const incomingPlanName = plan ? getPlanName(planObject) : null;
    const currentTier = existingPlanName ? (PLAN_HIERARCHY[existingPlanName] ?? 0) : 0;
    const incomingTier = incomingPlanName ? (PLAN_HIERARCHY[incomingPlanName] ?? 0) : 0;
    const isDowngrade = incomingTier > 0 && currentTier > 0 && incomingTier < currentTier;
    const isCredit = (webhookPrice !== null && webhookPrice < 0) || (isDowngrade && existingPlanName !== null);

    const shouldApplyNextPlan =
      !isZeroPrice && !isCredit && existingNextPlanName && String(existingNextPlanName) === String(plan);

    const hasScheduledDowngrade = existingNextPlanName && existingNextPlanName !== "free";
    const periodEnd = currentState?.currentPeriodEnd ? new Date(currentState.currentPeriodEnd) : null;
    const now = new Date();
    const periodNotEnded = periodEnd && periodEnd > now;
    const shouldPreserveCurrentPlan = hasScheduledDowngrade && periodNotEnded && !shouldApplyNextPlan;

    const shouldUpdatePlan = !shouldPreserveCurrentPlan && !isCredit;

    if (shouldUpdatePlan) {
      nextState.plan = planObject;
      nextState.interval = interval ?? undefined;
    }

    const isScheduledToCancel = existingStatus === "cancelled_at_period_end";
    const periodEndForCancel = currentState?.currentPeriodEnd ? new Date(currentState.currentPeriodEnd) : null;
    const cancelPeriodNotEnded = periodEndForCancel && periodEndForCancel > now;
    const shouldPreserveCancellation = isScheduledToCancel && cancelPeriodNotEnded;

    if (!shouldPreserveCancellation) {
      nextState.status = isTrialOrder ? "trialing" : "active";
    } else {
      nextState.status = "cancelled_at_period_end";
    }

    nextState.trialingEndsAt = isTrialOrder
      ? (trialingEndsAt ?? orderTrialEnd ?? currentState?.trialingEndsAt ?? null)
      : null;

    if (shouldPreserveCurrentPlan || isCredit) {
      // Keep existing price when preserving the current plan or when this event is a credit.
    } else if (isTrialOrder) {
      nextState.price = 0;
    } else if (webhookPrice !== null && Number.isFinite(webhookPrice) && webhookPrice > 0) {
      nextState.price = webhookPrice;
    } else {
      const planName = getPlanName(planObject);
      nextState.price = getEffectivePrice(planName, interval);
    }

    const subscriptionId = eventData?.subscription_id ?? eventData?.subscription?.id ?? incomingPolarSubscriptionId;
    if (subscriptionId) {
      nextState.polarSubscriptionId = subscriptionId;
    }
    if (eventData?.id) {
      nextState.polarTransactionId = eventData.id;
    }
    if (currentPeriodEnd) {
      nextState.currentPeriodEnd = currentPeriodEnd;
    }
    if (shouldApplyNextPlan) {
      nextState.nextPlan = null;
    }
    if (shouldPreserveCancellation && currentState?.nextPlan) {
      nextState.nextPlan = currentState.nextPlan;
    }
  } else if (eventType === "subscription.created") {
    if (incomingPolarSubscriptionId) {
      nextState.polarSubscriptionId = incomingPolarSubscriptionId;
    }
    if (plan) {
      const planObject = normalizePlan(plan, productId, interval, getProductIdForPlan);
      const planName = getPlanName(planObject);
      nextState.plan = planObject;
      nextState.interval = interval ?? undefined;

      const hasTrialInCreated =
        eventData?.status === "trialing" ||
        !!trialingEndsAt ||
        !!eventData?.trial_end ||
        !!eventData?.trial_start;

      if (planName !== "free") {
        nextState.status = hasTrialInCreated ? "trialing" : "active";
      }

      if (hasTrialInCreated) {
        if (trialingEndsAt) {
          nextState.trialingEndsAt = trialingEndsAt;
        }
        nextState.price = 0;
      }
    }
    if (currentPeriodEnd) {
      nextState.currentPeriodEnd = currentPeriodEnd;
    }
    if (nextState.status !== "trialing" && webhookPrice !== null && Number.isFinite(webhookPrice) && webhookPrice > 0) {
      nextState.price = webhookPrice;
    } else if (nextState.status !== "trialing" && plan) {
      const planName = getPlanName(normalizePlan(plan, productId, interval, getProductIdForPlan));
      nextState.price = getEffectivePrice(planName, interval);
    }
  } else if (eventType === "subscription.updated" && eventData?.status === "trialing") {
    if (existingStatus !== "active") {
      nextState.status = "trialing";
      nextState.trialingEndsAt = trialingEndsAt ?? currentPeriodEnd ?? currentState?.trialingEndsAt ?? null;
      if (incomingPolarSubscriptionId) {
        nextState.polarSubscriptionId = incomingPolarSubscriptionId;
      }
      if (plan) {
        nextState.plan = normalizePlan(plan, productId, interval, getProductIdForPlan);
        nextState.interval = interval ?? undefined;
      } else if (currentState?.plan) {
        nextState.plan = normalizePlan(currentState.plan, null, currentState?.interval ?? interval, getProductIdForPlan);
        nextState.interval = nextState.interval ?? currentState?.interval ?? undefined;
      }
    }
  } else if (eventType === "subscription.canceled" || eventType === "subscription.cancelled") {
    const endsAt = eventData?.ends_at ?? eventData?.current_period_end ?? null;
    const periodEndDate = endsAt ? new Date(endsAt) : (currentPeriodEnd ? new Date(currentPeriodEnd) : null);
    const now = new Date();
    const endsInFuture = periodEndDate && periodEndDate > now;

    if (endsInFuture) {
      nextState.status = "cancelled_at_period_end";
      nextState.nextPlan = normalizePlan("free");
      nextState.currentPeriodEnd = periodEndDate ?? currentPeriodEnd ?? null;
      if (incomingPolarSubscriptionId) {
        nextState.polarSubscriptionId = incomingPolarSubscriptionId;
      }
    } else {
      nextState.plan = normalizePlan("free");
      nextState.status = "free";
      nextState.nextPlan = null;
      nextState.price = 0;
      nextState.currentPeriodEnd = null;
      nextState.trialingEndsAt = null;
      nextState.polarSubscriptionId = null;
      nextState.polarTransactionId = null;
    }
  } else if (eventType === "subscription.updated" && eventData?.status !== "trialing") {
    const cancelAtPeriodEnd = eventData?.cancel_at_period_end === true;

    if (cancelAtPeriodEnd) {
      const endsAt = eventData?.ends_at ?? eventData?.current_period_end ?? null;
      const periodEndDate = endsAt ? new Date(endsAt) : (currentPeriodEnd ? new Date(currentPeriodEnd) : null);
      const now = new Date();
      const endsInFuture = periodEndDate && periodEndDate > now;

      if (endsInFuture) {
        nextState.status = "cancelled_at_period_end";
        nextState.nextPlan = normalizePlan("free");
        nextState.currentPeriodEnd = periodEndDate ?? currentPeriodEnd ?? null;
        if (incomingPolarSubscriptionId) {
          nextState.polarSubscriptionId = incomingPolarSubscriptionId;
        }
      }
    } else if (eventData?.status === "active" && incomingPolarSubscriptionId) {
      nextState.status = "active";
      nextState.trialingEndsAt = null;
      nextState.polarSubscriptionId = incomingPolarSubscriptionId;
      if (currentPeriodEnd) {
        nextState.currentPeriodEnd = currentPeriodEnd;
      }
      if (webhookPrice !== null && Number.isFinite(webhookPrice) && webhookPrice > 0) {
        nextState.price = webhookPrice;
      }
      const existingNextPlanNameInner = currentState?.nextPlan ? getPlanName(currentState.nextPlan) : null;
      if (!existingNextPlanNameInner || !["pro", "plus", "agency"].includes(existingNextPlanNameInner)) {
        nextState.nextPlan = null;
      }
    }
  } else if (eventType === "subscription.active") {
    const hasTrialSignal =
      eventData?.status === "trialing" ||
      !!trialingEndsAt ||
      !!eventData?.trial_end ||
      !!eventData?.trial_start ||
      existingStatus === "trialing";

    nextState.status = hasTrialSignal ? "trialing" : "active";
    nextState.nextPlan = null;
    nextState.trialingEndsAt = hasTrialSignal ? (trialingEndsAt ?? currentState?.trialingEndsAt ?? null) : null;
    if (incomingPolarSubscriptionId) {
      nextState.polarSubscriptionId = incomingPolarSubscriptionId;
    }
    if (currentPeriodEnd) {
      nextState.currentPeriodEnd = currentPeriodEnd;
    }
    if (!hasTrialSignal && webhookPrice !== null && Number.isFinite(webhookPrice) && webhookPrice > 0) {
      nextState.price = webhookPrice;
    } else if (hasTrialSignal) {
      nextState.price = 0;
    }
  } else if (eventType === "subscription.uncanceled") {
    const now = new Date();
    const existingTrialingEndsAt = currentState?.trialingEndsAt ? new Date(currentState.trialingEndsAt) : null;
    const wasTrialing = !!existingTrialingEndsAt;
    const trialNotEnded = existingTrialingEndsAt && existingTrialingEndsAt > now;
    const shouldRestoreToTrialing = wasTrialing && trialNotEnded;

    nextState.status = shouldRestoreToTrialing ? "trialing" : "active";
    nextState.nextPlan = null;

    if (shouldRestoreToTrialing) {
      nextState.trialingEndsAt = existingTrialingEndsAt;
      nextState.price = 0;
    } else {
      nextState.trialingEndsAt = null;
      if (webhookPrice !== null && Number.isFinite(webhookPrice) && webhookPrice > 0) {
        nextState.price = webhookPrice;
      }
    }

    if (incomingPolarSubscriptionId) {
      nextState.polarSubscriptionId = incomingPolarSubscriptionId;
    }
    if (currentPeriodEnd) {
      nextState.currentPeriodEnd = currentPeriodEnd;
    }
  } else if (eventType === "subscription.revoked") {
    nextState.plan = normalizePlan("free");
    nextState.status = "free";
    nextState.nextPlan = null;
    nextState.price = 0;
    nextState.currentPeriodEnd = null;
    nextState.trialingEndsAt = null;
    nextState.polarSubscriptionId = null;
    nextState.polarTransactionId = null;
  }

  if (nextState.status === "trialing" && !nextState.trialingEndsAt && nextState.currentPeriodEnd) {
    nextState.trialingEndsAt = nextState.currentPeriodEnd;
  }

  Object.keys(nextState).forEach((key) => {
    if (nextState[key] === undefined) {
      delete nextState[key];
    }
  });

  return nextState;
}

export { validateSubscriptionState, buildNextSubscriptionState };
```

### 6) Webhook idempotency service (extracted replay guard)

File: `sample-backend/src/modules/webhook/webhook.idempotency.service.js`
```javascript
import crypto from "crypto";

/**
 * Generate unique event key for idempotency.
 * Prefer provider event identity (event.id/event.timestamp). Fallback to
 * payload-derived hash only when provider identity is unavailable.
 */
function getEventKey(eventType, eventId, eventData = null) {
  if (eventId) {
    return `${eventType}:${eventId}`;
  }
  if (eventData) {
    const payloadKey = {
      dataId: eventData?.id ?? null,
      created_at: eventData?.created_at ?? null,
      modified_at: eventData?.modified_at ?? null,
      status: eventData?.status ?? null,
      cancel_at_period_end: eventData?.cancel_at_period_end ?? null,
      ends_at: eventData?.ends_at ?? null
    };
    const payloadHash = crypto
      .createHash("md5")
      .update(JSON.stringify(payloadKey))
      .digest("hex")
      .substring(0, 12);
    return `${eventType}:payload:${payloadHash}`;
  }
  return `${eventType}:unknown`;
}

/**
 * Check if webhook event was already processed (idempotency)
 * @returns {object|null} Existing event record or null
 */
async function checkEventIdempotency(eventType, eventId, eventData = null, deps = {}) {
  const { webhookEventModel, loggerInstance } = deps;
  if (!webhookEventModel) {
    throw new Error("webhookEventModel is required for checkEventIdempotency");
  }
  const eventKey = getEventKey(eventType, eventId, eventData);
  try {
    const existing = await webhookEventModel.findOne({ eventKey }).lean();
    return existing;
  } catch (err) {
    loggerInstance?.error?.({ err, eventKey }, "Error checking webhook idempotency");
    return null;
  }
}

/**
 * Mark webhook event as processed
 */
async function markEventProcessed(eventType, eventId, userId, previousState, nextState, eventData = null, deps = {}) {
  const { webhookEventModel, loggerInstance } = deps;
  if (!webhookEventModel) {
    throw new Error("webhookEventModel is required for markEventProcessed");
  }
  const eventKey = getEventKey(eventType, eventId, eventData);
  try {
    await webhookEventModel.create({
      eventKey,
      eventType,
      eventId: eventId || "unknown",
      userId: userId || null,
      previousState: previousState ? JSON.parse(JSON.stringify(previousState)) : null,
      nextState: nextState ? JSON.parse(JSON.stringify(nextState)) : null
    });
  } catch (err) {
    if (err.code === 11000) {
      loggerInstance?.warn?.({ eventKey }, "Webhook event already marked as processed");
    } else {
      loggerInstance?.error?.({ err, eventKey }, "Error marking webhook event as processed");
    }
  }
}

export { getEventKey, checkEventIdempotency, markEventProcessed };
```

---

## Complete Client Code Sections

### 1) Client query and mutation setup

File: `sample-frontend/src/pages/Subscription.tsx`
```typescript
const { data: plans = [], isLoading: isLoadingPlans } = useQuery({
  queryKey: ["subscription", "plans"],
  queryFn: fetchSubscriptionPlans,
  enabled: !!user,
});

const { data: currentSubscription, isLoading: isLoadingCurrent } = useQuery({
  queryKey: ["subscription", "current", contextChannelId ?? "me"],
  queryFn: () => fetchCurrentSubscription(contextChannelId),
  enabled: !!user,
});

const { data: usage } = useQuery({
  queryKey: ["subscription", "usage", contextChannelId ?? "me"],
  queryFn: () => fetchSubscriptionUsage(contextChannelId),
  enabled: !!user,
});

const upgradeMutation = useMutation({
  mutationFn: (vars: { plan: string; interval: BillingInterval }) =>
    upgradeSubscription(vars.plan, vars.interval),
  onSuccess: async () => {
    await queryClient.invalidateQueries({ queryKey: ["subscription"] });
    dispatch(initializeSession());
  },
});

const portalMutation = useMutation({
  mutationFn: () => createBillingPortal()
});

const resumeMutation = useMutation({
  mutationFn: () => resumeSubscription(),
  onSuccess: async () => {
    await queryClient.invalidateQueries({ queryKey: ['subscription'] });
    dispatch(initializeSession());
    toast.success('Subscription resumed. Your plan will continue as before.');
  },
});
```

### 2) Client derived subscription state

File: `sample-frontend/src/pages/Subscription.tsx`
```typescript
const getPlanName = (plan: string | { name: string } | undefined): string => {
  if (!plan) return 'free';
  if (typeof plan === 'string') return plan;
  if (typeof plan === 'object' && plan.name) return plan.name;
  return 'free';
};

const currentPlan = getPlanName(currentSubscription?.current_plan ?? currentSubscription?.plan ?? user?.subscriptionPlan ?? 'free');
const subscriptionStatus = (
  currentSubscription?.subscription_status ??
  (currentSubscription?.status === 'trialing'
    ? 'trialing'
    : currentSubscription?.status === 'canceled' || currentSubscription?.status === 'cancelled'
      ? 'cancelled'
      : 'active')
) as SubscriptionStatus;

const nextPlanValue = currentSubscription?.next_plan ?? currentSubscription?.nextPlan ?? null;
const nextPlan = nextPlanValue ? getPlanName(nextPlanValue) : null;
const currentPeriodEnd = currentSubscription?.current_period_end ?? currentSubscription?.currentPeriodEnd ?? null;
const trialingEndsAt = currentSubscription?.trialing_ends_at ?? null;
const currentInterval = (currentSubscription?.billing_interval ?? currentSubscription?.interval ?? 'monthly') as BillingInterval;
const trialUsedAt = currentSubscription?.trial_used_at ?? null;
const canStartTrial = trialUsedAt == null;
const activeDiscount = currentSubscription?.active_discount ?? null;

const trialEndsAtDate = trialingEndsAt ? new Date(trialingEndsAt) : null;
const isTrialActiveByDate = Boolean(
  trialEndsAtDate &&
  !Number.isNaN(trialEndsAtDate.getTime()) &&
  trialEndsAtDate.getTime() > Date.now()
);
const isTrialing = subscriptionStatus === 'trialing' || isTrialActiveByDate;
const isCancelledAtPeriodEnd = subscriptionStatus === 'cancelled_at_period_end';
```

### 3) Client upgrade action handling

File: `sample-frontend/src/pages/Subscription.tsx`
```typescript
const handleUpgrade = async (plan: Plan) => {
  track('upgrade_clicked', { plan: plan.id, interval: billingInterval });
  try {
    const data = await upgradeMutation.mutateAsync({ plan: plan.id, interval: billingInterval });
    if (data?.checkoutUrl) {
      if (activeDiscount) {
        track('coupon_applied_ui', { plan: plan.id, discount_target: activeDiscount.target_plan });
      }
      window.location.href = data.checkoutUrl;
      return;
    }
    if (data?.nextPlan != null && data?.currentPlan != null) {
      toast.success('Downgrade scheduled for next billing cycle. Your current plan stays active until then.');
    } else {
      toast.success(`Switched to ${plan.name} plan.`);
    }
  } catch {
    toast.error('Failed to update subscription. Please try again.');
  }
};

const handlePlanAction = async (plan: Plan) => {
  const isFreePlan = plan.id === 'free' || plan.price === 0;
  if (isFreePlan && currentPlan !== 'free') {
    setPendingDowngradePlan(plan);
    setShowSwitchToFreeConfirm(true);
    return;
  }

  if (isTrialing && !isFreePlan && currentPlan !== plan.id) {
    setPendingUpgradePlan(plan);
    setShowUpgradeConfirm(true);
    return;
  }

  await handleUpgrade(plan);
};
```

### 4) Client cancel-resume-billing interactions

File: `sample-frontend/src/pages/Subscription.tsx`
```typescript
// Resume button action
await resumeMutation.mutateAsync();

// Manage billing action
const data = await portalMutation.mutateAsync();
window.location.href = data.url;

// UI behavior
// - If status is cancelled_at_period_end: show "Resume subscription"
// - Else: show "Manage billing"
// - For free plan: show upgrade CTA and trial availability based on trial_used_at
```

---

## Invariants and Guarantees

**Subscription invariants:**
1. Only workspace owner can mutate billing state.
2. Only one effective paid subscription is allowed per billing user.
3. Trial plan changes always revoke the trial first.
4. Paid downgrade is deferred and represented by `nextPlan`.
5. Cancel endpoint is soft cancel (`cancelled_at_period_end`), not revoke.
6. Free plan selection is immediate revoke/reset.
7. Webhook processing is idempotent and guarded against stale/revoked events.

**Discount invariants:**
1. Only one active discount per user (enforced by unique index).
2. Discounts are single-use (`max_redemptions: 1`, `duration: "once"`).
3. Discount status transitions are idempotent and race-safe.
4. Discounts are restricted to target plan products in Polar (`restrict_to`).
5. Discount eligibility respects cooldown periods (default: 30 days).
6. Discount creation requires Polar success before DB write (compensation path exists).
7. Discount usage reconciliation handles both internal `discountId` and Polar `discount_id` paths.
8. Expired discounts remain in database (status change, not deletion).

---

## Testing Checklist

Verify all cases before release:

- [ ] Free -> Paid checkout
- [ ] Trial -> same plan (expect validation error)
- [ ] Trial -> upgrade (revoke + checkout)
- [ ] Trial -> downgrade (revoke + checkout)
- [ ] Paid -> upgrade immediate with invoice proration
- [ ] Paid -> same-tier interval change immediate with invoice proration
- [ ] Paid -> downgrade deferred to period end
- [ ] Cancel endpoint sets `cancelled_at_period_end`
- [ ] Resume endpoint restores status correctly
- [ ] Scheduled downgrade job executes and clears `nextPlan`
- [ ] Credit `order.paid` events do not revert upgraded plan
- [ ] Stale events do not reactivate revoked subscriptions
- [ ] Discount creation: eligibility checks, Polar creation, DB persistence
- [ ] Discount attachment: applies only to matching target plan
- [ ] Discount exhausted retry: marks as used and retries checkout
- [ ] Discount webhook reconciliation: handles both internal and Polar discount IDs
- [ ] Discount expiry cleanup: marks expired discounts
- [ ] Discount reconciliation job: detects and reconciles state drift
- [ ] Unit tests for webhook transition service pass (`webhook.transition.service.test.js`)
- [ ] Unit tests for webhook idempotency service pass (`webhook.idempotency.service.test.js`)
- [ ] Env schema tests cover webhook TTL default and override (`env.schema.test.js`)

---

## Known Pitfalls

1. Keep webhook state machine as single source of truth for event-driven transitions.
2. Do not apply scheduled downgrades immediately in API layer.
3. Do not overwrite active paid state with stale trialing webhooks.
4. Reconcile `currentPeriodEnd` with Polar if DB value is missing/stale.
5. Cancel scheduled jobs when state changes invalidate them.
6. Keep `trialUsedAt` accurate; it controls `allowTrial` in checkout.
7. In canonical response helpers, compare `current_plan.name` to `"free"` (not object-to-string checks).
8. Discount transitions must use lifecycle manager for idempotency.
9. Discount eligibility checks must run before Polar creation.
10. Discount attachment must verify `restrict_to` matches selected plan.
11. Discount reconciliation must handle both `discountId` and `discount_id` paths.
12. Expired discounts are marked as expired (status change), not deleted.
13. Keep `WEBHOOK_EVENT_TTL_SECONDS` long enough to cover realistic webhook replay windows.

---

**Last Updated:** February 2026
**Version:** 2.3

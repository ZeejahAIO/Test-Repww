## Loyalty 2.0 — POS Solution (Implementation Plan)

This document is a **high-level solution guide** for implementing **Loyalty 2.0** in **POS** (`android-pos`).  
It’s inspired by the kiosk solution format you shared, but it maps the approach to **existing POS code and modules**.

### Main features covered

- **Customizable spend → points**
- **Points expiration** (and “up-next expiry” summary)
- **Max points redemption per order** (instead of 1 reward per order)
- **Daily loyalty transaction limit**
- **Signup bonus** messaging + flow
- **Birthday bonus** (configurable timeframe)
- **Reward types**
  - Free Item
  - Item Discount (Dollar Off)
  - Item Discount (% off) with max cap
  - Ticket Discount (Dollar Off) (min spend optional)
  - Ticket Discount (% off) with max cap (min spend optional)
- **Tiers**
  - Custom tier names
  - Minimum points per tier
  - Tier multiplier impacts **points earning only** (not reward eligibility)
- **CFD/SFD flow changes**
  - CFD signup/sign-in pre-order and post-payment
  - SFD guest panel + view rewards + redemption
- **Ordering with loyalty + discount calculation rules**
- **Refund/cancel/void/remove reward** behaviors
- **Points on hold** lifecycle
- **Menu Manager visibility/out-of-stock filtering** for item-based rewards
- **Receipts** include earned/redeemed/balance + reward details

---

## Technical debt coverage (recommended)

These are POS equivalents of issues that will cause instability/perf regressions during Loyalty 2.0 changes.

- **Loyalty rewards adapter uses `notifyDataSetChanged()` and does business logic in bind**
  - File: `pos/src/main/java/aio/app/pos/smartPanel/presentation/core/fragments/guestBook/loyaltyMenu/loyaltyMenuFragment/LoyaltyMenuAdapter.kt`
  - Current: `setData()` clears/adds + `notifyDataSetChanged()`, and item eligibility is computed inside `bindData`.
  - Fix: convert to `ListAdapter + DiffUtil` (or stable IDs + partial updates). Move eligibility/state mapping out of the adapter.
- **Remove “tier gating” coupling**
  - Current reward availability is checked via `customerPoints < loyalty.tier`.
  - Loyalty 2.0 removes reward-tier association; replace `tier` gating with **points-based + limits-based** eligibility states.
- **Centralize loyalty math and ordering**
  - Avoid implementing discount ordering/caps/compounding in fragments. Create a single source of truth (repository/domain functions).

---

## Architecture point of view (POS)

Keep the existing POS structure and avoid introducing extra layers unless already standard in this codebase.

- **UI → ViewModel → Repository → Remote/DB**

Guiding principles:

- UI renders **immutable UI state** (no business rules inside adapters / `onBind`).
- ViewModels build **UI-ready strings/state** and coordinate flows.
- Repository/domain functions handle:
  - reward eligibility computation
  - points earned/spent/held calculations
  - discount ordering rules + caps + compounding
  - refund/cancel/void/remove reward side effects

---

## Codebase baseline (what exists today)

### Current loyalty UI entry point

- `LoyaltyMenuDialogFragment`  
  `pos/src/main/java/aio/app/pos/smartPanel/presentation/core/fragments/guestBook/loyaltyMenu/loyaltyMenuFragment/LoyaltyMenuDialogFragment.kt`
  - Shows a loyalty grid and “Points available”
  - Reads customer points as:
    - `remainingPoints = loyaltyPoints - loyaltyPointsUsed`
  - Triggers:
    - `SmartPanelActions.OnLoyaltyItemClicked(loyalty)`

### Current loyalty models

- Domain UI model: `LoyaltyItemModel`  
  `pos/src/main/java/aio/app/pos/smartPanel/domain/model/guestBook/loyalties/LoyaltyItemModel.kt`
  - Includes `tier` which is used as gating.
- Remote DTO: `LoyaltyItem`  
  `pos/src/main/java/aio/app/pos/smartPanel/data/remote/model/loyalty/getLoyalty/LoyaltyItem.kt`
  - Includes `tier`, modifiers, taxes, stock status, etc.

### Current repo APIs

- `SmartPanelRepository.getLoyaltyMenu(isFromMqtt: Boolean)`
- `SmartPanelRepository.applyLoyalty(request)`
- `SmartPanelRepository.getSmartPanelCustomerInfo(customerId, isLoyaltyApplied, loyaltyPointsUsed)`

---

## Backend dependencies / contracts (Loyalty 2.0)

POS will need Loyalty 2.0 data that includes both **restaurant configuration** and **guest loyalty state**.

### Required restaurant config

- `pointsPerDollar` (spend→points conversion)
- `pointsExpiryEnabled`, `expiryWindowDays` (or equivalent rule model)
- `maxRedeemPointsPerOrder` (points cap per ticket)
- `maxLoyaltyTransactionsPerDay`
- `signupBonusEnabled`, `signupBonusPoints`, `signupCtaText`, `signupMarketingMessages[]`
- `birthdayBonusEnabled`, `birthdayBonusPoints`, `birthdayTimeframeRule` (birthday only / month / ±3 / ±7 days)
- `allowPointsForGuestUsers` (card-alias accumulation)
- `tiers[]`:
  - `tierId`, `tierName`, `minPoints`, `multiplier`

### Required guest state

- `availablePoints`
- `pointsOnHold`
- `pointsExpiringNext` (points + date)
- `currentTierId` (or derived from points)
- `dailyTxnCountUsed` (or backend boolean `dailyLimitReached`)

### Required rewards list

Each reward must include:

- `rewardId` (unique per reward)
- `type` enum:
  - `FREE_ITEM`
  - `ITEM_DISCOUNT_AMOUNT`
  - `ITEM_DISCOUNT_PERCENT` (+ max cap)
  - `TICKET_DISCOUNT_AMOUNT` (+ optional min spend)
  - `TICKET_DISCOUNT_PERCENT` (+ max cap + optional min spend)
- `pointsCost`
- Item linkage (for item rewards): `itemId` + any modifier rules needed
- `maxDiscountCapAmount?` (for percent types)
- `minSpendAmount?` (ticket discount optional)
- Visibility/in-stock signals (or rely on menu manager mapping)

---

## POS screen-by-screen solution

### 1) Guest selection and locking behavior (SFD)

Requirement (PRD):

- Guest can be changed until a loyalty reward is added.
- Once a loyalty reward is applied, guest change is blocked until reward removed/voided.

Implementation:

- Keep using `SmartPanelSharedViewModel` actions/events.
- Add explicit “reward applied” state at ticket level (so guest removal/change UI can react consistently).

### 2) Rewards selection UI (POS)

Replace “tier-gated rewards list” with **points + limits** based states.

#### Reward UI states (per reward)

Define a reward state enum (UI-level):

- `AVAILABLE`
- `SELECTED`
- `LOCKED` (insufficient points)
- `UNAVAILABLE` (limits/min spend/gift-card restriction/etc.)
- `OUT_OF_STOCK` (menu manager says unavailable)
- `HIDDEN` (item removed/visibility off)

#### Rewards screen layout guidance

Header should show:

- total available points
- points on hold
- per-order redemption limit (points)
- daily txn limit (and whether reached)
- next points expiry summary

Rewards list:

- show reward cards by type
- allow multi-select with rule: **unique reward per ticket**
- update eligibility in real time as points are held

Implementation suggestion:

- Refactor `LoyaltyMenuDialogFragment` into a Loyalty 2.0 rewards dialog that renders immutable state.
- Replace `LoyaltyMenuAdapter` with `ListAdapter` + `DiffUtil`.

### 3) Applying rewards to the ticket (business logic)

#### Free item

- Add reward item to ticket with **$0 price**
- Modifier handling:
  - priced modifiers not part of reward should retain price
  - modifiers included in the reward should be $0

#### Item discount (amount / percent)

- Apply to the associated item only
- Block other item comps on the same item
- For percent: enforce max cap
- Modifier discounting:
  - default: modifier price not discounted
  - exception: if backend says modifier is included in reward, apply percent discount to (item + modifier)

#### Ticket discount (amount / percent)

- Not applicable to gift card purchase (per PRD)
- Apply after item discounts/comps
- Enforce ordering rules with comps and compounding:
  - Amount-based loyalty ticket discount
  - Amount-based ticket comp
  - Percent-based loyalty ticket discount
  - Percent-based ticket comp
- Percent-based ticket discount must enforce max cap
- Min spend optional: if set, reward is `UNAVAILABLE` until subtotal meets min spend

### 4) Points earned calculation

Rules:

- compute on **discounted subtotal excluding tax**
- multiplier uses **current tier before this order**
- tier changes do **not** impact the current order’s points earning

Formula:

- `pointsEarned = discountedSubtotalExclTax * pointsPerDollar * currentTierMultiplier`

### 5) Points on hold lifecycle

- On reward select: points move to **hold**
- On reward remove/void: release hold
- On payment capture:
  - subtract redeemed points (captured hold)
  - add earned points
- On cancel payment:
  - remove earned points
  - redeemed points go back to **hold**

### 6) CFD changes (customer-facing display)

Implement the PRD flows:

- idle pre-signup marketing states (marketing video on/off)
- pre-payment: show “potential points to earn” + signup CTA + signup bonus messaging carousel
- post-payment: “don’t lose your points” + signup CTA + skip
- if “allow points for guest users” enabled: show accumulated points tied to card alias and add them on signup/login

### 7) Refund / cancel / void / remove reward

Refund system must show and act on loyalty fields:

- full refund:
  - editable redeemed points field
  - add redeemed points back
  - remove earned points
- partial refund:
  - earned points removed proportionally to refunded subtotal
  - redeemed points editable and refunded
  - if points-only refund, don’t call payment adjustment API (per PRD)
- void order / remove reward / void reward item:
  - release holds

### 8) Receipts

Receipts should include:

- points earned
- points redeemed
- current balance: `before - redeemed + earned`
- reward details display rules:
  - free item shows $0
  - item discount shows negative discount line + points used
  - ticket discount shows discount line + points used

---

## Implementation plan (epics → tickets)

### Epic 1 — Contracts + models

- Add Loyalty 2.0 DTOs and mapping from API to domain.
- Replace `tier` gating fields with:
  - `pointsCost`
  - `rewardType`
  - `caps/minSpend`
  - `rewardId`
  - item linkages
- Add a domain model for:
  - restaurant config
  - guest loyalty state
  - tiers

### Epic 2 — Rewards UI (POS)

- Refactor `LoyaltyMenuDialogFragment` UI to Loyalty 2.0 design/states.
- Convert `LoyaltyMenuAdapter` to `ListAdapter + DiffUtil`.
- Implement multi-select, points-held header, and eligibility updates.

### Epic 3 — Ticket integration + discount engine

- Implement reward effects application and validation.
- Implement discount ordering, caps, and compounding rules.
- Enforce gift-card restriction for ticket loyalty discounts.

### Epic 4 — CFD (pre-signup + post-payment)

- Implement CFD idle states and signup bonus messaging.
- Implement pre-payment potential points widget.
- Implement post-payment signup CTA + skip.

### Epic 5 — Payment + holds + capture

- Wire hold/capture/release events to backend.
- Ensure split payments attribute earned/redeemed points proportionally for refunds.

### Epic 6 — Refund + receipts

- Update refund UI to show/edit loyalty points.
- Implement points rollback logic (full + partial).
- Update printer templates/receipts.

### Epic 7 — Cleanup + performance

- Remove old tier gating remnants.
- Add unit tests for:
  - eligibility states
  - points earned calculation
  - discount ordering/caps/compounding
  - refund proportional rollback

---

## POS files likely to change first (starting point)

- `pos/.../LoyaltyMenuDialogFragment.kt`
- `pos/.../LoyaltyMenuAdapter.kt`
- `pos/.../LoyaltyItemModel.kt` (or replacement model)
- `pos/.../SmartPanelRepository.kt` / `SmartPanelRepositoryImpl.kt`
- Ticket/payment/refund/printing modules (based on where discounts and refund receipts are currently computed)


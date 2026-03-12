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

---

## Jira-ready breakdown (Epics → Stories)

This section turns the above plan into **Jira-friendly tickets** with titles, summaries, acceptance criteria, and file hints.

### Epic 1 — Loyalty 2.0 Contracts & Models (POS)

#### Story 1.1 — Define Loyalty 2.0 domain models

- **Title**: POS – Define Loyalty 2.0 domain models (config, guest, rewards, tiers)
- **Summary**: Introduce new domain models for restaurant loyalty config, guest loyalty state, tiers, and rewards, replacing tier-based gating.
- **Acceptance Criteria**
  - Domain models exist for:
    - `RestaurantLoyaltyConfig` (pointsPerDollar, maxRedeemPointsPerOrder, maxLoyaltyTxPerDay, signup/birthday config, allowPointsForGuestUsers, etc.).
    - `GuestLoyaltyState` (availablePoints, pointsOnHold, nextExpiry, currentTierId, dailyTxnCountUsed or dailyLimitReached).
    - `LoyaltyTier` (id, name, minPoints, multiplier).
    - `LoyaltyReward` (rewardId, type, pointsCost, itemId?, maxCapAmount?, minSpendAmount?, visibility/out-of-stock flags).
  - No UI or adapter reads `tier` directly as an eligibility threshold anymore.
  - Unit tests cover basic mapping invariants (e.g., percent rewards always carry a max cap field).
- **Likely Files**
  - New domain package, e.g. `pos/.../domain/loyalty/*.kt`.
  - Existing `LoyaltyItemModel.kt` (may be repurposed or wrapped).

#### Story 1.2 — Map API DTOs to Loyalty 2.0 domain models

- **Title**: POS – Map Loyalty 2.0 API response to new domain models
- **Summary**: Extend/replace existing mapping from Retrofit DTOs to the new domain models.
- **Acceptance Criteria**
  - All required Loyalty 2.0 fields (config, guest state, rewards, tiers) are mapped from API DTOs to domain models.
  - Mapping functions are pure (no side effects) and unit-tested.
  - UI/adapters do not depend directly on DTO classes.
- **Likely Files**
  - `RetrofitAPIService.kt`.
  - `smartPanel/data/remote/model/loyalty/...` DTOs.
  - `smartPanel/data/mapper/...` (new or extended mappers).

---

### Epic 2 — Rewards UI (Loyalty Menu Dialog)

#### Story 2.1 — Refactor LoyaltyMenuAdapter to ListAdapter + DiffUtil

- **Title**: POS – Refactor LoyaltyMenuAdapter to ListAdapter + DiffUtil
- **Summary**: Improve performance and UX by using `ListAdapter` with `DiffUtil` and immutable UI models.
- **Acceptance Criteria**
  - `LoyaltyMenuAdapter` extends `ListAdapter<RewardCardUi, VH>` with a `DiffUtil.ItemCallback`.
  - `notifyDataSetChanged()` is removed from the loyalty adapter.
  - Scroll position is preserved across loyalty state updates.
  - Eligibility/alpha/locked visuals are derived from a `RewardCardUi.state` enum, not from business logic in `onBind`.
- **Likely Files**
  - `LoyaltyMenuAdapter.kt`.
  - New UI model `RewardCardUi` (or similar).

#### Story 2.2 — Implement Loyalty 2.0 reward states in LoyaltyMenuDialogFragment

- **Title**: POS – Implement Loyalty 2.0 reward states in LoyaltyMenuDialogFragment
- **Summary**: Loyalty rewards dialog supports multiple reward types with proper state: AVAILABLE, SELECTED, LOCKED, UNAVAILABLE, OUT_OF_STOCK.
- **Acceptance Criteria**
  - Each reward card displays:
    - Type tag (Free, $ Off, % Off, On Entire Order, etc.).
    - Item name or “On Entire Order” label.
    - Points cost.
    - Max cap/min spend info when applicable.
  - Visual states:
    - `AVAILABLE`: normal; clickable.
    - `SELECTED`: visually selected; cannot be selected again.
    - `LOCKED`: insufficient points; greyed; tap shows toast.
    - `UNAVAILABLE`: max per-order points or daily txn limit reached; greyed; tap shows specific toast.
    - `OUT_OF_STOCK`: matches out-of-stock style; not clickable.
  - Header shows:
    - Available points.
    - Points held / max points per order.
    - Daily transaction limit (if configured) and whether it is reached.
  - Selecting/deselecting rewards updates header and card states correctly.
- **Likely Files**
  - `LoyaltyMenuDialogFragment.kt`.
  - `LoyaltyMenuAdapter.kt`.
  - Layouts/strings for new labels and toasts.

#### Story 2.3 — Multi-reward selection & uniqueness enforcement

- **Title**: POS – Support multi-reward selection with uniqueness and point updates
- **Summary**: Allow multiple different rewards per ticket while enforcing point caps and uniqueness.
- **Acceptance Criteria**
  - User can select multiple rewards on a single ticket as long as:
    - Sum of `pointsCost` of selected rewards ≤ `maxRedeemPointsPerOrder`.
    - Daily loyalty transaction limit is not exceeded.
  - Same `rewardId` cannot be selected more than once for the same ticket.
  - When a reward is selected:
    - Points held increase by `pointsCost`.
    - Eligible rewards update based on remaining points (grey out ineligible).
  - When a reward is deselected/removed:
    - Points held decrease.
    - Rewards that become affordable are re-enabled.
- **Likely Files**
  - `LoyaltyMenuDialogFragment.kt`.
  - `SmartPanelSharedViewModel`.
  - Loyalty state holder in repository/domain.

---

### Epic 3 — Ticket Integration & Discount Engine

#### Story 3.1 — Implement loyalty reward application engine

- **Title**: POS – Implement loyalty reward application engine (free item, item discount, ticket discount)
- **Summary**: Central domain logic takes selected rewards and ticket state and returns ticket changes plus validation errors.
- **Acceptance Criteria**
  - Engine input: `Ticket`, `selectedRewards[]`, `RestaurantLoyaltyConfig`, `GuestLoyaltyState`.
  - Engine output includes:
    - Free items to add with price = 0.
    - Item-level discounts (amount/percent) with caps enforced.
    - Ticket-level discounts (amount/percent) with caps and min-spend honored.
    - Validation errors (daily limit reached, per-order points exceeded, gift card restriction, min spend not met, etc.).
  - Rules:
    - Ticket-level loyalty discounts are never applied to gift card purchases.
    - Item-level loyalty discounts block additional item comps on that line.
    - Unique reward per ticket enforced.
  - Covered by unit tests for key combinations.
- **Likely Files**
  - New `LoyaltyDiscountEngine.kt` (or similar domain class).
  - Ticket mappers/transformers, e.g. `TicketMappers.kt`.

#### Story 3.2 — Integrate loyalty engine into QsrMenuFragment / TsrMenuFragment

- **Title**: POS – Wire loyalty engine into QSR/TSR ticket flows
- **Summary**: Use the loyalty engine for `OnLoyaltyItemClicked` / “Apply rewards” actions instead of fragment-level logic.
- **Acceptance Criteria**
  - `QsrMenuFragment` and `TsrMenuFragment` respond to `SmartPanelActions.OnLoyaltyItemClicked` by:
    - Delegating to repository/engine.
    - Updating ticket items according to engine output.
  - Fragments do not contain discount ordering/cap logic directly.
  - Engine errors are converted to appropriate user messages/toasts.
- **Likely Files**
  - `QsrMenuFragment.kt`.
  - `TsrMenuFragment.kt`.
  - `SmartPanelSharedViewModel`.
  - Ticket repositories/mappers.

#### Story 3.3 — Implement discount ordering & compounding rules

- **Title**: POS – Enforce loyalty and comp discount ordering and compounding rules
- **Summary**: Ensure item and ticket discounts (loyalty and comps) are applied in correct order and compounded correctly.
- **Acceptance Criteria**
  - Ticket-level ordering:
    - 1) Amount-based loyalty ticket discounts.
    - 2) Amount-based ticket comps.
    - 3) Percent-based loyalty ticket discounts.
    - 4) Percent-based ticket comps.
  - Each subsequent discount applies to the already discounted subtotal.
  - Percent-based discounts never exceed their configured caps.
  - Unit tests verify ordering and compounding for multiple scenarios.
- **Likely Files**
  - Discount/payment calculator components.
  - New tests near the payment calculator modules.

---

### Epic 4 — CFD Flows (Pre-signup, Post-payment)

#### Story 4.1 — CFD pre-signup idle + order-in-progress loyalty messaging

- **Title**: POS – Implement CFD pre-signup and order-in-progress loyalty views
- **Summary**: CFD shows loyalty messaging and signup CTA before signup, both idle and when a ticket is present.
- **Acceptance Criteria**
  - Idle with marketing video:
    - Marketing video plays in background.
    - Signup/login CTA text matches web-dash configuration.
  - Idle without marketing video:
    - No signup bonus: show “Login/sign up to earn points…” state.
    - With signup bonus: rotate configured messages (bonus points, free items, discounts) with defined message/transition timing.
  - When ticket exists and guest not signed:
    - Left: ticket summary (subtotal, tax, service charge, total only).
    - Right: potential points to earn (based on subtotal and spend→points config) and signup CTA.
- **Likely Files**
  - `customer_facing_ticket.xml` and related CFD layouts.
  - `MarketingCfdExtension.kt`, `MarketingScreenRepoImpl.kt`.
  - CFD state/viewmodel classes.

#### Story 4.2 — CFD post-payment signup/login screen

- **Title**: POS – Implement CFD post-payment loyalty signup/login screen
- **Summary**: After payment, show points that can be claimed and allow signup/login or skip.
- **Acceptance Criteria**
  - After payment completion when guest is not associated:
    - CFD shows total points to claim:
      - Non-card payment: points from this order only.
      - Card payment with guest-users-points enabled: this order + any previous alias points.
    - Shows “Don’t lose your points – sign up for discount on orders” (or configured text).
    - Shows “Login/Sign up” CTA and “Skip” button.
  - On signup:
    - Phone + name flow completes.
    - Points applied as per PRD: order points + signup bonus + previous alias points if any.
  - On skip:
    - User is taken to existing receipt selection flow (no receipt/print/email).
- **Likely Files**
  - Same CFD layouts and state management as above.
  - Backend integration for alias-based guest points (if not present).

---

### Epic 5 — Payment, Holds & Split Payments

#### Story 5.1 — Points on hold lifecycle implementation

- **Title**: POS – Implement points-on-hold and capture lifecycle
- **Summary**: When rewards are selected, points go on hold; on payment capture they are redeemed; on cancel/void they are released or returned to hold.
- **Acceptance Criteria**
  - On selecting a reward:
    - Required points move from available to on-hold state.
  - On removing/voiding a reward:
    - Points move from hold back to available.
  - On successful payment capture:
    - Redeemed points are deducted from guest balance.
    - Earned points from order are added.
  - On cancel payment:
    - Earned points for that payment are removed.
    - Redeemed points move back to hold, not lost.
- **Likely Files**
  - Loyalty repository/service in POS.
  - Payment completion/cancel handlers.

#### Story 5.2 — Split payments attribution for earned/redeemed points

- **Title**: POS – Attribute earned and redeemed points across split payments
- **Summary**: Earned and redeemed points are recorded against each split payment proportionally for accurate refunds.
- **Acceptance Criteria**
  - When payment is split:
    - Points earned and redeemed are split between payments proportional to their payment amounts/subtotals.
  - Data model persists per-payment point attribution.
  - Refund logic can use this to compute correct rollback on each payment.
- **Likely Files**
  - Payment logging DTOs (`PaymentDto`, etc.).
  - `PaymentRepositoryImpl` and related classes.

---

### Epic 6 — Refund, Cancel, Void, Remove Reward

#### Story 6.1 — Full and partial refund loyalty behavior

- **Title**: POS – Implement loyalty behavior on full and partial refunds
- **Summary**: Adjust earned and redeemed points correctly when issuing full or partial refunds.
- **Acceptance Criteria**
  - Full refund:
    - Refund UI shows editable “loyalty points redeemed” field.
    - On submit:
      - Redeemed points are added back to guest account.
      - Earned points for that payment are removed from guest account.
  - Partial refund:
    - Points earned are removed in proportion to refunded subtotal.
    - Redeemed points are editable; only specified portion is returned.
    - If only points are refunded, no payment adjustment API is called (per PRD).
- **Likely Files**
  - `RefundMainFragment` and child views.
  - Refund-related repository logic.
  - Loyalty repository/service.

#### Story 6.2 — Cancel payment / void order / remove reward behavior

- **Title**: POS – Implement loyalty behavior on cancel payment, void order, and remove/void reward
- **Summary**: Define consistent point state transitions for cancel payment, void order, removing reward, and voiding reward items.
- **Acceptance Criteria**
  - Cancel payment:
    - Earned points for that payment are removed.
    - Redeemed points move back to hold.
  - Void order:
    - All points on hold for that order are released to available.
  - Remove reward / void reward item:
    - Points on hold for that reward are released.
- **Likely Files**
  - Cancel payment flow handlers.
  - Order void flow handlers.
  - Ticket/reward removal handlers.

---

### Epic 7 — Receipts & Printing

#### Story 7.1 — Print loyalty details on receipts

- **Title**: POS – Update receipts to show loyalty rewards and point summaries
- **Summary**: Receipt templates show free items, loyalty discounts, and points summary (earned, redeemed, balance).
- **Acceptance Criteria**
  - Item lines:
    - Free item: price and total displayed as $0.00, non-reward modifiers priced normally.
    - Item discounts (amount/percent): discount shown as negative line; loyalty points used clearly displayed.
  - Ticket discounts:
    - Shown after item details as separate lines with discount amount and points used.
  - Payment section:
    - Discount field includes: loyalty item discount + loyalty ticket discount + item comps + ticket comps.
  - Loyalty summary:
    - Points earned on the order.
    - Points redeemed on the order.
    - Current balance = pointsBeforeOrder - redeemed + earned.
- **Likely Files**
  - `PrinterTemplates.kt`.
  - Printer data models for receipts.


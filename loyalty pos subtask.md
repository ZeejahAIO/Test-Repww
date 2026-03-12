

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


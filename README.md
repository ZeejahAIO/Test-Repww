# Loyalty Feature – Technical Analysis

**Document purpose:** Technical analysis for the Loyalty (Rewards / Loyalty Menu & Apply Loyalty) feature in POS and SmartPanel.  
**Audience:** Development team.  
**Reference:** Internal technical assessment; PRD/Figma references to be added if available.

---

## 1. What Is the Change?

### 1.1 Business change

**Current (before Loyalty):** Staff could not redeem loyalty rewards from POS. There was no tier-based loyalty menu, no way to add loyalty items to a ticket, and no display of customer loyalty points or eligible items on customer-facing screens.

**New:** With Loyalty implemented:

- **Loyalty menu:** Staff can open a loyalty/rewards menu (from Guest Book / SmartPanel) showing tier-based items. Items are grouped by tier; customer’s current tier determines which items are eligible.
- **Add to ticket:** Staff can add a selected loyalty item to the current ticket (QSR: offline add to ticket; TSR: apply-loyalty flow). The item is added with modifiers (and required modifiers) and appears on the ticket with loyalty tier/points.
- **Customer info:** When a customer is associated with a ticket, the app can show loyalty points (available/used), tier, and eligible items. Customer-facing display (CFD) can show loyalty profile, points, and eligible items.
- **Persistence:** Loyalty tier and points are stored on ticket/ticket items (e.g. `isLoyaltyApplied`, `loyaltyPointsApplied`, `loyalty_tiers_table` in Room) and reflected on receipts.

### 1.2 Product / UX change

| Aspect | Detail |
|--------|--------|
| **Location** | **SmartPanel / Guest Book:** Loyalty menu dialog (grid of tier headers + loyalty items). **Register (QSR/TSR):** Modifier panel flow when user selects a loyalty item from SmartPanel; “done” adds the item to the ticket. **CFD:** Loyalty profile view (login, points, tiers, eligible items). **Secondary screen:** Loyalty presentation (items by tier, tap to select). |
| **New control** | “Loyalty” entry point in Guest Book → opens **Loyalty Menu** dialog. In the dialog, each loyalty item is tappable; on tap the dialog dismisses and the selected item is sent to the Register modifier flow (QSR/TSR). Print/receipt flows show loyalty-related data where applicable. |
| **Scope** | **QSR:** Offline add of loyalty item to ticket (item + modifiers pushed to local DB; sync per existing ticket flow). **TSR:** Apply-loyalty API can update ticket with loyalty application; offline add also supported. **Sources:** Works for dine-in, takeout, delivery, etc.; behaviour may vary by order type. **Not in scope for this analysis:** mPOS-specific behaviour; exact TSR vs QSR apply-loyalty API usage by product. |
| **On tap (Loyalty item)** | Guest Book → Loyalty item tap → `SmartPanelActions.OnLoyaltyItemClicked` → Register (QSR/TSR) receives action, opens modifier panel with that item pre-selected; user configures modifiers and confirms → item is added to ticket (offline or via apply-loyalty where applicable). Success: item on ticket; secondary screen can show ticket presentation. |
| **Customer info** | When customer is set on ticket, SmartPanel/CFD can show points (available/used), tier, and eligible items. Customer info is loaded via **getSmartPanelCustomerInfo** API and cached in repository. |

### 1.3 Technical change (summary)

- **New UI:** Loyalty menu dialog (`LoyaltyMenuDialogFragment`), loyalty item adapters (`LoyaltyMenuAdapter`, `LoyaltyPresentationAdapter`), loyalty secondary screen, and CFD loyalty views. Register modifier panels (QsrMenuFragment, TsrMenuFragment) handle “loyalty item selected” and add-to-ticket flow.
- **New flows:** (1) Load loyalty menu via **getLoyaltyMenu** API → cache in repository → expose via `loyaltyMenu` / `menuItemsFlow`. (2) On loyalty item click → `SmartPanelSharedViewModel.triggerSmartPanelAction(OnLoyaltyItemClicked)` → Register subscribes and sets `selectedLoyaltyModel`, opens modifier UI → on done, `addLoyaltyItemOffline` (or apply-loyalty) runs. (3) Customer info: **getSmartPanelCustomerInfo** API → cache → StateFlow to UI.
- **New data:** Room table `loyalty_tiers_table` (`LoyaltyEntity`); ticket item fields for loyalty (e.g. `isLoyaltyApplied`, `loyaltyPointsApplied`). MenuDao: `getLoyaltyItem(itemId, modifierOptionId)`, `loyaltyPointsUsed`, etc.
- **Reuse:** Existing ticket/item APIs, modifier panels, MQTT/ticket sync, printer/receipt formatting. KDS print path is unchanged; loyalty is ticket/item and customer-display focused.

---

## 2. How Is It Implemented?

### 2.1 High-level flow

1. **Loyalty menu visibility and loading**
   - Guest Book (or equivalent) shows entry to Loyalty menu. On open, `GuestBookViewModel.getLoyaltyMenu()` is called (or menu is taken from `menuItemsFlow`).
   - `SmartPanelRepository.getLoyaltyMenu(isFromMqtt)` calls **GET api/loyalty/pos-menu**. Response is mapped to `BaseLoyaltyItem` list (tiers + items), cached in `SmartPanelRepositoryImpl.menuItems`, and exposed via `loyaltyMenu` StateFlow and `menuItemsFlow`.
   - `LoyaltyMenuDialogFragment` collects `loyaltyMenu` and/or `menuItemsFlow` and shows loading/success/error; success fills `LoyaltyMenuAdapter` with tier headers and loyalty items.

2. **Loyalty item selection and add-to-ticket**
   - User taps a loyalty item in the dialog → `onLoyaltyClick(loyalty)` → analytics + `smartPanelSharedViewModel.triggerSmartPanelAction(SmartPanelActions.OnLoyaltyItemClicked(loyalty.copy()))` → dialog dismisses.
   - Register (QsrMenuFragment / TsrMenuFragment) subscribes to `smartPanelNav`; on `OnLoyaltyItemClicked` it sets `selectedLoyaltyModel = loyaltyItemModel`, loads modifiers for that item (`getCombinedModifierOfLoyaltyItemTimeBase`), and opens modifier panel (e.g. `setCustomerDetails(loyaltyItemModel, true)`). If no required modifiers, it may call `checkLoyalty()` immediately.
   - When user confirms (e.g. modifier “Done”), Fragment calls `checkLoyalty()`: if `selectedLoyaltyModel != null`, it calls `smartPanelSharedViewModel.selectedLoyaltyModel.addLoyaltyItemOffline(...)` (extension in SmartPanelSharedViewModel) with current modifier list, special request, guest type, ticket id, order type. Then clears `selectedLoyaltyModel`.
   - **addLoyaltyItemOffline:** Builds request entity and metadata, gets loyalty tier from `menuTicketRepository.getLoyaltyItem(...)`, calls `makeItemReadyCache`, then `menuTicketRepository.pushItemToDb(...)`, fetches latest item, and emits `SmartPanelActions.OnLoyaltyAppliedQsrOffline(loyaltyItem)`. UI (e.g. TSR) reacts by closing modifier panel and refreshing ticket.

3. **Apply Loyalty (TSR / server-side)**
   - Where product uses apply-loyalty API: `SmartPanelRepository.applyLoyalty(ApplyLoyaltyRequestModel)` calls **POST api/ticket/apply-loyalty**. Response includes updated ticket data. Used when ticket is applied with loyalty on the server; POS may then refresh ticket state.

4. **Customer info and points**
   - When customer is set on ticket, `SmartPanelSharedViewModel.setCustomerInfo(customer, isLoyaltyApplied, loyaltyPointsUsed)` is called. It calls `smartPanelRepository.getSmartPanelCustomerInfo(customerId, ...)` → **GET api/customer/smartPanel?customerId=**. Result is cached in `customersInfoList` and emitted via `customerInfo` StateFlow. LoyaltyMenuDialogFragment (and CFD) collect this and show points (e.g. `loyaltyPoints - loyaltyPointsUsed`), tier, and eligible items.

5. **Visibility rules**
   - Loyalty menu is shown from Guest Book / SmartPanel entry points. “Print to KDS” style visibility (Active tab, serveType, orderSource) does not apply to Loyalty; Loyalty is available in the Register + SmartPanel flows where Guest Book and ticket context exist.

### 2.2 Implementation steps (as implemented in POS app)

| Step | Task | Notes |
|------|------|------|
| 1 | **Commons – data model** | `LoyaltyEntity` in commons (Room entity for `loyalty_tiers_table`). Used by MenuDao and menu sync. |
| 2 | **API layer** | Retrofit: `getLoyaltyMenu()`, `applyLoyalty(ApplyLoyaltyRequestModel)`, `getSmartPanelCustomerInfo(customerId)`. Implemented in `RetrofitAPIService`. |
| 3 | **Repository** | `SmartPanelRepository` / `SmartPanelRepositoryImpl`: `getLoyaltyMenu`, `applyLoyalty`, `getSmartPanelCustomerInfo`, `clearCustomerInfo`, `clearMenuList`. In-memory cache: `menuItems`, `customersInfoList`. `MenuTicketRepository.getLoyaltyItem(itemId, modifierOptionId)` delegates to `GetAllMenuUseCase` / MenuDao. |
| 4 | **ViewModels** | `GuestBookViewModel`: exposes `loyaltyMenu` StateFlow, calls `getLoyaltyMenu()`. `SmartPanelSharedViewModel`: `customerInfo` StateFlow, `setCustomerInfo`, `triggerSmartPanelAction`, extension `LoyaltyItemModel.addLoyaltyItemOffline(...)` that uses MenuTicketRepository, GetAllMenuUseCase, SharedDataRepository. |
| 5 | **Loyalty menu UI** | `LoyaltyMenuDialogFragment`: collects `loyaltyMenu` and `menuItemsFlow`, shows loading/error/success; `LoyaltyMenuAdapter` with tier headers and items; on item click emits `OnLoyaltyItemClicked` and dismisses. |
| 6 | **Register integration** | QsrMenuFragment / TsrMenuFragment: `selectedLoyaltyModel`, subscribe to `smartPanelNav` for `OnLoyaltyItemClicked`, `OnLoyaltyApplied`, `OnLoyaltyAppliedQsrOffline`. `checkLoyalty()` calls `addLoyaltyItemOffline` and clears selection. Modifier panel shows “loyalty” context (e.g. `isLoyalty` flag). |
| 7 | **Secondary screen / CFD** | `LoyaltyPresentationAdapter`, `LoyaltySecondaryScreen` for secondary display. CFD: `MarketingCfdExtension` (e.g. `updateLoyaltyProfileView`, `showSignInLoyaltyView`) and related layouts. |
| 8 | **Domain / mappers** | `LoyaltyItemModel`, `ApplyLoyaltyRequestModel`, `BaseLoyaltyItem`, `CustomerModel` (loyalty fields). `LoyaltyMappers.kt`: API response → `BaseLoyaltyItem` list. |
| 9 | **Room** | `MenuDao`: `getLoyaltyItem`, `insertLoyaltyEntityAsList`, `loyaltyPointsUsed`, and ticket-item loyalty fields. `GetAllMenuUseCase`: `getLoyaltyItem`, `getLoyaltyItemModifiersTimeBaseMenu`, `isLoyaltyApplied`, `loyaltyPointsUsed`. |
| 10 | **Strings / analytics** | String resources for loyalty UI; analytics e.g. `Analytics().postLoyaltyItemSelected(...)`. New Relic logging (e.g. LOYALTY_APPLIED, GUEST_BOOK_CUSTOMER_INFO). |

### 2.3 Flow diagrams (reference)

**Loyalty item add-to-ticket (QSR-style):**  
Guest Book → Open Loyalty menu → getLoyaltyMenu API (or cache) → User taps item → OnLoyaltyItemClicked → Register sets selectedLoyaltyModel, opens modifier panel → User configures modifiers, taps Done → checkLoyalty() → addLoyaltyItemOffline → pushItemToDb → OnLoyaltyAppliedQsrOffline → UI updates → End.

**Customer info:**  
Ticket has customer → setCustomerInfo(customer, …) → getSmartPanelCustomerInfo API → cache → customerInfo StateFlow → LoyaltyMenuDialogFragment / CFD set points and tier → End.

**Button visibility (conceptual):**  
Loyalty menu entry is shown in Guest Book / SmartPanel when feature is available. “Print to KDS” style (Active tab, serveType, orderSource) does not govern Loyalty; visibility is by screen (Guest Book) and ticket/customer context.

---

## 3. Is There Any Tech Debt to Handle Before (or After) This Feature?

**Short answer:** No blocking tech debt for the feature as already implemented. The following are recommended improvements for maintainability and consistency; none block current behaviour.

- **Loyalty selection state in Fragment:** `selectedLoyaltyModel` and related flags live in `QsrMenuFragment` / `TsrMenuFragment` instead of a ViewModel. On configuration change, selection can be lost. **Recommendation:** Move to ViewModel or SavedStateHandle when touching this flow (backlog).
- **Business logic in Fragment:** `checkLoyalty()` and “when to call addLoyaltyItemOffline” live in the Fragment. **Recommendation:** Move decision and invocation into ViewModel when refactoring (backlog).
- **Unscoped coroutine in repository:** `SmartPanelRepositoryImpl.orderOutUpdateJob()` uses `CoroutineScope(Default).launch { while(true) { … } }` and is not cancelled with repository lifecycle. **Recommendation:** Fix by using a scope tied to repository or injected scope (should fix in a near-term iteration).
- **Duplicate flow collection in LoyaltyMenuDialogFragment:** Both `guestBookViewModel.loyaltyMenu` and `guestBookViewModel.menuItemsFlow` are collected and both drive `observedLoyaltyMenu()`. **Recommendation:** Use a single source (e.g. only `menuItemsFlow`) to avoid redundant updates (quick fix).
- **No unit/integration tests for Loyalty:** No loyalty-specific tests found. **Recommendation:** Add repository/ViewModel unit tests and critical-path integration tests (next sprint).
- **Extension function on domain in ViewModel:** `LoyaltyItemModel.addLoyaltyItemOffline()` is defined inside SmartPanelSharedViewModel. **Recommendation:** Optional refactor when touching loyalty flow (e.g. move to use case or repository).

**Optional (non-blocking):**  
If more “optional” actions per screen appear (e.g. multiple print-like actions), consider a small abstraction (e.g. secondary actions list) instead of hard-coding each. Not required for Loyalty.

---

## 4. Where Is the Impact in the Code?

### 4.1 Module / layer overview

| Layer / area | Impact | Files / components (indicative) |
|--------------|--------|----------------------------------|
| **Commons** | Loyalty entity for Room | `commons/.../datamodels/LoyaltyEntity.kt` |
| **POS – SmartPanel** | Repository, ViewModels, domain models, mappers, API DTOs | `smartPanel/data/repository/SmartPanelRepositoryImpl.kt`, `smartPanel/domain/repository/SmartPanelRepository.kt`, `smartPanel/presentation/.../viewmodel/GuestBookViewModel.kt`, `smartPanel/shared/SmartPanelSharedViewModel.kt`, `smartPanel/domain/model/guestBook/loyalties/*.kt`, `smartPanel/data/mapper/LoyaltyMappers.kt`, `smartPanel/data/remote/model/loyalty/*.kt` |
| **POS – SmartPanel UI** | Loyalty menu dialog, adapters, secondary screen | `smartPanel/.../loyaltyMenu/loyaltyMenuFragment/LoyaltyMenuDialogFragment.kt`, `LoyaltyMenuAdapter.kt`, `smartPanel/.../secondaryScreen/LoyaltyPresentationAdapter.kt`, `LoyaltySecondaryScreen.kt` |
| **POS – Register** | Loyalty selection and add-to-ticket in modifier flow | `ui/main/fragments/register/QsrMenuFragment.kt`, `TsrMenuFragment.kt` (selectedLoyaltyModel, checkLoyalty, OnLoyaltyItemClicked / OnLoyaltyApplied handling) |
| **POS – Room / Data** | Loyalty queries and ticket item loyalty fields | `room/dao/MenuDao.kt`, `room/repository/MenuRepository.kt`, `room/repository/MenuRepositoryImpl.kt`, `room/usecases/GetAllMenuUseCase.kt`, `repositories/menus/MenuTicketRepository.kt`, `MenuTicketRepositoryImpl.kt` |
| **POS – Network** | Loyalty and customer APIs | `network/RetrofitAPIService.kt` (getLoyaltyMenu, applyLoyalty, getSmartPanelCustomerInfo) |
| **POS – CFD / Marketing** | Loyalty profile and points on customer-facing display | `utils/MarketingCfdExtension.kt`, `marketingcfd/MarketingScreenRepoImpl.kt`, related layouts (e.g. `customer_facing_ticket.xml`) |
| **POS – Receipts / Print** | Loyalty data on receipts where applicable | `printers/PrinterTemplates.kt`, `printers/OnlineOrderReceipt.kt` (loyalty-related content if any) |
| **Resources** | Strings, layouts | `res/values/strings.xml`, `res/layout/dialog_loyality_menu.xml`, `loyalty_presentation_item.xml`, `loyalty_presentation.xml`, etc. |

### 4.2 File-level impact (concise)

- **commons/.../LoyaltyEntity.kt**  
  Room entity for `loyalty_tiers_table` (itemId, modifierOptionId, loyaltyId, loyaltyName, loyaltyTier). Used by menu sync and MenuDao.

- **pos/.../SmartPanelRepository.kt & SmartPanelRepositoryImpl.kt**  
  `getLoyaltyMenu(isFromMqtt)`, `applyLoyalty`, `getSmartPanelCustomerInfo`, `clearCustomerInfo`, `clearMenuList`. In-memory cache: `menuItems`, `customersInfoList`. API calls via Retrofit; safeApiCall and Resource.

- **pos/.../GuestBookViewModel.kt**  
  `loyaltyMenu` StateFlow, `getLoyaltyMenu()` (launches IO, emits Loading then result). Exposes `menuItemsFlow` from repository.

- **pos/.../SmartPanelSharedViewModel.kt**  
  `customerInfo` StateFlow, `setCustomerInfo`, `triggerSmartPanelAction`. Extension `LoyaltyItemModel.addLoyaltyItemOffline(allModifierList, specialRequest, guestType, ticketId, orderType)`: builds metadata and request, gets loyalty tier from MenuTicketRepository, calls makeItemReadyCache, pushItemToDb, then emits OnLoyaltyAppliedQsrOffline.

- **pos/.../LoyaltyMenuDialogFragment.kt**  
  Collects `loyaltyMenu` and `menuItemsFlow` (both drive same UI — consider removing one). Collects `customerInfo` for points/tier. LoyaltyMenuAdapter; on item click: analytics + triggerSmartPanelAction(OnLoyaltyItemClicked) + dismiss.

- **pos/.../QsrMenuFragment.kt & TsrMenuFragment.kt**  
  `selectedLoyaltyModel`, subscription to smartPanelNav for OnLoyaltyItemClicked / OnLoyaltyApplied / OnLoyaltyAppliedQsrOffline. `checkLoyalty()`: if selectedLoyaltyModel != null, call addLoyaltyItemOffline then clear. Modifier panel shows loyalty context (e.g. isLoyalty).

- **pos/.../MenuDao.kt**  
  `getLoyaltyItem(itemId, modifierOptionId)`, `insertLoyaltyEntityAsList`, `loyaltyPointsUsed(ticketId, posRequestId)`, and ticket-item loyalty-related queries.

- **pos/.../GetAllMenuUseCase.kt**  
  `getLoyaltyItem`, `getLoyaltyItemModifiersTimeBaseMenu`, `isLoyaltyApplied`, `loyaltyPointsUsed` (delegate to MenuRepository).

- **pos/.../RetrofitAPIService.kt**  
  `getLoyaltyMenu()`, `applyLoyalty(@Body)`, `getSmartPanelCustomerInfo(@Query customerId)`.

- **pos/.../LoyaltyMappers.kt**  
  Maps API loyalty response (e.g. LoyaltyData) to `ArrayList<BaseLoyaltyItem>` (tiers + LoyaltyItemModel).

- **Strings / layouts**  
  Add or reuse strings for loyalty menu, points, errors. Layouts: dialog_loyality_menu, loyalty_presentation_item, loyalty_presentation, CFD loyalty sections.

### 4.3 Existing code to reuse / not change

- **KDS print path:** `printKDSPrintReceiptConnection`, PrinterTemplates (KDS format), MQTT KDS print handling — unchanged by Loyalty. Loyalty does not trigger KDS print from POS; it only affects ticket items and customer display.
- **Ticket and modifier flows:** Existing ticket/item models, modifier panels, QsrModifierAdapter / QsrModifierGroupAdapter, and ticket DB push/pull are reused. Loyalty adds a “source” (loyalty item) and optional apply-loyalty API; core ticket and modifier logic stays.
- **TicketData / TicketItem:** Already contain or are extended for loyalty fields (e.g. isLoyaltyApplied, loyaltyPointsApplied). Use as-is for visibility and API payloads.
- **SharedDataRepository, MenuTicketRepository:** Used for selected ticket, posRequestId, pushItemToDb, getLoyaltyItem — no structural change required for Loyalty.

---

## 5. Acceptance Criteria (for QA)

- Loyalty menu opens from Guest Book (or designated entry); shows tier headers and loyalty items; loading and error states work.
- Tapping a loyalty item dismisses the dialog and Register (QSR/TSR) opens modifier panel with that item pre-selected; required modifiers are enforced where configured.
- On modifier Done, the loyalty item is added to the current ticket (offline for QSR; TSR as per product flow). Item appears on ticket with correct modifiers and loyalty tier/points where applicable.
- Customer info (points available/used, tier, eligible items) is shown when customer is set on ticket and after getSmartPanelCustomerInfo succeeds; CFD shows loyalty profile where configured.
- Apply-loyalty API (when used) updates ticket as per backend contract; POS reflects updated ticket state.
- Voided items are not included in loyalty application; fulfilled items are included per PRD/product rules.
- Receipts and relevant prints show loyalty-related data where applicable.
- No regression in non-loyalty flows (modifier panel, ticket list, payment, KDS print).

---

## 6. Dependencies

- **Backend:**  
  - **GET api/loyalty/pos-menu** — returns loyalty menu (tiers and items).  
  - **POST api/ticket/apply-loyalty** — applies loyalty to ticket (request/response per backend contract).  
  - **GET api/customer/smartPanel?customerId=** — returns customer info including loyalty points, tier, and eligible items.

- **Internal:** Menu DB (Room) with `loyalty_tiers_table` and ticket-item loyalty fields; menu sync must populate loyalty tiers. Ticket and modifier flows; SharedDataRepository; MQTT/ticket sync unchanged.

- **External:** Retrofit for HTTP; analytics and New Relic logging as per project standards.

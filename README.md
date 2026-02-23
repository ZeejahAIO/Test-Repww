# Android Technical Assessment & Engineering Governance Document  
## Loyalty Feature (Post-Implementation Review)

**Document Version:** 1.0  
**Assessment Date:** February 2025  
**Scope:** Existing Loyalty feature implementation across POS and SmartPanel modules.

---

## 1. Feature Overview

| Field | Value |
|-------|--------|
| **Jira Ticket** | _[To be filled if applicable]_ |
| **Feature Name** | Loyalty (Rewards / Loyalty Menu & Apply Loyalty) |
| **Module / Layer Affected** | UI (Register – QSR/TSR), SmartPanel (Guest Book, Secondary Screen), Data (Room, API), Cross-module |
| **Tech Lead / Owner** | _[To be filled]_ |
| **Estimated Complexity** | **L3** (multi-module, offline/online flows, DB + API) |
| **Initial Risk Level** | **Medium** |
| **Dependencies** | Internal: Menu DB, Ticket APIs, SmartPanel APIs; External: Retrofit (loyalty & customer APIs); Infra: New Relic logging |
| **Target Release Date** | _[Already released – assessment is retrospective]_ |

### Business Context

- **Problem being solved:** Allow guests to redeem loyalty rewards (tier-based items) from the POS and SmartPanel; support offline add-to-ticket for QSR and apply-loyalty flow for TSR; show customer loyalty info and points on customer-facing displays.
- **Expected user impact:** Staff can add loyalty items to tickets (QSR/TSR), view loyalty menu by tier, and see customer points/tiers; customers see loyalty profile and eligible items on CFD.
- **Success criteria:** Loyalty menu loads, items can be added offline/online, customer info and points display correctly, and loyalty tier/points are persisted and reflected on tickets and receipts.

---

## 2. Existing Code Review (Pre-Implementation / Current State)

### 2.1 Architecture & Layering Review

| Check | Finding |
|-------|---------|
| **ViewModel responsibility boundaries** | **GuestBookViewModel:** Thin; delegates loyalty menu to `SmartPanelRepository.getLoyaltyMenu()`, exposes `loyaltyMenu` and `menuItemsFlow`. **SmartPanelSharedViewModel:** Handles customer info, smart panel actions, and **loyalty add-to-ticket** via extension `LoyaltyItemModel.addLoyaltyItemOffline()`; coordinates `MenuTicketRepository`, `GetAllMenuUseCase`, `SharedDataRepository`. Latter ViewModel carries significant loyalty + ticket logic. |
| **Repository abstraction** | **SmartPanelRepository** defines `getLoyaltyMenu`, `applyLoyalty`, `getSmartPanelCustomerInfo`, `clearCustomerInfo`, `clearMenuList`. **SmartPanelRepositoryImpl** implements with Retrofit and in-memory cache (`menuItems`, `customersInfoList`). **MenuTicketRepository** exposes `getLoyaltyItem(itemId, modifierOptionId)` for local DB (Room). Layering is present but repository holds mutable in-memory state. |
| **UseCase / Domain isolation** | **GetAllMenuUseCase** exposes `getLoyaltyItem`, `getLoyaltyItemModifiersTimeBaseMenu`, `isLoyaltyApplied`, `loyaltyPointsUsed` (delegate to MenuRepository). Domain models: `LoyaltyItemModel`, `ApplyLoyaltyRequestModel`, `CustomerModel` (with loyalty fields). No dedicated “LoyaltyUseCase”; loyalty flows are split across ViewModel, repository, and Fragment. |
| **Separation of concerns** | **Violation:** `selectedLoyaltyModel` and loyalty “flow” state (e.g. when to call `checkLoyalty()`) live in **QsrMenuFragment** and **TsrMenuFragment** (UI layer). Business rule “add loyalty item to ticket” is triggered from Fragment but executed in ViewModel via extension. Fragment knows about `SmartPanelActions.OnLoyaltyItemClicked`, `OnLoyaltyApplied`, `OnLoyaltyAppliedQsrOffline`. |
| **Cross-module dependency violations** | Register (QSr/TSR) depends on SmartPanel’s `SmartPanelSharedViewModel`, `SmartPanelActions`, and `LoyaltyItemModel`. SmartPanel depends on `MenuTicketRepository`, `GetAllMenuUseCase`, `SharedDataRepository` (register/ticket). No clear “loyalty domain” module; loyalty is spread across pos, smartPanel, commons. |
| **DI graph integrity** | Hilt used (`@AndroidEntryPoint`, `@HiltViewModel`, `@Inject`). SmartPanelRepositoryImpl receives `RetrofitAPIService`, `BusinessIdUseCase`, `TsrDao`; ViewModels receive repositories/use cases. No circular dependency observed. |

### 2.2 Concurrency & Coroutine Audit

| Check | Finding |
|-------|---------|
| **Dispatcher correctness** | ViewModels use `viewModelScope.launch(IO)` or `Default` for loyalty/customer work; Fragment uses `lifecycleScope` and `repeatOnLifecycle` / `flowWithLifecycle`. Loyalty mapper uses `withContext(IO)` for mapping (mapping is CPU-bound; IO not strictly necessary). |
| **Structured concurrency** | ViewModel and Fragment scopes used; no `GlobalScope` in loyalty paths. |
| **SupervisorJob misuse** | Not observed in loyalty code. |
| **Unscoped coroutines** | **Issue:** `SmartPanelRepositoryImpl.orderOutUpdateJob()` uses `CoroutineScope(Default).launch { while(true) { … } }` — **unscoped**; survives repository lifetime and is not cancelled when repository is no longer needed. |
| **Flow cold/hot misuse** | `loyaltyMenu` and `menuItemsFlow` are StateFlows (hot). **Issue:** In **LoyaltyMenuDialogFragment**, both `guestBookViewModel.loyaltyMenu` and `guestBookViewModel.menuItemsFlow` are collected and **both** drive `observedLoyaltyMenu()` — redundant and can cause duplicate UI updates. |
| **Backpressure / Cancellation** | StateFlow/SharedFlow used appropriately; cancellation follows lifecycle. |
| **Cancellation awareness** | Suspend functions used; no blocking calls in loyalty path in main thread. |

### 2.3 Data Layer Review

| Check | Finding |
|-------|---------|
| **Room queries** | `MenuDao.getLoyaltyItem(itemId, modifierOptionId)` — single row by composite key. `loyaltyPointsUsed`, `isLoyaltyApplied` on ticket items. `insertLoyaltyEntityAsList` for bulk insert. Queries are simple. |
| **Index usage** | `LoyaltyEntity` table `loyalty_tiers_table` with primary key `(itemId, modifierOptionId)` — index implied by primary key. No explicit index audit for other loyalty-related queries. |
| **N+1 / Transactions** | No N+1 observed in loyalty-specific code. Ticket item updates use existing transaction patterns. |
| **Migration readiness** | `LoyaltyEntity` in `MenuDatabase`; migrations not inspected in this review. |
| **API error mapping** | `safeApiCall` used in repository; returns `Resource.Success` / `Resource.Error`. Errors propagated to UI via StateFlow. |

### 2.4 UI & State Management Review

| Check | Finding |
|-------|---------|
| **Single source of truth** | Loyalty menu: repository holds cached `menuItems` and exposes via `menuItemsFlow` and `getLoyaltyMenu()`. **selectedLoyaltyModel** is Fragment state only — **not** in ViewModel — so no single source of truth for “currently selected loyalty item” across configuration change. |
| **Immutable UI state** | `LoyaltyItemModel` has `var modifierGroup`, `var modifiers`, `var itemTaxes` — mutable domain model. UI state in Fragment is mutable (`selectedLoyaltyModel`). |
| **StateFlow/LiveData** | StateFlow used for `loyaltyMenu`, `customerInfo`, `menuItemsFlow`; SharedFlow for `smartPanelNav`. No LiveData in loyalty flow. |
| **One-time events** | `SmartPanelActions` (e.g. `OnLoyaltyItemClicked`, `OnLoyaltyApplied`) delivered via SharedFlow — appropriate for one-time events. |
| **Configuration change safety** | **Risk:** `selectedLoyaltyModel` and loyalty-related flags in QsrMenuFragment/TsrMenuFragment are lost on configuration change unless saved in savedStateHandle or ViewModel. |
| **Compose** | Not used for loyalty UI; traditional Views/Fragments. |

### 2.5 Performance & Memory Review

| Check | Finding |
|-------|---------|
| **Large ViewModels** | SmartPanelSharedViewModel holds loyalty add-to-ticket logic and multiple repositories; size is moderate but could grow. |
| **Memory leaks** | Fragment clears `_binding` in `onDestroyView`. ViewModel uses viewModelScope; no obvious context retention in loyalty code. |
| **Heavy object allocation** | Loyalty menu cached in repository (`ArrayList<BaseLoyaltyItem>`); customer list cached (`customersInfoList`). Could grow with many customers/menu items. |
| **Bitmap handling** | Loyalty item images loaded in adapters (e.g. `LoyaltyPresentationAdapter`, `LoyaltyMenuAdapter`) — use of `load()` extension not audited for caching. |
| **RecyclerView** | Loyalty menus use RecyclerView/GridLayoutManager; no obvious inefficiency. |
| **Cold start** | Loyalty not on critical cold-start path; getLoyaltyMenu can be called when Guest Book / loyalty UI is opened. |
| **Frame drops** | No heavy work on main thread in loyalty UI; withContext(Main) used for adapter updates. |

### 2.6 Reliability & Stability

| Check | Finding |
|-------|---------|
| **Crash / ANR reports** | Not available in codebase; recommend checking last 30–90 days for loyalty-related stack traces. |
| **Unhandled exceptions** | Repository uses `safeApiCall` and try/catch in ViewModel with logging; exceptions mapped to Resource.Error. |
| **Error propagation** | UI observes Resource.Loading / Success / Error and shows loading/error states in LoyaltyMenuDialogFragment. |
| **Retry logic** | No explicit retry for getLoyaltyMenu or applyLoyalty in reviewed code. |

### 2.7 Test Coverage Assessment

| Check | Finding |
|-------|---------|
| **Unit test coverage** | **No loyalty-specific unit tests** found (search for Loyalty* and loyalty in *Test* files returned no matches). |
| **Integration tests** | None found for loyalty flows. |
| **Flaky / edge-case tests** | N/A. |
| **Mocking** | N/A. |

---

## 3. Identified Issues Log

| # | Category | Issue Description | Severity | Risk Impact |
|---|----------|-------------------|----------|-------------|
| 1 | Architecture | **Loyalty selection state in Fragment:** `selectedLoyaltyModel` and related flags live in QsrMenuFragment/TsrMenuFragment instead of ViewModel. | **Medium** | State lost on config change; logic split between Fragment and ViewModel. |
| 2 | Architecture | **Business logic in Fragment:** `checkLoyalty()` in Fragment decides when to call ViewModel’s `addLoyaltyItemOffline` and manipulates modifier list. | **Medium** | Harder to test and reuse; Fragment is not a single responsibility. |
| 3 | Architecture | **Extension function on domain model in ViewModel:** `LoyaltyItemModel.addLoyaltyItemOffline()` is an extension defined in SmartPanelSharedViewModel; domain type is coupled to ViewModel. | **Low** | Unusual coupling; complicates testing and reuse. |
| 4 | Concurrency | **Unscoped coroutine in repository:** `SmartPanelRepositoryImpl.orderOutUpdateJob()` uses `CoroutineScope(Default).launch` with infinite loop. | **High** | Job never cancelled; potential leak and work after repository no longer needed. |
| 5 | UI / State | **Duplicate flow collection in LoyaltyMenuDialogFragment:** Both `loyaltyMenu` and `menuItemsFlow` are collected and both call `observedLoyaltyMenu()`. | **Medium** | Redundant updates; possible race or double UI refresh. |
| 6 | Data / State | **Mutable domain model:** `LoyaltyItemModel` has `var` for modifierGroup, modifiers, itemTaxes. | **Low** | Mutable shared state can lead to accidental mutation. |
| 7 | Data | **In-memory caches in repository:** `menuItems` and `customersInfoList` in SmartPanelRepositoryImpl never bounded; can grow. | **Low** | Memory growth in long sessions or many customers. |
| 8 | Reliability | **No retry for loyalty API calls.** | **Low** | Transient failures require user to retry manually. |
| 9 | Test | **No unit or integration tests for loyalty feature.** | **High** | Regressions and refactors are risky. |

---

## 4. Improvement Decision Matrix (Mandatory)

| Issue | Fix in Current Iteration? (Y/N) | Justification | Backlog Ticket | Target Sprint |
|-------|---------------------------------|---------------|----------------|---------------|
| 1 – selectedLoyaltyModel in Fragment | N | Requires refactor of QSR/TSR modifier flow and state ownership; high touch surface. | REFACTOR-LOYALTY-STATE | Backlog |
| 2 – checkLoyalty in Fragment | N | Tied to issue 1; move to ViewModel when state is moved. | REFACTOR-LOYALTY-STATE | Backlog |
| 3 – addLoyaltyItemOffline extension in ViewModel | N | Low severity; refactor when touching loyalty flow. | REFACTOR-LOYALTY-VM | Backlog |
| 4 – Unscoped orderOutUpdateJob | **Y** | Concurrency violation; must fix. Use scope tied to repository lifecycle or inject a scope. | FIX-ORDEROUT-SCOPE | Current/Next |
| 5 – Duplicate loyalty flow collection | **Y** | Quick fix; remove one collector or unify source. | FIX-LOYALTY-DIALOG-COLLECTOR | Current/Next |
| 6 – Mutable LoyaltyItemModel | N | Low impact; can be done with domain cleanup. | CLEANUP-LOYALTY-MODEL | Backlog |
| 7 – Unbounded in-memory caches | N | Monitor; add limits or TTL if issues appear. | PERF-LOYALTY-CACHE | Backlog |
| 8 – No retry for loyalty API | N | Enhancement; not blocking. | FEATURE-LOYALTY-RETRY | Backlog |
| 9 – No tests for loyalty | **Y** | Critical for maintainability; add at least repository/ViewModel unit tests. | TEST-LOYALTY-COVERAGE | Next sprint |

**Governance rules applied:**  
- Concurrency violation (issue 4) → **Must fix.**  
- Architectural issues (1, 2, 3) → Documented; fix planned in backlog with explicit approval.  
- Test coverage (9) → **Must address** in next sprint.

---

## 5. Impact Assessment

### 5.1 UI Impact

- **Layout changes:** Loyalty uses existing layouts (e.g. `dialog_loyality_menu.xml`, `loyalty_presentation_item.xml`, customer-facing loyalty views). No change required for this assessment.
- **Navigation:** Loyalty menu opened as dialog; loyalty item click triggers `SmartPanelActions.OnLoyaltyItemClicked` and dismiss. Navigation is Fragment/Activity based.
- **State handling:** Recommended to move `selectedLoyaltyModel` and loyalty flow flags into ViewModel (or dedicated LoyaltyViewModel) and restore on config change to avoid impact on user experience.

### 5.2 API Contract Impact

- **Endpoints used:** `GET api/loyalty/pos-menu`, `POST api/ticket/apply-loyalty`, `GET api/customer/smartPanel?customerId=`.
- **Request/response:** ApplyLoyaltyRequestModel, LoyaltyMenuResponse, SmartPanelCustomerInfoResponse. No change in this assessment.
- **Error model:** Handled via existing `ResultNew` / `Resource`; no versioning change.

### 5.3 Database Impact

- **Schema:** `loyalty_tiers_table` (LoyaltyEntity); ticket item tables store loyalty-related fields (e.g. loyaltyPointsApplied, isLoyaltyApplied).
- **Migration:** Not required for current assessment.
- **Rollback:** No schema change proposed; rollback of app version follows standard process.

### 5.4 Performance Impact

- **CPU:** Loyalty menu mapping and DB lookups are light; add-to-ticket flow does DB and repository work on IO dispatcher.
- **Memory:** In-memory loyalty menu and customer list in repository; recommend monitoring or capping in future.
- **Network:** Loyalty menu and customer info fetched on demand; apply-loyalty is on user action.
- **Startup:** Loyalty not on critical path.
- **Load:** Tied to number of loyalty menu items and concurrent customer info requests.

### 5.5 Backward Compatibility

- **Feature flags:** Not observed in loyalty code; any rollout would be app-level.
- **Legacy support:** Existing tickets and loyalty data must remain readable; no breaking change identified.

---

## 6. Risk Assessment

| Risk | Probability | Impact | Mitigation Strategy |
|------|--------------|--------|---------------------|
| Configuration change loses selected loyalty item | Medium | Medium | Move selection state to ViewModel / savedStateHandle; add backlog ticket. |
| Unscoped coroutine causes leak or unnecessary work | High | Medium | Fix orderOutUpdateJob scope (tied to repository or injected scope). |
| Duplicate flow collection causes UI glitches | Low | Low | Remove redundant collector in LoyaltyMenuDialogFragment. |
| No tests lead to regressions | High | High | Add unit tests for repository and ViewModel loyalty flows. |
| API or DB change breaks loyalty | Low | High | Maintain API contract; add integration tests for critical paths. |

**Rollback strategy:** Revert app version; no server-side loyalty contract change assumed.  
**Monitoring plan:** Use existing New Relic / logging for loyalty (e.g. LOYALTY_APPLIED, GUEST_BOOK_CUSTOMER_INFO); add alerts on loyalty API error rate and apply-loyalty failures if available.

---

## 7. Final Approval

| Role | Name | Approval |
|------|------|----------|
| Engineering Manager | _[To be filled]_ | |
| Platform Lead | _[To be filled]_ | |
| Squad Lead | _[To be filled]_ | |
| Product Acknowledgment | _[To be filled]_ | |
| **Date** | _[To be filled]_ | |

---

## Appendix: Key Files Referenced

- **UI:** `QsrMenuFragment.kt`, `TsrMenuFragment.kt`, `LoyaltyMenuDialogFragment.kt`, `LoyaltyPresentationAdapter.kt`, `LoyaltySecondaryScreen.kt`
- **ViewModels:** `SmartPanelSharedViewModel.kt`, `GuestBookViewModel.kt`
- **Repository:** `SmartPanelRepository.kt`, `SmartPanelRepositoryImpl.kt`, `MenuTicketRepository.kt`
- **Data:** `LoyaltyEntity.kt` (commons), `MenuDao.kt` (getLoyaltyItem, loyaltyPointsUsed, etc.), `GetAllMenuUseCase.kt`, `LoyaltyMappers.kt`
- **API:** `RetrofitAPIService.kt` (getLoyaltyMenu, applyLoyalty, getSmartPanelCustomerInfo)
- **Domain:** `LoyaltyItemModel.kt`, `ApplyLoyaltyRequestModel.kt`, `SmartPanelActions.kt`

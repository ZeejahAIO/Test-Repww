# Android Technical Assessment & Engineering Governance Document  
## Loyalty Feature (Rewards / Loyalty Menu & Apply Loyalty)

**Before Starting Development** please consider going over this document: *Android Best Practices.docx*

### Purpose

This document ensures that every Android feature implementation:

- Evaluates existing module quality before modification
- Preserves architectural integrity
- Prevents uncontrolled technical debt
- Assesses performance and memory impact
- Documents trade-offs explicitly
- Maintains long-term maintainability and scalability

This assessment is **mandatory** for all L3 and L4 complexity tickets and **recommended** for L2.

---

## 1. Feature Overview

- **Jira Ticket:** _[To be filled if applicable]_
- **Feature Name:** Loyalty (Rewards / Loyalty Menu & Apply Loyalty)
- **Module / Layer Affected:** UI (Register – QSR/TSR, SmartPanel Guest Book, Secondary Screen, CFD); Domain (SmartPanel domain models, loyalty use flows); Data (Room, Retrofit APIs); Cross-module (Register ↔ SmartPanel, commons)
- **Tech Lead / Owner:** _[To be filled]_
- **Estimated Complexity:** **L3** (multi-module, offline/online flows, DB + API, cross-module dependencies)
- **Initial Risk Level:** **Medium**
- **Dependencies:** Internal: Menu DB (Room), Ticket APIs, SmartPanel APIs, SharedDataRepository, MenuTicketRepository; External: Retrofit (GET loyalty/pos-menu, POST ticket/apply-loyalty, GET customer/smartPanel); Infra: New Relic logging
- **Target Release Date:** _[Already released – assessment is retrospective]_

### Business Context

- **Problem being solved:** Allow guests to redeem loyalty rewards (tier-based items) from POS and SmartPanel; support offline add-to-ticket for QSR and apply-loyalty flow for TSR; show customer loyalty info and points on customer-facing displays.
- **Expected user impact:** Staff can add loyalty items to tickets (QSR/TSR), view loyalty menu by tier, and see customer points/tiers; customers see loyalty profile and eligible items on CFD.
- **Success criteria:** Loyalty menu loads, items can be added offline/online, customer info and points display correctly, and loyalty tier/points are persisted and reflected on tickets and receipts.

---

## 2. Existing Code Review (Mandatory Pre-Implementation Step)

The feature owner must review the current state of the affected module before implementation.

### 2.1 Architecture & Layering Review

- **ViewModel responsibility boundaries:** GuestBookViewModel is thin (delegates loyalty menu to SmartPanelRepository, exposes loyaltyMenu and menuItemsFlow). SmartPanelSharedViewModel carries significant loyalty logic: customer info, smart panel actions, and loyalty add-to-ticket via extension `LoyaltyItemModel.addLoyaltyItemOffline()`; coordinates MenuTicketRepository, GetAllMenuUseCase, SharedDataRepository. Latter ViewModel is moderately large and mixes coordination with domain-like behaviour.
- **Repository abstraction correctness:** SmartPanelRepository defines getLoyaltyMenu, applyLoyalty, getSmartPanelCustomerInfo, clearCustomerInfo, clearMenuList; implementation uses Retrofit and in-memory cache (menuItems, customersInfoList). MenuTicketRepository exposes getLoyaltyItem(itemId, modifierOptionId) for Room. Abstraction is present but repository holds mutable in-memory state without bounds.
- **UseCase / Domain isolation:** GetAllMenuUseCase exposes getLoyaltyItem, getLoyaltyItemModifiersTimeBaseMenu, isLoyaltyApplied, loyaltyPointsUsed (delegate to MenuRepository). Domain models: LoyaltyItemModel, ApplyLoyaltyRequestModel, CustomerModel (loyalty fields). No dedicated LoyaltyUseCase; loyalty flows are split across ViewModel, repository, and Fragment.
- **Separation of concerns validation:** Violation: selectedLoyaltyModel and when to call checkLoyalty() live in QsrMenuFragment and TsrMenuFragment (UI layer). Business rule “add loyalty item to ticket” is triggered from Fragment but executed in ViewModel via extension. Fragment knows about SmartPanelActions (OnLoyaltyItemClicked, OnLoyaltyApplied, OnLoyaltyAppliedQsrOffline).
- **Cross-module dependency violations:** Register (QSR/TSR) depends on SmartPanel’s SmartPanelSharedViewModel, SmartPanelActions, LoyaltyItemModel. SmartPanel depends on MenuTicketRepository, GetAllMenuUseCase, SharedDataRepository. No clear loyalty domain module; loyalty is spread across pos, smartPanel, commons.
- **DI graph integrity:** Hilt used (@AndroidEntryPoint, @HiltViewModel, @Inject). SmartPanelRepositoryImpl receives RetrofitAPIService, BusinessIdUseCase, TsrDao; ViewModels receive repositories/use cases. No circular dependency observed.

### 2.2 Concurrency & Coroutine Audit

- **Dispatcher correctness:** ViewModels use viewModelScope.launch(IO) or Default for loyalty/customer work; Fragment uses lifecycleScope, repeatOnLifecycle, flowWithLifecycle. Loyalty mapper uses withContext(IO) for mapping (mapping is CPU-bound; IO not strictly necessary).
- **Structured concurrency:** ViewModel and Fragment scopes used; no GlobalScope in loyalty paths.
- **SupervisorJob misuse:** Not observed in loyalty code.
- **Unscoped coroutines:** Issue: SmartPanelRepositoryImpl.orderOutUpdateJob() uses CoroutineScope(Default).launch { while(true) { … } } — unscoped; survives repository lifetime and is not cancelled when repository is no longer needed.
- **Flow cold/hot misuse:** loyaltyMenu and menuItemsFlow are StateFlows (hot). Issue: In LoyaltyMenuDialogFragment, both guestBookViewModel.loyaltyMenu and guestBookViewModel.menuItemsFlow are collected and both drive observedLoyaltyMenu() — redundant and can cause duplicate UI updates.
- **Backpressure handling:** StateFlow/SharedFlow used appropriately; cancellation follows lifecycle.
- **Cancellation awareness:** Suspend functions used; no blocking calls in loyalty path on main thread.

### 2.3 Data Layer Review

- **Room queries performance:** MenuDao.getLoyaltyItem(itemId, modifierOptionId) — single row by composite key. loyaltyPointsUsed, isLoyaltyApplied on ticket items. insertLoyaltyEntityAsList for bulk insert. Queries are simple.
- **Index usage validation:** LoyaltyEntity table loyalty_tiers_table has primary key (itemId, modifierOptionId) — index implied. No explicit index audit for other loyalty-related queries.
- **N+1 query risks:** No N+1 observed in loyalty-specific code.
- **Transaction correctness:** Ticket item updates use existing transaction patterns; no loyalty-specific transaction issues identified.
- **Migration readiness:** LoyaltyEntity in MenuDatabase; migrations not inspected in this review.
- **API error mapping:** safeApiCall used in repository; returns Resource.Success / Resource.Error. Errors propagated to UI via StateFlow.

### 2.4 UI & State Management Review

- **Single source of truth validation:** Loyalty menu: repository holds cached menuItems and exposes via menuItemsFlow and getLoyaltyMenu(). selectedLoyaltyModel is Fragment state only — not in ViewModel — so no single source of truth for “currently selected loyalty item” across configuration change.
- **Immutable UI state usage:** LoyaltyItemModel has var modifierGroup, var modifiers, var itemTaxes — mutable domain model. UI state in Fragment is mutable (selectedLoyaltyModel).
- **StateFlow/LiveData misuse:** StateFlow used for loyaltyMenu, customerInfo, menuItemsFlow; SharedFlow for smartPanelNav. No LiveData in loyalty flow. Duplicate collection of loyalty menu flows in LoyaltyMenuDialogFragment is a misuse (two sources driving same UI).
- **One-time event handling:** SmartPanelActions (e.g. OnLoyaltyItemClicked, OnLoyaltyApplied) delivered via SharedFlow — appropriate for one-time events.
- **Configuration change safety:** Risk: selectedLoyaltyModel and loyalty-related flags in QsrMenuFragment/TsrMenuFragment are lost on configuration change unless saved in SavedStateHandle or ViewModel.
- **Compose recomposition risks:** Not applicable; traditional Views/Fragments used for loyalty UI.

### 2.5 Performance & Memory Review

- **Large ViewModels:** SmartPanelSharedViewModel holds loyalty add-to-ticket logic and multiple repositories; size is moderate but could grow.
- **Memory leaks:** Fragment clears _binding in onDestroyView. ViewModel uses viewModelScope; no obvious context retention in loyalty code.
- **Heavy object allocation:** Loyalty menu cached in repository (ArrayList<BaseLoyaltyItem>); customer list cached (customersInfoList). Could grow with many customers/menu items; no bounds.
- **Bitmap handling:** Loyalty item images loaded in adapters (LoyaltyPresentationAdapter, LoyaltyMenuAdapter) via load() extension; caching not audited.
- **RecyclerView inefficiencies:** Loyalty menus use RecyclerView/GridLayoutManager; no obvious inefficiency.
- **Cold start impact:** Loyalty not on critical cold-start path; getLoyaltyMenu can be called when Guest Book / loyalty UI is opened.
- **Frame drops risk:** No heavy work on main thread in loyalty UI; withContext(Main) used for adapter updates.

### 2.6 Reliability & Stability

- **Crash reports (last 30–90 days):** Not available in codebase; recommend checking for loyalty-related stack traces.
- **ANR reports:** Not available in codebase.
- **Unhandled exceptions:** Repository uses safeApiCall and try/catch in ViewModel with logging; exceptions mapped to Resource.Error.
- **Error propagation strategy:** UI observes Resource.Loading / Success / Error and shows loading/error states in LoyaltyMenuDialogFragment.
- **Retry logic correctness:** No explicit retry for getLoyaltyMenu or applyLoyalty in reviewed code; transient failures require user to retry manually.

### 2.7 Test Coverage Assessment

- **Unit test coverage %:** No loyalty-specific unit tests found; effective coverage for loyalty flows is 0%.
- **Integration test presence:** None found for loyalty flows.
- **Flaky tests:** N/A.
- **Missing edge-case tests:** All loyalty edge cases (menu load failure, apply failure, customer info failure, config change) are untested.
- **Mocking anti-patterns:** N/A (no tests to evaluate).

---

## 3. Identified Issues Log

Document all findings clearly.

| Category | Issue Description | Severity | Risk Impact |
|----------|-------------------|----------|-------------|
| Architecture | Loyalty selection state in Fragment: selectedLoyaltyModel and related flags live in QsrMenuFragment/TsrMenuFragment instead of ViewModel. | Medium | State lost on config change; logic split between Fragment and ViewModel. |
| Architecture | Business logic in Fragment: checkLoyalty() in Fragment decides when to call ViewModel’s addLoyaltyItemOffline and manipulates modifier list. | Medium | Harder to test and reuse; Fragment not single responsibility. |
| Architecture | Extension function on domain model in ViewModel: LoyaltyItemModel.addLoyaltyItemOffline() is an extension defined in SmartPanelSharedViewModel; domain type coupled to ViewModel. | Low | Unusual coupling; complicates testing and reuse. |
| Concurrency | Unscoped coroutine in repository: SmartPanelRepositoryImpl.orderOutUpdateJob() uses CoroutineScope(Default).launch { while(true) { … } }. | High | Job never cancelled; potential leak and work after repository no longer needed. |
| UI / State | Duplicate flow collection in LoyaltyMenuDialogFragment: both loyaltyMenu and menuItemsFlow are collected and both call observedLoyaltyMenu(). | Medium | Redundant updates; possible race or double UI refresh. |
| Data / State | Mutable domain model: LoyaltyItemModel has var for modifierGroup, modifiers, itemTaxes. | Low | Mutable shared state can lead to accidental mutation. |
| Data | In-memory caches in repository: menuItems and customersInfoList in SmartPanelRepositoryImpl never bounded; can grow. | Low | Memory growth in long sessions or many customers. |
| Reliability | No retry for loyalty API calls. | Low | Transient failures require user to retry manually. |
| Test | No unit or integration tests for loyalty feature. | High | Regressions and refactors are risky. |

---

## 4. Improvement Decision Matrix (Mandatory)

For each identified issue:

| Issue | Fix in Current Iteration? (Y/N) | Justification | Backlog Ticket | Target Sprint |
|-------|---------------------------------|---------------|----------------|---------------|
| selectedLoyaltyModel in Fragment | N | Requires refactor of QSR/TSR modifier flow and state ownership; high touch surface. | REFACTOR-LOYALTY-STATE | Backlog |
| checkLoyalty in Fragment | N | Tied to issue above; move to ViewModel when state is moved. | REFACTOR-LOYALTY-STATE | Backlog |
| addLoyaltyItemOffline extension in ViewModel | N | Low severity; refactor when touching loyalty flow. | REFACTOR-LOYALTY-VM | Backlog |
| Unscoped orderOutUpdateJob | **Y** | Concurrency violation; must fix. Use scope tied to repository lifecycle or inject a scope. | FIX-ORDEROUT-SCOPE | Current/Next |
| Duplicate loyalty flow collection | **Y** | Quick fix; remove one collector or unify source. | FIX-LOYALTY-DIALOG-COLLECTOR | Current/Next |
| Mutable LoyaltyItemModel | N | Low impact; can be done with domain cleanup. | CLEANUP-LOYALTY-MODEL | Backlog |
| Unbounded in-memory caches | N | Monitor; add limits or TTL if issues appear. | PERF-LOYALTY-CACHE | Backlog |
| No retry for loyalty API | N | Enhancement; not blocking. | FEATURE-LOYALTY-RETRY | Backlog |
| No tests for loyalty | **Y** | Critical for maintainability; add at least repository/ViewModel unit tests. | TEST-LOYALTY-COVERAGE | Next sprint |

### Governance Rules

- **Crash, ANR, or memory leak** → Must fix immediately. (None identified as direct crash/ANR/leak; unscoped coroutine is a concurrency violation.)
- **Concurrency violation** → Must fix. (Issue 4: unscoped orderOutUpdateJob — **must fix.**)
- **Architectural violation** → Explicit approval required. (Issues 1, 2, 3 documented; fix planned in backlog with approval.)
- **Performance regression risk** → Must be benchmarked. (Unbounded caches documented; monitor; no immediate benchmark required.)
- **No issue may be ignored without documentation.** (All issues documented and dispositioned in this matrix.)

---

## 5. Impact Assessment

Evaluate downstream impact.

### 5.1 UI Impact

- **Layout changes?** Loyalty uses existing layouts (dialog_loyality_menu, loyalty_presentation_item, customer-facing loyalty views). No change required for this assessment.
- **Navigation changes?** Loyalty menu opened as dialog; loyalty item click triggers SmartPanelActions.OnLoyaltyItemClicked and dismiss. Navigation is Fragment/Activity based; no structural change.
- **State handling modifications?** Recommended: move selectedLoyaltyModel and loyalty flow flags into ViewModel or SavedStateHandle to survive configuration change and improve testability.

### 5.2 API Contract Impact

- **Request/response changes?** Endpoints used: GET api/loyalty/pos-menu, POST api/ticket/apply-loyalty, GET api/customer/smartPanel. No change in this assessment.
- **Error model changes?** Handled via existing ResultNew / Resource; no versioning change.
- **Versioning required?** No.

### 5.3 Database Impact

- **Schema changes?** loyalty_tiers_table (LoyaltyEntity); ticket item tables store loyalty-related fields (e.g. loyaltyPointsApplied, isLoyaltyApplied). No change in this assessment.
- **Migration needed?** Not required for current assessment.
- **Data backfill required?** No.
- **Rollback feasibility?** No schema change proposed; rollback of app version follows standard process.

### 5.4 Performance Impact

- **CPU impact:** Loyalty menu mapping and DB lookups are light; add-to-ticket flow does DB and repository work on IO dispatcher.
- **Memory footprint:** In-memory loyalty menu and customer list in repository; recommend monitoring or capping in future.
- **Network payload size:** Loyalty menu and customer info fetched on demand; apply-loyalty on user action; no disproportionate payload.
- **Startup time impact:** Loyalty not on critical cold-start path.
- **Expected load increase:** Tied to number of loyalty menu items and concurrent customer info requests; no blocking concern identified.

### 5.5 Backward Compatibility

- **Feature flags required?** Not observed in loyalty code; any rollout would be app-level.
- **Gradual rollout?** Per product decision; no technical blocker.
- **Legacy support maintained?** Existing tickets and loyalty data must remain readable; no breaking change identified.

---

## 6. Risk Assessment

| Risk | Probability | Impact | Mitigation Strategy |
|------|--------------|--------|---------------------|
| Configuration change loses selected loyalty item | Medium | Medium | Move selection state to ViewModel / SavedStateHandle; backlog ticket created. |
| Unscoped coroutine causes leak or unnecessary work | High | Medium | Fix orderOutUpdateJob scope (tied to repository or injected scope) — must fix. |
| Duplicate flow collection causes UI glitches | Low | Low | Remove redundant collector in LoyaltyMenuDialogFragment. |
| No tests lead to regressions | High | High | Add unit tests for repository and ViewModel loyalty flows (next sprint). |
| API or DB change breaks loyalty | Low | High | Maintain API contract; add integration tests for critical paths. |

### Rollback strategy

Revert app version; no server-side loyalty contract change assumed. If backend contract changes, coordinate with backend team for backward compatibility or versioned endpoints.

### Monitoring plan

Use existing New Relic / logging for loyalty (e.g. LOYALTY_APPLIED, GUEST_BOOK_CUSTOMER_INFO). Add or review alerts on loyalty API error rate and apply-loyalty failures if available. Monitor crash/ANR reports for loyalty-related stack traces (last 30–90 days).

---

## 7. Final Approval

| Role | Approval |
|------|----------|
| Engineering Manager Approval | _[To be filled]_ |
| Platform Lead Approval | _[To be filled]_ |
| Squad Lead Approval | _[To be filled]_ |
| Product Acknowledgment | _[To be filled]_ |
| **Date** | _[To be filled]_ |

---

## 8. Assessment Result

**Summary:** The Loyalty feature is implemented and functional across POS (QSR/TSR) and SmartPanel (Guest Book, customer info, CFD). The assessment confirms **L3 complexity** and **Medium** initial risk, with no blocking tech debt that prevents current operation.

**Mandatory actions:**

1. **Fix concurrency violation:** Unscoped `orderOutUpdateJob` in SmartPanelRepositoryImpl must be fixed (scope tied to repository or injected).
2. **Fix duplicate flow collection:** Remove or unify the duplicate loyalty menu collection in LoyaltyMenuDialogFragment.
3. **Add test coverage:** Add at least repository and ViewModel unit tests for loyalty flows in the next sprint.

**Backlog (documented, not ignored):**

- Move loyalty selection state and checkLoyalty logic from Fragment to ViewModel (REFACTOR-LOYALTY-STATE).
- Refactor addLoyaltyItemOffline out of ViewModel extension (REFACTOR-LOYALTY-VM).
- Consider immutable LoyaltyItemModel and bounded caches (CLEANUP-LOYALTY-MODEL, PERF-LOYALTY-CACHE).
- Optional: retry for loyalty API (FEATURE-LOYALTY-RETRY).

**Governance compliance:** All identified issues are documented in the Issues Log and dispositioned in the Improvement Decision Matrix. Concurrency violation is marked for fix in current/next iteration; architectural items require explicit approval when scheduled. No issue is left undocumented.

**Result:** **Conditionally approved** — feature may remain in production; mandatory fixes (unscoped coroutine, duplicate collector, test coverage) must be completed as per matrix. Final sign-off to be recorded in Section 7.

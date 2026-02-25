# Loyalty Technical Assessment

Assessment scope: static code review of loyalty flow in `mpos` (`ViewModel`, dialog/UI flow, repository/API, menu modifier mapping, register/menu integration).

---

## 1) Feature Overview

- **Feature Name:** Loyalty menu selection + apply-to-ticket
- **Module / Layer Affected:** UI, ViewModel, Data/Repository, Network API, Room/DB mapping
- **Estimated Complexity:** L3
- **Initial Risk Level:** High
- **Dependencies:** Loyalty APIs, menu/modifier Room data, shared runtime constants

### Business Context

- Apply loyalty rewards to the active ticket based on customer points/tier.
- Show only eligible loyalty items and allow add/apply flow with modifiers.
- Keep ticket/payment totals and item states consistent after loyalty apply.

---

## 2) Existing Code Review (Pre-Implementation Check)

### 2.1 Architecture & Layering

**Observations**

- ViewModel + Repository abstraction exists.
- Hilt DI is used.
- Loyalty flow is coupled to global mutable state in `Constants`.

**Findings**

- `LoyaltyViewModel` directly mutates `Constants.ticketsByTableResponse` and `Constants._menuItemLiveData`.
- Shared global state increases cross-screen side effects and regression risk.
- No dedicated domain/use-case boundary for loyalty behavior.

### 2.2 Concurrency & Coroutine Audit

**Observations**

- Network calls are suspend-based and wrapped in `safeApiCall`.
- Modifier assembly for loyalty uses background dispatcher.

**Findings**

- Loyalty collectors in fragments use plain `lifecycleScope.launch { collect }` (not `repeatOnLifecycle`).
- `setLoyaltyAction()` dispatches another IO coroutine just to emit event.
- `MutableSharedFlow` without replay can drop events for late collectors.

### 2.3 Data Layer Review

**Observations**

- API contracts for `getLoyaltyMenu` and `applyLoyalty` are clear.
- Room queries exist for loyalty modifiers/groups.

**Findings**

- Apply failure collapses to `OnLoyaltyApplied(null)` without detailed error mapping.
- No retry/backoff strategy for loyalty apply.
- No fallback strategy for loyalty menu fetch failures.

### 2.4 UI & State Management Review

**Observations**

- Tier gating and stock checks are implemented.
- Dialog receiver register/unregister lifecycle handling exists.

**Findings**

- Tier points are set through global mutable `Constants.loyaltyPoints`.
- Apply success/failure UX is not clearly differentiated for users.
- Adapter updates rely on full refresh (`notifyDataSetChanged`).

### 2.5 Performance & Memory Review

**Findings**

- `notifyDataSetChanged()` on data/tier changes can cause unnecessary rebind/repaint.
- Modifier filtering per selection may become costly on large menus.
- No obvious leak in loyalty dialog binding cleanup.

### 2.6 Reliability & Stability

**Findings**

- Silent loyalty apply failures increase operational/support risk.
- Fallback IDs (`-1`) can appear in request construction if state is inconsistent.
- Limited explicit telemetry around loyalty failure paths.

### 2.7 Test Coverage Assessment

**Findings**

- Loyalty-specific unit/integration tests are effectively absent.
- Missing edge-case coverage: tier boundaries, apply failure, required modifier flows, inter-fragment event ordering.

---

## 3) Identified Issues Log

| Category | Issue Description | Severity | Risk Impact |
|---|---|---|---|
| Architecture | Global mutable state (`Constants`) used as loyalty state bus | High | Cross-screen side effects, hard debugging |
| Reliability | Apply failure mapped to null without explicit UX message | High | Failed reward attempts appear as no-op |
| Concurrency/Lifecycle | Collectors not lifecycle-gated with `repeatOnLifecycle` | Medium | UI updates while stopped, race edges |
| Performance | Broad adapter refresh via `notifyDataSetChanged()` | Medium | Jank risk with larger lists |
| Error handling | Detailed API error not surfaced to UI | High | No actionable user/operator feedback |
| Testing | No meaningful loyalty automation | High | High regression probability |

---

## 4) Improvement Decision Matrix (Mandatory)

| Issue | Fix in Current Iteration? (Y/N) | Justification | Backlog Ticket | Target Sprint |
|---|---|---|---|---|
| Distinct loyalty apply failure handling + UX | Y | User-facing reliability issue | LOY-101 | Current |
| Reduce dependence on `Constants` for loyalty flow | Y (partial) | Architectural debt with direct regressions | LOY-102 | Current + Next |
| Use `repeatOnLifecycle` for loyalty collectors | Y | Concurrency governance must-fix | LOY-103 | Current |
| Replace full adapter refresh with DiffUtil/ListAdapter | N | Optimization, lower immediate risk | LOY-104 | Next |
| Add loyalty unit/integration tests | Y | Required for safe refactor | LOY-105 | Current |
| Add retries + improved telemetry for loyalty apply | N (timeboxed) | Valuable but secondary to correctness | LOY-106 | Next |

### Governance Rules Compliance

- Concurrency violation: must fix.
- Reliability degradation: must fix.
- Architectural violation: either fix now or document approved phased remediation.

---

## 5) Impact Assessment

### 5.1 UI Impact

- Add explicit success/failure state messaging for apply.
- Keep rewards eligibility and disabled states synchronized with latest ticket data.

### 5.2 API Contract Impact

- No backend contract changes required.
- Improve client-side error propagation and mapping.

### 5.3 Database Impact

- No schema change required for immediate fixes.

### 5.4 Performance Impact

- Lifecycle and error fixes: neutral to positive.
- Adapter diffing (future): positive UI rendering performance.

### 5.5 Backward Compatibility

- Low risk if phased carefully.
- Feature-flag optional for larger state refactor.

---

## 6) Risk Assessment

| Risk | Probability | Impact | Mitigation Strategy |
|---|---|---|---|
| Apply appears to do nothing on failure | Medium | High | Surface explicit error action and UI feedback |
| Fragment-state inconsistency | Medium | High | Move to single source of truth in ViewModel |
| Regression during refactor | High | High | Add tests before and during migration |
| UI jank with larger loyalty list | Medium | Medium | Use diff-based adapter updates |

### Rollback Strategy

- Keep API contracts unchanged.
- Roll back loyalty UI/state refactor independently if post-release instability appears.

### Monitoring Plan

- Track `getLoyaltyMenu` and `applyLoyalty` success/failure counts.
- Add failure reason logging and latency metrics.
- Add crash breadcrumbs around loyalty click/apply actions.

---

## Final Approval

- Engineering Manager Approval: _Pending_
- Platform Lead Approval: _Pending_
- Squad Lead Approval: _Pending_
- Product Acknowledgment: _Pending_
- Date: _Pending_


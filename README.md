# Android POS – HTTP Endpoints & Resources by Screen

This document lists all HTTP endpoints used by the Android POS app, grouped by API source and resource, with a screen-wise mapping of which screens trigger which endpoints.

---

## 1. API Sources

| Source | File | Base URL / Notes |
|--------|------|------------------|
| **RetrofitAPIService** | `pos/.../network/RetrofitAPIService.kt` | Main backend API |
| **RetrofitApiServiceB** | `pos/.../network/RetrofitApiServiceB.kt` | Menu, dashboard, splash |
| **PaymentApiServiceAdyen** | `pos/.../network/PaymentApiServiceAdyen.kt` | Adyen payment (15s timeout) |
| **RetrofitStripeAPIService** | `pos/.../network/RetrofitStripeAPIService.kt` | Stripe API (`https://api.stripe.com/v1/`) |
| **OnRestaurantByDeviceApi** | `pos/.../network/OnRestaurantByDeviceApi.kt` | Device → restaurant lookup |

---

## 2. Endpoints by Resource

### 2.1 Authentication

| Method | Path | Resource | Used by |
|--------|------|----------|---------|
| POST | `/api/authentication/login-with-pos` | Login with PIN (deviceId, logoutNeeded, VerifyPin) | **ClockInActivity**, ClockInFragment |
| POST | `/api/authentication/logout` | Logout | ClockInViewModel |
| POST | `/api/authentication/approve-with-pin` | Manager approval with PIN | ClockInViewModel |
| POST | `api/authentication/refresh-token` | Refresh token (x-id-token header) | SupportInterceptor (global) |

---

### 2.2 Table

| Method | Path | Resource | Used by |
|--------|------|----------|---------|
| GET | `/api/table/my-tables` | My tables (TSR) | **TsrMenuFragment**, Table/MyTables flow |
| PATCH | `api/table/change-table-pos` | Change table (ChangeTableRequest) | ApiQueueingRemoteSource, TicketViewModel |
| GET | `/api/table/ticket/{tableId}` | Tickets by table | TicketViewModel, Register flow |
| PATCH | `/api/table/free/table` | Free/empty table | TicketViewModel |
| GET | `/api/table/floor/restaurant` | All floors | **TableLayoutBuilderFragment**, FloorTableViewModel |
| GET | `/api/table/floor/{floorId}` | Single floor + tables | TableBuilderViewModel |
| POST | `/api/table/floor` | Create floor (FloorRequest) | TableBuilderViewModel |
| PATCH | `/api/table/bulk-table/{floorId}` | Save floor tables (SaveTableRequest) | TableBuilderViewModel |
| DELETE | `/api/table/floor/{floorId}` | Delete floor | TableBuilderViewModel |
| DELETE | `/api/table/{tableId}` | Delete table | TableBuilderViewModel |
| GET | `/api/table/tableNo` | Table numbers | TableBuilderViewModel |
| GET | `api/table/get-tables-status/{restaurantId}` | Tables status | GetTableStatusRepo, **TableLayoutViewFragment** |

---

### 2.3 Ticket

| Method | Path | Resource | Used by |
|--------|------|----------|---------|
| POST | `/api/ticket/createId` | Create ticket ID | ApiQueueingRemoteSource, TicketsRepository |
| GET | `/api/ticket/{ticketId}` | Get ticket by ID | TicketViewModel, MenuViewModelForRoom, Payment flow |
| PATCH | `api/ticket/update/{ticketId}` | Update ticket (TicketRequest) | TicketsRepository, ApiQueueingRemoteSource |
| PATCH | `api/ticket/remove/ticket-item/{ticketId}` | Remove ticket item(s) | ApiQueueingRemoteSource |
| PATCH | `/api/ticket/update-status/{ticketId}` | Update ticket item status | TicketsRepository |
| PATCH | `/api/ticket/voidorder` | Void order (orderId, ApiSequencingRequest) | ApiQueueingRemoteSource |
| POST | `/api/ticket/cancel-scheduler-order/{ticketId}` | Cancel scheduled order | TicketsRepository |
| POST | `/api/ticket/manual-fire-kds-order/{ticketId}` | Fire KDS order (isThirdPartyOrder) | TicketsRepository |
| DELETE | `/api/ticket/items/{ticketId}` | Remove ticket item (ticketItemId) | TicketsRepository |
| HTTP DELETE | `/api/ticket/items/{ticketId}` | Remove custom item (customItemId) | TicketsRepository |
| PATCH | `/api/ticket/split-by-order/{orderId}` | Split ticket (SplitTicketRequestModel) | TicketsRepository |
| PATCH | `/api/ticket/update/{ticketId}` | Apply discount (DiscountRequest) | TicketsRepository |
| DELETE | `/api/ticket/order/empty-tickets` | Delete empty tickets (orderId) | TicketsRepository |
| PATCH | `/api/ticket/delete-unsent-items/{orderId}` | Delete pending/empty ticket items | TicketsRepository |
| GET | `/api/ticket/get/allorders` | All orders (orderType, orderStatus, page, perPage, search, tableId) | **OrdersHubTicketsFragment**, OrdersHubViewModel |
| GET | `/api/ticket/all/filter` | Filtered tickets (assignedTo, orderTracking, orderType, orderSource, paymentStatus, page, perPage) | **OrdersHubTicketsFragment**, OrdersFiltersViewModel |
| GET | `api/ticket/splash/alltickets/v2` | Splash – all tickets v2 | **SplashActivity** (RetrofitApiServiceB) |
| GET | `api/ticket/count/un-closed` | Unclosed ticket count | DashBoardRepository |
| GET | `api/ticket/get-pending-tickets-status` | Pending tickets status | TicketsRepository |
| GET | `api/ticket/kds-pos-print/{ticketId}` | KDS print data | TicketsRepository |
| GET | `/api/ticket/order-delay` | Order delay | TicketsRepository |
| PATCH | `/api/ticket/order-delay` | Update order delay | TicketsRepository |
| POST | `/api/ticket/add-new-customer/{ticketId}` | Add customer (AddCustomerRequest) | **TemporaryGuestFragment**, TicketViewModel |
| PATCH | `api/ticket/add-existing-customer-to-ticket/{ticketId}` | Add existing customer to ticket | TicketsRepository |
| POST | `api/ticket/ticket-item-discount/{ticketId}` | Item-level discounts | TicketsRepository |
| PATCH | `api/ticket/update-note/{ticketId}` | Update ticket note | ApiQueueingRemoteSource |
| POST | `api/ticket/add-or-remove-ticket-discount/{ticketId}` | Apply ticket discount (TsrDiscounts) | ApiQueueingRemoteSource |
| PATCH | `api/ticket/update-no-of-guests/{ticketId}` | Update guest count | ApiQueueingRemoteSource, TsrTicketViewModel |
| PATCH | `api/ticket/update-ticket-on-hold/{ticketId}` | Set ticket on hold | ApiQueueingRemoteSource |
| PATCH | `api/ticket/update-orderType/{orderId}` | Update order type | ApiQueueingRemoteSource |
| PATCH | `api/ticket/update-taxExemption/{orderId}` | Tax exemption | ApiQueueingRemoteSource |
| PATCH | `api/ticket/update-service-charges/{ticketId}` | Update service charges | ApiQueueingRemoteSource |
| PATCH | `api/ticket/service-charges/{ticketId}` | Set service charges (ServiceChargeRequest) | TicketsRepository |
| POST | `api/ticket/qsr-create/{ticketId}` | QSR create ticket | ApiQueueingRemoteSource |
| POST | `api/ticket/tsr-create/{orderId}` | TSR create ticket (list TsrTicketRequest) | ApiQueueingRemoteSource |
| POST | `api/ticket/split-tickets-offline/{orderId}` | TSR split offline | TsrRepositoryImpl |
| PATCH | `api/ticket/move-check-pos` | Move check (MoveCheckRequest) | MoveCheckRemoteSource |
| PATCH | `api/ticket/move-check-offline` | Move check offline (MoveTicketRequest) | ApiQueueingRemoteSource |
| PATCH | `api/ticket/undo-split-mpos/{orderId}` | Undo split (MPOS) | UndoSplitRemoteSource |
| PATCH | `api/ticket/undo-split-items/{orderId}` | Undo split items (UndoTicketItemRequest) | UndoSplitRemoteSource |
| POST | `api/ticket/undo-split-tickets-offline/{orderId}` | Undo split offline (UndoTicketRequest) | ApiQueueingRemoteSource |
| PATCH | `api/ticket/move-item-offline` | Move items offline | ApiQueueingRemoteSource |
| POST | `api/payment/verify-pos-payment-send-email/{ticketId}` | Verify payment & send email | TicketsRepository |
| POST | `/api/ticket/add-customer` | Add customer (marketing) | MarketingScreenRepoImpl |

---

### 2.4 Payment

| Method | Path | Resource | Used by |
|--------|------|----------|---------|
| POST | `api/payment/v2/calculation/{id}` | Payment calculation v2 (PaymentIntentRequest) | **PaymentParentFragment**, PayByCash/Tender/GiftCard |
| POST | `api/payment/{ticketId}` | Post payment (PaymentPost) – cash/gift/tender | PaymentRepository |
| POST | `api/payment/calculation/{id}` | Payment calculation (legacy) | PaymentRepository |
| POST | `api/payment/intent` | Payment intent info | PaymentRepository |
| POST | `/api/payment/new/intent/v2` | Create intent v2 | PaymentRepository |
| POST | `api/payment/process-payment` | Process payment (Adyen PaymentIntentRequestAdyen) | PaymentRepository (Adyen) |
| POST | `api/payment/cardpaymenttrigger/{ticketId}` | Card payment trigger | PaymentRepository |
| POST | `api/payment/refund/{ticketId}` | Cash refund (CashRefundPost) | PaymentRepository, **RefundMainFragment** |
| GET | `api/payment/{ticketId}` | Get payment ID | TicketsRepository |
| GET | `api/payment/refund/{ticketId}` | Get refund payment ID | PaymentRepository |
| GET | `api/tender/get-active-tenders` | Active tenders | PaymentRepository |
| POST | `api/payment/connectiontoken` | Stripe connection token | PaymentRepository (Stripe reader) |
| POST | `/api/webhook/{intentId}` | Payment webhook status (type, intentType, retryAttempt) | **AddTipUnclosedFragment**, PaymentViewModel |
| POST | `/api/payment/intent/cancel/v2` | Cancel payment intent | PaymentRepository |
| POST | `api/payment/intent/capture/v2` | Capture payment intent | **AddTipUnclosedFragment**, PaymentViewModel |
| POST | `/api/payment/receipt/{paymentId}` | Send receipt | TicketsRepository |
| POST | `api/payment/stripe/refund` | Refund (RefundRequest) | PaymentRepository |
| POST | `api/payment/cancel/{paymentId}` | Cancel payment | PaymentRepository |
| POST | `api/payment/adjust-payment/{paymentId}` | Adjust payment (AdjustPaymentRequest) | **AddTipUnclosedFragment**, PaymentViewModel |
| POST | `/api/payment/apply-deal` | Apply deal (DealRequest) | TicketsRepository |
| DELETE | `/api/payment/deals/{ticketId}/{dealId}` | Remove deal | TicketsRepository |
| POST | `api/payment/gift-card` | Verify gift card number (GiftCardRequest) | TsrRepositoryImpl, **PayByGiftCardFragment** |
| POST | `api/payment/gift-card` | Verify gift card + PIN (GiftCardPinRequest) | TsrRepositoryImpl |
| GET | `api/payment/ams/initiated/{ticketId}` | Adyen payment initiated check | PaymentRepository, PaymentApiServiceAdyen |
| POST | `/api/payment/move-payment` | Move payment (MovePaymentRequest) | PaymentRepository |

---

### 2.5 Stripe (External – api.stripe.com)

| Method | Path | Resource | Used by |
|--------|------|----------|---------|
| POST | `customers` | Get Stripe customer ID | StripeCardRepositoryImpl |
| POST | `ephemeral_keys` | Ephemeral key (customer) | StripeCardRepositoryImpl |
| POST | `payment_intents` | Create payment intent (metadata, amount, etc.) | StripeCardRepositoryImpl |
| POST | `refunds` | Stripe refund | StripeCardRepositoryImpl |

---

### 2.6 Employee / Time Log

| Method | Path | Resource | Used by |
|--------|------|----------|---------|
| POST | `/api/employee/timeLog/clock-in` | Clock in (ClockInRequest) | **ClockInActivity**, ClockInViewModel |
| POST | `/api/employee/timeLog/clock-out` | Clock out (invalidateToken, ClockInRequest) | ClockInRepository, DashBoardRepository |
| POST | `/api/employee/timeLog/break-in-v2` | Break in | BreaksViewModel |
| POST | `/api/employee/timeLog/break-out-v2` | Break out | BreaksViewModel |
| POST | `api/employee/timeLog/meal-penalty` | Meal penalty | BreakRepositoryImpl |
| POST | `api/employee/timeLog/break-log` | Log break | BreakRepositoryImpl |
| GET | `api/employee/timeLog/remaining-breaks` | Remaining breaks | **ClockOutPendingBreaksFragment**, BreaksViewModel |
| GET | `api/employee/timeLog/meal-break-status` | Meal break status (breakLawId) | BreakRepositoryImpl |
| GET | `/api/employee/timeLog/employee-time-data` | Employee time data (time, employeeId, roleId) | DashBoardRepository |
| GET | `/api/employee/timeLog/all-employees-time-data-v3` | All employees time (onlyClockedIn) | DashBoardRepository, TicketsRepository |
| GET | `api/employee/timeLog/all-employees-time-data-v2` | All employees time v2 | CashDrawerRepository |
| POST | `/api/employee/timeLog/clock-out-employee` | Clock out employee from dashboard | **DashBoardActivity**, DashBoardViewModel |
| POST | `/api/employee/documents/upload-image` | Upload document (Multipart) | **ClockInActivity**, ClockInViewModel |
| PATCH | `/api/employee/change-server` | Change server (ChangeServerRequestModel) | TicketsRepository |
| PATCH | `/api/employee/assign-server` | Assign server (AssignServerModel) | TicketsRepository |
| GET | `api/employee` | Employees by role (role, noMaxLimit, unlinkedToCashDrawer, status) | CashDrawerRepository |
| GET | `api/employee/pos/all-employees/` | FOH / clocked-in servers (roleTag, excludeAioBuddy or onlyClockedIn) | TicketsRepository, CashDrawerRepository |
| GET | `api/employee` | Restaurant employees (noMaxLimit, isCashDrawerPermitted, status) | CashDrawerRepository |
| PATCH | `/api/employee/add-review` | Rate employee (RatingBody) | TicketsRepository |

---

### 2.7 Cash Drawer

| Method | Path | Resource | Used by |
|--------|------|----------|---------|
| GET | `api/cash-drawer/{cashDrawerId}` | Cash drawer logs (filter, rangeType, startDate, endDate, isHouseDrawer) | **CashDrawerStationDetailFragment**, CashDrawerViewModel |
| GET | `api/cash-drawer/cash_payment_logs` | Cash payment logs (servers, devices) | CashDrawerViewModel |
| GET | `api/cash-drawer/users/{cashDrawerId}` | Users linked to drawer | **DrawerUsersFragment**, CashDrawerViewModel |
| POST | `api/cash-drawer/drawer-logs/{drawerId}` | Drawer logs by drawer (CashDrawerLogsRequest) | CashDrawerViewModel |
| GET | `api/cash-drawer/all` | All cash drawer stations | CashDrawerViewModel |
| POST | `api/cash-drawer/close/{drawerId}` | Close drawer (CashDrawerCloseRequest) | **CloseCashDrawerFragment**, CashDrawerViewModel |
| PATCH | `api/cash-drawer/adjust_balance/{drawerId}` | Starting balance (StartingBalanceRequest) | CashDrawerViewModel |
| POST | `/api/cash-drawer/drawer-logs/{drawerId}` | Submit cash tips (DeclareRequest) | DashBoardRepository |
| POST | `/api/cash-drawer/assign_user` | Assign user to drawer (AddDrawerUserRequest) | **AddDrawerUsersFragment**, CashDrawerViewModel |
| HTTP DELETE | `api/cash-drawer/users` | Unlink user from drawer (DrawerUserRequest) | CashDrawerViewModel |
| GET | `api/cash-drawer/checkout-settlement-employee-list` | Checkout settlement employee list | CashDrawerViewModel |
| POST | `api/cash-drawer/checkout-settlement/{cashDrawerId}` | Settlement (SettlementRequest) | CashDrawerViewModel |

---

### 2.8 Restaurant / POS / Settings

| Method | Path | Resource | Used by |
|--------|------|----------|---------|
| GET | `api/restaurant/restaurant-by-deviceId/{deviceId}` | Restaurant by device ID | MenuRepositoryImpl, OnRestaurantByDeviceApi, **SplashActivity** |
| GET | `api/restaurant/get/stations` | Stations | SettingsRepository, **PrintersFragment** |
| GET | `api/restaurant/pos/getall` | All POS devices (restaurantId) | RestaurantInfoImplRepository |
| GET | `/api/restaurant/pos/{posId}` | POS device by ID (restaurantId) | **DeviceNameFragment**, RestaurantInfoViewModel, SettingsRepository |
| PATCH | `/api/restaurant/pos/edit/{posId}` | Update device (POSData) | **DevicesSettingsFragment**, SettingsViewModel |
| PATCH | `/api/restaurant/printer/stations/mapping/{printerId}` | Update printer mapping | **PrinterSettingsDetailFragment**, SettingsViewModel |
| PATCH | `/api/restaurant/kds/edit/{kdsId}` | Update KDS (ConnectedKd) | **KdsSettingsDetailFragment**, SettingsViewModel |
| GET | `/api/restaurant/printer/getall` | All printers | **PrintersFragment**, SettingsViewModel |
| GET | `/api/restaurant/kds/get/all` | All KDS | SettingsRepository |
| POST | `/api/restaurant/close-out` | Close out (CloseOutRequest) | DashBoardRepository, **DashBoardActivity** |
| POST | `api/restaurant/open-up/` | Open up (closeOutId) | **ReOpeningShiftFragment**, ClockInViewModel |

---

### 2.9 Menu / POS Load

| Method | Path | Resource | Used by |
|--------|------|----------|---------|
| GET | `api/menu/pos/{restaurantId}` | Menu for POS (deviceSerialNumber) | MenuRepositoryImpl, **Register flow** (menu load / splash) |

---

### 2.10 Dashboard / Reports

| Method | Path | Resource | Used by |
|--------|------|----------|---------|
| POST | `/api/pos-dashboard/overview` | Dashboard overview | **OverViewFragment**, DashBoardViewModel |
| POST | `/api/pos-dashboard/my-day` | My day / staff dashboard (StaffRequest) | DashBoardViewModel |
| GET | `/api/pos-dashboard/z-report/complete-status` | Z-report complete status | DashBoardRepository |
| GET | `/api/pos-dashboard/server-checkout` | Server checkout stats | **ServerCheckoutReportFragment**, **ServerReportInnerFragment**, DashBoardViewModel |
| GET | `/api/pos-dashboard/z-report` | Z-report stats (isCloseOut) | DashBoardViewModel (RetrofitApiServiceB) |

---

### 2.11 Online Ordering

| Method | Path | Resource | Used by |
|--------|------|----------|---------|
| GET | `/api/online-ordering-settings/{restaurantId}` | Online ordering status | OrderHubRemoteDataSource |
| PATCH | `/api/online-ordering-settings/{id}` | Set online ordering status | OrderHubRemoteDataSource |

---

### 2.12 Customer / Loyalty / Smart Panel

| Method | Path | Resource | Used by |
|--------|------|----------|---------|
| GET | `/api/customer` | Search customers (search, page, limit) | **SearchCustomerFragment**, TicketViewModel |
| GET | `api/loyalty/pos-menu` | Loyalty menu | SmartPanel / loyalty flow |
| POST | `api/ticket/apply-loyalty` | Apply loyalty (ApplyLoyaltyRequestModel) | SmartPanel |
| GET | `api/customer/smartPanel` | Smart panel customer info (customerId) | SmartPanel |

---

### 2.13 Taxes / Modifier / Item Stock

| Method | Path | Resource | Used by |
|--------|------|----------|---------|
| GET | `/api/taxes/restaurant/{id}` | Taxes for restaurant | TicketsRepository |
| POST | `api/item/stock-in-stock-out` | Item stock in/out (StockStatusRequest) | ApiQueueingRemoteSource |
| POST | `api/modifier-option/update/stock-status` | Modifier option stock (ModifierOptionStatusRequest) | ApiQueueingRemoteSource |

---

### 2.14 Kitchen Hub / Order Out

| Method | Path | Resource | Used by |
|--------|------|----------|---------|
| PUT | `api/kitchen-hub/respond-to-order` | Accept/reject order out (OrderOutRequestModel) | SmartPanel / order-out flow |

---

### 2.15 MQTT / Reconcile

| Method | Path | Resource | Used by |
|--------|------|----------|---------|
| GET | `api/triangulumnova/reconcile-mqtt-events` | Reconcile MQTT (restaurantId, ticketId, seqNoStart) | TicketSequenceRemoteSourceImpl |

---

## 3. Screen → ViewModels → Endpoints (Summary)

| Screen (Fragment / Activity) | ViewModels / Repos | Main endpoints used |
|-------------------------------|--------------------|----------------------|
| **ClockInActivity** / ClockInFragment | ClockInViewModel, ClockInRepository | login-with-pos, logout, approve-with-pin, clock-in, upload-image, open-up, clock-out |
| **SplashActivity** | MenuRepository (getRestaurantForPos), apiServiceB.allTickets | restaurant-by-deviceId, api/menu/pos/{id}, api/ticket/splash/alltickets/v2 |
| **RegisterActivity** / RegisterFragment, **TicketFragment**, **TsrMenuFragment**, **QsrMenuFragment** | MainViewModel, TicketViewModel, TsrTicketViewModel, QsrTicketViewModel, MenuViewModelForRoom, PaymentViewModel, ApiQueueManager | createId, update ticket, get ticket, table/ticket, payment calculation, post payment, tsr-create, qsr-create, split, void, service-charges, discounts, deals, taxes, getTicketResponse, getTicketsByTable |
| **PaymentParentFragment**, **PayByCashFragment**, **PayByTenderFragment**, **PayByGiftCardFragment**, **PayByCardFragment** | PaymentViewModel, TicketViewModel, TsrTicketViewModel | payment v2/calculation, post payment, createIntentInfo, processIntent, capture, webhook, gift-card, getPaymentId, getTenderList |
| **AddTipUnclosedFragment** | PaymentViewModel, OrdersFiltersViewModel, StripeCardViewModel | getFilteredData (filtered tickets), adjustPayment, getPaymentWebhookStatus, capturePayment |
| **RefundMainFragment**, **PaymentRefundFragment**, **ItemsRefundFragment** | PaymentViewModel, TicketViewModel | getPaymentRefundId, postCashRefund, refundPaymentNew, cancelPayment |
| **OrdersHubParentFragment**, **OrdersHubTicketsFragment** | OrdersHubViewModel, OrdersFiltersViewModel | getFilteredTickets, getAllOrders, getOnlineOrderingStatus, setOnlineOrderingStatus |
| **DashBoardActivity**, **OverViewFragment**, **ServerCheckoutReportFragment**, **ServerReportInnerFragment**, **EmployeeScheduleFragment** | DashBoardViewModel | overViewDetail, getStaffDashBoardData, getZReportCompleteStatus, getZReportStats, getServerCheckoutStats, getNewServerCheckoutData, getEmployeeTimeData, getAllEmployeeTimeData, clockOutEmployeeFromDashboard, submitCashTips, closeOut, getUnclosed |
| **TableLayoutBuilderFragment**, **TableLayoutViewFragment**, **TableBuilderEditorFragment** | TableBuilderViewModel, FloorTableViewModel | getAllFloors, getFloorTable, setFloorName, saveFloorTables, deleteFloor, deleteTable, getTableNo, getTablesStatus |
| **CashDrawerActivity**, **CashDrawerStationDetailFragment**, **CloseCashDrawerFragment**, **DrawerUsersFragment**, **AddDrawerUsersFragment**, **CashLogsParentFragment** | CashDrawerViewModel | getCashDrawerLogsList, getCashDrawerLogsByDrawerId, getCashDrawerStations, getAddedUsers, submitCashDrawerClose, setStartingBalance, addEmployeeToDrawer, unLinkEmployeeFromDrawer, getRestaurantEmployees, getCheckoutSettlementEmployeeList, settlementEmployee |
| **SettingsFragment**, **DevicesSettingsFragment**, **DeviceNameFragment**, **PrintersFragment**, **PrinterSettingsDetailFragment**, **KDSFragment**, **KdsSettingsDetailFragment** | SettingsViewModel, RestaurantInfoViewModel | getRestaurantDevices, getPOSDevices, updateDeviceData, updatePrinterSettings, updateKDSSettings, getRestaurantPrinters, getRestaurantKDS, getStations |
| **TemporaryGuestFragment**, **SearchCustomerFragment** | TicketViewModel, QsrTicketViewModel | addCustomer, getCustomers, addExistingCustomerToTicket |
| **ReOpeningShiftFragment** | ClockInViewModel | openUp |
| **ClockOutPendingBreaksFragment** | BreaksViewModel | getPendingBreaks, breakIn, breakOut, mealPenalty, logBreak, checkBreakStatus |
| **CallServerFragment**, **ViewTableLayout** | TicketViewModel, TableViewModel | getTicketsByTable, modifyServer, makeTableEmpty, getServerListClockedIn |
| **MyTablesFragment** (Smart Panel) | MyTablesViewModel | getMyTables |
| **Move check / Move items** | MoveCheckViewModel, MoveItemsViewModel | moveCheck, moveCheckOffline, moveItemsOffline |
| **Undo split** | UndoSplitTicketViewModel | undoSplitTicket, undoSplitTicketItem, undoSplitTicketsOffline |

---

## 4. Quick Reference – All Paths (RetrofitAPIService + ApiServiceB)

- `GET  /api/table/my-tables`
- `PATCH api/table/change-table-pos`
- `PATCH api/ticket/update/{ticketId}` (update + apply discount)
- `PATCH api/ticket/remove/ticket-item/{ticketId}`
- `POST api/payment/v2/calculation/{id}`
- `POST api/payment/{ticketId}`
- `POST api/payment/calculation/{id}`
- `POST api/payment/intent`
- `POST /api/payment/new/intent/v2`
- `POST api/payment/process-payment`
- `POST api/payment/cardpaymenttrigger/{ticketId}`
- `POST api/payment/refund/{ticketId}`
- `POST /api/ticket/createId`
- `GET  /api/ticket/get/allorders`
- `GET  /api/online-ordering-settings/{restaurantId}`
- `PATCH /api/online-ordering-settings/{id}`
- `GET  /api/ticket/{ticketId}`
- `DELETE /api/ticket/items/{ticketId}`
- `PATCH /api/restaurant/pos/edit/{posId}`
- `PATCH /api/restaurant/printer/stations/mapping/{printerId}`
- `PATCH /api/restaurant/kds/edit/{kdsId}`
- `PATCH /api/ticket/update-status/{ticketId}`
- `PATCH /api/ticket/voidorder`
- `POST /api/ticket/cancel-scheduler-order/{ticketId}`
- `POST /api/ticket/manual-fire-kds-order/{ticketId}`
- `PATCH /api/employee/change-server`
- `DELETE /api/ticket/items/{ticketId}` (custom item)
- `PATCH /api/ticket/split-by-order/{orderId}`
- `GET  /api/table/ticket/{tableId}`
- `PATCH /api/table/free/table`
- `PATCH /api/employee/assign-server`
- `POST /api/authentication/login-with-pos`
- `POST /api/authentication/logout`
- `POST /api/employee/timeLog/clock-in`
- `POST /api/authentication/approve-with-pin`
- `POST /api/employee/timeLog/clock-out`
- `POST /api/employee/timeLog/break-in-v2`
- `POST /api/employee/documents/upload-image`
- `GET  /api/ticket/all/filter`
- `GET  api/restaurant/get/stations`
- `GET  api/restaurant/pos/getall`
- `GET  api/cash-drawer/{cashDrawerId}`
- `GET  api/cash-drawer/cash_payment_logs`
- `GET  api/cash-drawer/users/{cashDrawerId}`
- `POST api/cash-drawer/drawer-logs/{drawerId}` (logs + tips)
- `GET  api/cash-drawer/all`
- `POST api/cash-drawer/close/{drawerId}`
- `PATCH api/cash-drawer/adjust_balance/{drawerId}`
- `POST /api/pos-dashboard/overview`
- `POST /api/pos-dashboard/my-day`
- `GET  /api/pos-dashboard/z-report/complete-status`
- `GET  /api/employee/timeLog/employee-time-data`
- `GET  /api/employee/timeLog/all-employees-time-data-v3`
- `POST /api/webhook/{intentId}`
- `POST /api/payment/intent/cancel/v2`
- `PATCH api/ticket/service-charges/{ticketId}`
- `GET  api/employee` (by role + restaurant employees)
- `GET  api/employee/timeLog/all-employees-time-data-v2`
- `GET  api/employee/pos/all-employees/` (FOH + clocked-in)
- `POST /api/cash-drawer/assign_user`
- `DELETE api/cash-drawer/users`
- `POST /api/restaurant/close-out`
- `GET  /api/restaurant/printer/getall`
- `GET  /api/restaurant/kds/get/all`
- `GET  /api/restaurant/pos/{posId}`
- `GET  /api/taxes/restaurant/{id}`
- `PATCH /api/employee/add-review`
- `POST /api/payment/receipt/{paymentId}`
- `POST /api/employee/timeLog/break-out-v2`
- `DELETE /api/ticket/order/empty-tickets`
- `PATCH /api/ticket/delete-unsent-items/{orderId}`
- `POST /api/table/floor`
- `PATCH /api/table/bulk-table/{floorId}`
- `GET  /api/table/floor/restaurant`
- `DELETE /api/table/floor/{floorId}`
- `DELETE /api/table/{tableId}`
- `GET  /api/table/floor/{floorId}`
- `GET  /api/table/tableNo`
- `POST api/restaurant/open-up/`
- `GET  api/ticket/count/un-closed`
- `POST api/payment/connectiontoken`
- `GET  api/payment/{ticketId}`
- `GET  api/ticket/kds-pos-print/{ticketId}`
- `GET  api/employee/timeLog/remaining-breaks`
- `GET  api/ticket/get-pending-tickets-status`
- `POST api/employee/timeLog/meal-penalty`
- `GET  api/employee/timeLog/meal-break-status`
- `POST api/employee/timeLog/break-log`
- `GET  api/payment/refund/{ticketId}`
- `GET  api/tender/get-active-tenders`
- `POST api/authentication/refresh-token`
- `POST /api/ticket/add-new-customer/{ticketId}`
- `GET  /api/customer`
- `POST api/ticket/ticket-item-discount/{ticketId}`
- `GET  api/table/get-tables-status/{restaurantId}`
- `POST api/payment/intent/capture/v2`
- `GET  api/loyalty/pos-menu`
- `POST api/ticket/apply-loyalty`
- `GET  api/customer/smartPanel`
- `GET  api/restaurant/restaurant-by-deviceId/{deviceId}`
- `POST /api/payment/apply-deal`
- `DELETE /api/payment/deals/{ticketId}/{dealId}`
- `POST api/ticket/qsr-create/{ticketId}`
- `POST api/ticket/tsr-create/{orderId}`
- `POST api/item/stock-in-stock-out`
- `PUT  api/kitchen-hub/respond-to-order`
- Plus remaining ticket/payment/cash-drawer/settlement/move-check/undo-split endpoints from Section 2.

---

*Generated from the android-pos codebase. For request/response types, see the corresponding Retrofit interface and repository implementations.*

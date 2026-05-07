# Code Review — AdsSdk

**Reviewed by:** Claude Code  
**Date:** 2026-05-07  
**Branch:** master

---

## Overview

A multi-module Android ads SDK using Clean Architecture (Domain/Data/Presentation), Hilt DI, and AdMob. Covers interstitial, banner, collapsible banner, native ads, app-open ads, GDPR consent (UMP), and in-app billing.

---

## Summary Table

| Severity | Count | Top Item |
|---|---|---|
| Critical | 5 | `list[0]` crash in billing, `!!` NPEs, duplicate `withNativeAdOptions` |
| High | 5 | BillingClient leak, `isShowingAd` copy-not-reference, swallowed callback |
| Medium | 5 | 1000ms race condition, no purchase callbacks, inconsistent ad guards |
| Low | 5 | `data/data` naming, no tests, mutable data class |

---

## Critical Bugs

### 1. `InAppConsumableBilling.purchaseItem()` — crash on empty list
**File:** `module-ads/.../billing/InAppConsumableBilling.kt:75`

```kotlin
launchPurchaseFlow(list[0])  // IndexOutOfBoundsException if product ID not found
```

If the product ID does not exist in the Play Store, `list` is empty and this crashes. Fix: check `list.isNotEmpty()` and handle the empty case before accessing index 0.

---

### 2. `InAppConsumableBilling.launchPurchaseFlow()` — force unwrap on nullable param
**File:** `module-ads/.../billing/InAppConsumableBilling.kt:82`

```kotlin
setProductDetails(productDetails!!)  // NPE — the param is already nullable
```

The function signature accepts `ProductDetails?` but immediately force-unwraps it. Use a null check and return early instead.

---

### 3. `InAppConsumableBilling.handlePurchase()` — force unwrap on BillingClient
**File:** `module-ads/.../billing/InAppConsumableBilling.kt:97`

```kotlin
billingClient!!.acknowledgePurchase(...)  // NPE if billingClient is null
```

`billingClient` is declared as `var BillingClient?` and can be null. Use the safe-call operator `?.` instead.

---

### 4. `NativeAdRepositoryImpl` — `withNativeAdOptions` called twice
**File:** `module-ads/.../data/data/NativeAdRepositoryImpl.kt:78-80` and `113-117`

`withNativeAdOptions()` is called twice on the same `AdLoader.Builder`. The second call overrides the first, silently dropping the `VideoOptions` (start-muted) configuration. Merge both into a single `withNativeAdOptions()` call.

---

### 5. `NativeAdRepositoryImpl` — advertiser view always hidden
**File:** `module-ads/.../data/data/NativeAdRepositoryImpl.kt:226`

```kotlin
(adverView as TextView).text = mNativeAd?.advertiser
adverView.visibility = View.GONE   // Bug: should be View.VISIBLE
```

The advertiser text is populated but the view is set to `GONE` in both the null and non-null branches. The advertiser is never visible.

---

## High Severity

### 6. `InAppConsumableBilling.restorePurchases()` — BillingClient connection leak
**File:** `module-ads/.../billing/InAppConsumableBilling.kt:132`

`restorePurchases()` creates and assigns a brand-new `BillingClient` to `billingClient` without calling `endConnection()` on the existing one first. The old connection is leaked. Always call `billingClient?.endConnection()` before creating a new instance.

---

### 7. `OpenAdHelper.isShowingAd` — value copy, not a live reference
**File:** `module-ads/.../open_ad/OpenAdHelper.kt:34`

```kotlin
var isShowingAd = appOpenAdManager.isShowingAd  // copies the Boolean value (false)
```

This captures the initial `false` value at construction time. `OpenAdHelper.isShowingAd` will always remain `false` regardless of what happens inside `AppOpenAdManager`. Any external consumer of this property gets stale data.

---

### 8. `OpenAdHelper` — `init` block resets shared static state
**File:** `module-ads/.../open_ad/OpenAdHelper.kt:31`

```kotlin
init {
    appOpenAdManager = AppOpenAdManager()
    enableResumeAd()   // resets static isAppOpen = true
}
```

Every new `OpenAdHelper` instance resets the shared static flag `isAppOpen` to `true`. Calling `disableResumeAd()` and then creating a new `OpenAdHelper` anywhere silently re-enables ads globally.

---

### 9. `OpenAdHelper.showAdIfAvailable` — early return swallows completion callback
**File:** `module-ads/.../open_ad/OpenAdHelper.kt:219-222`

```kotlin
if (!isAppOpen) {
    debug("open ad is disabled")
    return   // onShowAdComplete() is never called — caller is blocked forever
}
```

When the open ad is disabled, the function returns without invoking `onShowAdCompleteListener.onShowAdComplete()`. Any navigation or logic gated on this callback will never execute, silently breaking app flow.

---

### 10. `NativeAdRepositoryImpl.onAdImpression` — nulls ad reference while ad is still displayed
**File:** `module-ads/.../data/data/NativeAdRepositoryImpl.kt:88`

```kotlin
override fun onAdImpression() {
    nativeAdLoadCallback.onNativeAdImpression()
    mNativeAd = null   // ad is still on screen; destroyNativeAd() is now a no-op
}
```

Setting `mNativeAd = null` on impression means `destroyNativeAd()` can no longer clean up the ad object, causing a memory leak.

---

## Medium Severity

### 11. `InterstitialAdRepositoryImpl` — callback set after delayed show is posted
**File:** `module-ads/.../data/data/InterstitialAdRepositoryImpl.kt:153-190`

```kotlin
fullScreenDialog = FullScreenDialog(activity).apply { show() }
Handler(Looper.getMainLooper()).postDelayed({
    interstitialAd.show(activity)           // posted first
}, 1000)
interstitialAd.fullScreenContentCallback = ...   // set second
```

The 1000ms delay is a workaround for showing a loading dialog, but the `fullScreenContentCallback` is attached after the show is already posted. The callback should be set *before* the delayed post to guarantee it is in place when the ad shows.

---

### 12. `InterstitialAdRepositoryImpl` — dialog shown without lifecycle guard
**File:** `module-ads/.../data/data/InterstitialAdRepositoryImpl.kt:153`

`FullScreenDialog` is shown immediately (line 153) but the interstitial ad only shows after a 1000ms delay. If the activity is destroyed or rotated in that window, the dialog is attached to a dead context and will never be dismissed, causing a window leak.

---

### 13. Inconsistent ad guards across ad types

Interstitial loading checks `isPurchased` and `isRemoteConfig` before loading. Banner and native ad loading have no equivalent checks. Callers must implement this logic themselves with no enforcement at the SDK level, making it easy to accidentally serve ads to purchased users.

---

### 14. `InAppConsumableBilling` — no result callbacks for purchase or restore
**File:** `module-ads/.../billing/InAppConsumableBilling.kt`

`purchaseItem()`, `handlePurchase()`, and `consumePurchase()` only report results via `debug()` logs. There is no callback interface to surface purchase success, failure, or restore results to calling code. Purchase outcomes are unobservable by the consumer.

---

### 15. `AdMobViewModel` extends `AndroidViewModel` unnecessarily
**File:** `module-ads/.../presentation/AdMobViewModel.kt:35`

The `application: Application` parameter is injected but never used — context is passed via individual method parameters instead. Should extend `ViewModel` rather than `AndroidViewModel`.

---

## Low Severity / Code Quality

### 16. Doubled directory: `data/data/`

All repository implementations sit at `module_ads/data/data/`. The redundant nesting is confusing. Consider renaming to `data/repositories/` to match standard Clean Architecture convention.

---

### 17. `AppModule.provideBreakInfoArrayList()` — unused leftover
**File:** `module-ads/.../di/AppModule.kt:65-68`

An `ArrayList<Any>` singleton is provided with no documentation and no apparent usage anywhere in the module. Likely a leftover that should be removed.

---

### 18. `GoogleMobileAdsConsentManager` — placeholder test device ID always active
**File:** `module-ads/.../utils/GoogleMobileAdsConsentManager.kt:51`

```kotlin
.addTestDeviceHashedId("TEST-DEVICE-HASHED-ID")
```

The placeholder string is never replaced with a real device ID, so debug consent settings are always applied (even in production builds). This should be conditional on `BuildConfig.DEBUG` or removed.

---

### 19. Zero actual tests

All four test files (`ExampleUnitTest`, `ExampleInstrumentedTest` in both modules) are empty templates. Given the ad-gating logic (purchase + remote config checks) and the billing state machine, unit tests for `InterstitialAdRepositoryImpl`'s caching and `InAppConsumableBilling`'s purchase flow would catch several of the bugs listed above.

---

### 20. `InterstitialAdInfo` — mutable `data class` used as shared state
**File:** `module-ads/.../data/model/InterstitialAdInfo.kt`

`InterstitialAdInfo` is a `data class` with mutable `var` fields (`isAdsLoading`, `interstitialAd`) that are mutated externally via `adInfo.apply { ... }`. Passing a mutable data model between caller and repository leaks internal loading state to consumers. The mutable state should be encapsulated entirely within the repository.

---

## Strengths

- Clean Architecture layers (Domain / Data / Presentation) are properly separated.
- Hilt DI is correctly wired end-to-end.
- GDPR compliance via UMP is integrated at the right level.
- App-open ad lifecycle is properly tied to `ProcessLifecycleOwner`.
- Network availability is checked before every ad load.
- Debug logging and toasts are gated behind `BuildConfig.DEBUG`.
- Adaptive banner sizing handles both pre- and post-API 30 correctly.
- `InterstitialAd` caching with HashMap supports multiple concurrent ad slots.

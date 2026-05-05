# Ad Format Implementation Patterns for KMP

## Banner Ads

Persistent view, simplest format. Fixed sizes: 320x50, 320x100, adaptive.

**Android:**
```kotlin
@Composable
actual fun BannerAdView(adUnitId: String, modifier: Modifier) {
    AndroidView(
        factory = { ctx ->
            AdView(ctx).apply {
                setAdSize(AdSize.BANNER)
                this.adUnitId = adUnitId
                loadAd(AdRequest.Builder().build())
            }
        },
        modifier = modifier
    )
}
```

**iOS (via bridge):**
```kotlin
@Composable
actual fun BannerAdView(adUnitId: String, modifier: Modifier) {
    val bannerView = remember { IosAdBridge.provider?.createBannerView(adUnitId) }
    bannerView?.let { view ->
        UIKitView(
            factory = { view },
            modifier = modifier.fillMaxWidth().height(50.dp),
            properties = UIKitInteropProperties(isInteractive = true)
        )
    }
}
```

## Interstitial Ads

Full-screen, load in advance, show at transition points. No view needed.

**Common interface:**
```kotlin
interface InterstitialAdNode {
    val config: InterstitialAdConfig
    var isEnabled: Boolean
    fun load(context: PlatformContext)
    fun show(context: PlatformContext, onAdShowComplete: () -> Unit)
    fun handleAdOpportunity(context: PlatformContext, onComplete: () -> Unit)
}
```

**Throttling pattern:** Track `lastShowTime`, `showedCounter`, `actionCounter` to limit frequency.

## App Open Ads

Show on cold start or foregrounding. Load async, show before main content.

**Key pattern:** `loadAndShowAsync` with timeout — don't block app launch forever.

```kotlin
interface AdAppOpen {
    fun load(context: PlatformContext)
    suspend fun loadAndShowAsync(context: PlatformContext, timeoutMillis: Long)
    fun show(context: PlatformContext, onAdShowComplete: () -> Unit)
}
```

**iOS implementation uses `suspendCancellableCoroutine` with `withTimeoutOrNull`.**

## Native Ads (Most Complex)

Custom-rendered ads using app's own UI. Requires platform views for click tracking.

**Common data model:**
```kotlin
data class NativeAdData(
    val headline: String?,
    val body: String?,
    val callToAction: String?,
    val advertiser: String?,
    val store: String?,
    val price: String?,
    val starRating: Double?,
    val icon: Any?,
)
```

### Android Native
- `NativeAdLoader` loads `NativeAd`
- Wrap in `com.google.android.gms.ads.nativead.NativeAdView` via `AndroidView`
- Register each asset view: `headlineView`, `bodyView`, `callToActionView`, `mediaView`, `iconView`
- `ComposeView` inside `AndroidView` lets you use Compose for the template

### iOS Native (Per-Node Handler Pattern)
- **Problem:** Shared bridge = callback collision when multiple native ads exist
- **Solution:** `NativeAdHandler` interface with factory method on bridge
- Each `IosNativeAdNode` owns its own handler instance
- Swift handler builds `GoogleMobileAds.NativeAdView` with Auto Layout
- Embed via `UIKitView` with `isInteractive = true`

## Rewarded Ads

Video ads that grant in-app rewards. Similar to interstitial but with reward callback.

**Common interface:**
```kotlin
interface RewardedAdNode {
    fun load(context: PlatformContext)
    fun show(context: PlatformContext, onRewarded: (type: String, amount: Int) -> Unit, onComplete: () -> Unit)
    fun isLoaded(): Boolean
}
```

**Android:**
```kotlin
rewardedAd?.show(activity) { reward ->
    onRewarded(reward.type, reward.amount)
}
```

**iOS (Swift):**
```swift
func showRewardedAd(completion: @escaping () -> Void, onRewarded: @escaping (String, Int32) -> Void) {
    rewardedAd?.present(fromRootViewController: vc) { [weak self] in
        let reward = self?.rewardedAd?.adReward
        onRewarded(reward?.type ?? "", Int32(reward?.amount.intValue ?? 0))
    }
    // completion called in adDidDismissFullScreenContent delegate
}
```

## Ad Manager Singleton Pattern

Centralize ad lifecycle in a singleton:

```kotlin
abstract class BaseAdManager {
    abstract val adOpen: AdAppOpen
    abstract val interstitial: InterstitialAdNode
    abstract val nativeMain: NativeAdNode
    // etc.

    open fun init(context: PlatformContext) {
        applyTestDeviceIds(testDeviceIds)
        adOpen.load(context)
        interstitial.load(context)
        nativeMain.load(context)
    }
}
```

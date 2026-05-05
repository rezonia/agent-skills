# Official Google Mobile Ads Test Ad Unit IDs

Use these IDs during development. Using production IDs in test builds violates AdMob policy and risks account suspension.

## Android Test IDs

| Format | Ad Unit ID |
|--------|-----------|
| Banner | `ca-app-pub-3940256099942544/6300978111` |
| Interstitial | `ca-app-pub-3940256099942544/1033173712` |
| Interstitial (Video) | `ca-app-pub-3940256099942544/8691691433` |
| Native | `ca-app-pub-3940256099942544/2247696110` |
| Native (Video) | `ca-app-pub-3940256099942544/1044960115` |
| App Open | `ca-app-pub-3940256099942544/9257395915` |
| Rewarded | `ca-app-pub-3940256099942544/5224354917` |
| Rewarded Interstitial | `ca-app-pub-3940256099942544/5354046379` |

## iOS Test IDs

| Format | Ad Unit ID |
|--------|-----------|
| Banner | `ca-app-pub-3940256099942544/2934735716` |
| Interstitial | `ca-app-pub-3940256099942544/4411468910` |
| Native | `ca-app-pub-3940256099942544/3986624511` |
| App Open | `ca-app-pub-3940256099942544/5662855259` |
| Rewarded | `ca-app-pub-3940256099942544/1712485313` |
| Rewarded Interstitial | `ca-app-pub-3940256099942544/6978759866` |

## KMP Usage Pattern

```kotlin
// Platform.kt (commonMain)
expect val adUnitIds: AdUnitIds

data class AdUnitIds(
    val banner: String,
    val interstitial: String,
    val native: String,
    val appOpen: String,
    val rewarded: String,
)

// Platform.android.kt
actual val adUnitIds = if (BuildConfig.DEBUG) {
    AdUnitIds(
        banner = "ca-app-pub-3940256099942544/6300978111",
        interstitial = "ca-app-pub-3940256099942544/1033173712",
        native = "ca-app-pub-3940256099942544/2247696110",
        appOpen = "ca-app-pub-3940256099942544/9257395915",
        rewarded = "ca-app-pub-3940256099942544/5224354917",
    )
} else {
    AdUnitIds(/* production IDs from secure config */)
}
```

## Test Device Registration

```kotlin
// commonMain
expect fun applyTestDeviceIds(deviceIds: List<String>)

// androidMain
actual fun applyTestDeviceIds(deviceIds: List<String>) {
    val config = RequestConfiguration.Builder()
        .setTestDeviceIds(deviceIds)
        .build()
    MobileAds.setRequestConfiguration(config)
}

// iosMain — via bridge
actual fun applyTestDeviceIds(deviceIds: List<String>) {
    IosAdBridge.provider?.setTestDeviceIds(deviceIds)
}
```

Find your test device ID in logcat/console: `"Use RequestConfiguration.Builder.setTestDeviceIds(..."`.

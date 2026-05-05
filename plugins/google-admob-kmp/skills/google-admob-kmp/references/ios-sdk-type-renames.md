# iOS Google Mobile Ads SDK v12+ Type Renames

Starting with SDK v12.0.0, the `GAD` prefix was removed for Swift. ObjC retains `GAD` prefixed names.

## Complete Mapping

| ObjC / Old Swift | New Swift (v12+) | Category |
|------------------|------------------|----------|
| `GADMobileAds` | `MobileAds` | SDK entry |
| `GADMobileAds.sharedInstance()` | `MobileAds.shared` | Singleton |
| `GADRequest` | `Request` | Request |
| `GADResponseInfo` | `ResponseInfo` | Response |
| `GADBannerView` | `BannerView` | Banner |
| `GADBannerViewDelegate` | `BannerViewDelegate` | Banner |
| `GADAdSize` | `AdSize` | Banner |
| `GADInterstitialAd` | `InterstitialAd` | Interstitial |
| `GADRewardedAd` | `RewardedAd` | Rewarded |
| `GADRewardedInterstitialAd` | `RewardedInterstitialAd` | Rewarded |
| `GADAdReward` | `AdReward` | Rewarded |
| `GADAppOpenAd` | `AppOpenAd` | App Open |
| `GADNativeAd` | `NativeAd` | Native |
| `GADNativeAdView` | `NativeAdView` | Native |
| `GADNativeAdImage` | `NativeAdImage` | Native |
| `GADMediaView` | `MediaView` | Native |
| `GADMediaContent` | `MediaContent` | Native |
| `GADAdLoader` | `AdLoader` | Native |
| `GADAdLoaderDelegate` | `AdLoaderDelegate` | Native |
| `GADNativeAdLoaderDelegate` | `NativeAdLoaderDelegate` | Native |
| `GADNativeAdDelegate` | `NativeAdDelegate` | Native |
| `GADFullScreenContentDelegate` | `FullScreenContentDelegate` | Full-screen |
| `GADFullScreenPresentingAd` | `FullScreenPresentingAd` | Full-screen |
| `GADVideoOptions` | `VideoOptions` | Options |
| `GADServerSideVerificationOptions` | `ServerSideVerificationOptions` | Rewarded |

## KMP-Specific Notes

- **Kotlin/Native cinterop** still generates `GAD`-prefixed Kotlin bindings from ObjC headers
- **Swift bridge pattern** uses the new prefix-less names — this is the recommended approach
- **Module qualification** needed when SDK type names clash with Kotlin exports:
  - `GoogleMobileAds.NativeAdView` instead of just `NativeAdView`
  - `GoogleMobileAds.MediaView` if project defines a `MediaView`
  - `GoogleMobileAds.Request` if project has a `Request` type

## Argument Label Renames (SDK v13+)

Beyond type renames, SDK v13 also renamed several argument labels:

| Old | New | Used in |
|-----|-----|---------|
| `withAdUnitID:` | `with:` | `AppOpenAd.load`, `InterstitialAd.load`, `RewardedAd.load`, `AdLoader(adUnitID:)` |
| `fromRootViewController:` | `from:` | `ad.present(...)` for all full-screen ads |

```swift
// Old (v12)
InterstitialAd.load(withAdUnitID: id, request: Request()) { ... }
ad.present(fromRootViewController: vc)

// New (v13+)
InterstitialAd.load(with: id, request: Request()) { ... }
ad.present(from: vc)
```

## Swift / Kotlin Interop Type Mapping

When implementing `IosAdBridgeProvider` in Swift, match Kotlin export types exactly:

| Kotlin | Swift |
|--------|-------|
| `List<String>` | `[String]` (NOT `[Any]`) |
| `(Boolean) -> Unit` | `@escaping (KotlinBoolean) -> Void` |
| `() -> Unit` | `@escaping () -> Void` |
| `Double` | `Double` |
| `String?` | `String?` |

Mismatches cause `Type does not conform to protocol` errors.

## Migration Checklist

1. Find-replace `GAD` prefix in all `.swift` files
2. Replace `GADMobileAds.sharedInstance()` with `MobileAds.shared`
3. Replace `GADRequest()` with `Request()`
4. Update protocol conformances (e.g., `GADFullScreenContentDelegate` -> `FullScreenContentDelegate`)
5. Module-qualify any types that conflict with Kotlin exports
6. Update argument labels (v13+): `withAdUnitID:` -> `with:`, `fromRootViewController:` -> `from:`
7. Verify Swift parameter types match Kotlin exports (e.g., `[String]` not `[Any]`)
8. Test compilation in Xcode

# Consent Management (UMP SDK) for KMP

## Overview

The User Messaging Platform (UMP) SDK handles GDPR (EEA/UK) and ATT (iOS) consent. Initialize UMP **before** the Mobile Ads SDK.

## Flow

```
App Launch
  -> UMP: requestConsentInfoUpdate()
  -> UMP: loadAndShowConsentFormIfRequired()
    -> (User consents or declines)
  -> Check canRequestAds
    -> If true: Initialize MobileAds SDK, load ads
    -> If false: No ads, but app works normally
```

## KMP expect/actual Pattern

```kotlin
// commonMain
expect object ConsentManager {
    val canRequestAds: Boolean
    val isPrivacyOptionsRequired: Boolean
    suspend fun requestConsentInfoUpdate(): String?
    suspend fun loadAndShowConsentFormIfRequired(): String?
    suspend fun showPrivacyOptionsForm(): String?
    fun reset()
}
```

## Android Implementation

```kotlin
// androidMain
actual object ConsentManager {
    private lateinit var consentInfo: ConsentInformation

    actual val canRequestAds: Boolean
        get() = consentInfo.canRequestAds()

    actual suspend fun requestConsentInfoUpdate(): String? =
        suspendCancellableCoroutine { cont ->
            val params = ConsentRequestParameters.Builder().build()
            consentInfo.requestConsentInfoUpdate(activity, params,
                { cont.resume(null) },
                { cont.resume(it.message) }
            )
        }

    actual suspend fun loadAndShowConsentFormIfRequired(): String? =
        suspendCancellableCoroutine { cont ->
            UserMessagingPlatform.loadAndShowConsentFormIfRequired(activity,
                { cont.resume(null) },
                { cont.resume(it.message) }
            )
        }
}
```

## iOS Implementation

iOS uses a Swift bridge (UMPHelper) since UMP SDK is Swift-only:

```kotlin
// iosMain
actual object ConsentManager {
    var delegate: ConsentManagerDelegate? = null

    actual val canRequestAds: Boolean
        get() = delegate?.canRequestAds ?: false

    actual suspend fun requestConsentInfoUpdate(): String? =
        suspendCancellableCoroutine { cont ->
            delegate?.requestConsentInfoUpdate { error ->
                cont.resume(error)
            }
        }
}
```

```swift
// Swift UMPHelper
class UMPHelper: NSObject, ConsentManagerDelegate {
    var canRequestAds: Bool {
        ConsentInformation.shared.canRequestAds
    }

    func requestConsentInfoUpdate(completion: @escaping (String?) -> Void) {
        let parameters = RequestParameters()
        ConsentInformation.shared.requestConsentInfoUpdate(with: parameters) { error in
            completion(error?.localizedDescription)
        }
    }

    func loadAndShowConsentFormIfRequired(completion: @escaping (String?) -> Void) {
        ConsentForm.loadAndPresentIfRequired(from: nil) { error in
            completion(error?.localizedDescription)
        }
    }
}
```

## Initialization Order

1. **iOSApp.init():** Register UMP delegate, configure ad bridge, init Koin
2. **AppDelegate.didFinishLaunching:** `MobileAds.shared.start()`
3. **App ViewModel:** Call `ConsentManager.requestConsentInfoUpdate()`, then `loadAndShowConsentFormIfRequired()`, then load ads if `canRequestAds`

## Debug Testing (EEA Geography)

```swift
// iOS — force EEA geography for testing
#if DEBUG
let debugSettings = DebugSettings()
debugSettings.testDeviceIdentifiers = [deviceId]
debugSettings.geography = .EEA
parameters.debugSettings = debugSettings
#endif
```

```kotlin
// Android
val debugSettings = ConsentDebugSettings.Builder(context)
    .setDebugGeography(ConsentDebugSettings.DebugGeography.DEBUG_GEOGRAPHY_EEA)
    .addTestDeviceHashedId("YOUR_TEST_DEVICE_ID")
    .build()
```

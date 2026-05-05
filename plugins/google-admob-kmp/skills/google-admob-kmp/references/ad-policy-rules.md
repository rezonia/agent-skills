# AdMob Policy Rules for Ad Implementation

Critical policy constraints that affect how ads are rendered in Compose Multiplatform / KMP apps. Violations can result in account suspension or revenue withholding.

## 1. Native Ad Presentation — NO Excessive Animation

**Rule:** Native ads must NOT be displayed with animations that draw undue attention, obstruct content, or create misleading interactions.

**Prohibited:**
- Scale, bounce, slide, or shake animations on ad entry/exit
- Pulsing, glowing, or continuous motion effects on ad containers
- Animations that mimic system notifications or alerts
- Using `AnimatedVisibility` with `scaleIn`, `expandIn`, `slideIn`, or similar attention-grabbing transitions

**Permitted:**
- Subtle fade-in (`fadeIn`) only after ad is fully loaded
- Simple visibility toggle (no animation) — safest default
- Static display with no entrance animation

**Why it matters:** AdMob monitors ad presentation for policy compliance. Excessive animation triggers can lead to account-level enforcement without prior warning.

**Reference:** [AdMob native ad policies](https://support.google.com/admob/answer/6329638)

---

## 2. Loading Placeholders — Shimmer Skeletons

**Rule:** Shimmer or skeleton placeholders while loading native ads are **permitted**, provided they do not mimic ad content or mislead users.

**Permitted:**
- Neutral gray rectangular blocks as loading skeleton
- Subtle shimmer animation on placeholder only (not on actual ad)
- Fixed-height container to prevent layout jump when ad loads

**Prohibited:**
- Fake headline, body text, CTA button, or star rating in placeholder
- Placeholder that looks like a tappable ad
- Shimmer continuing after ad has loaded
- Clickable placeholder

**Implementation pattern:**
```kotlin
Box(modifier = modifier.height(350.dp)) {
    if (adState is Loading) {
        // Gray boxes only — no fake ad content
        ShimmerPlaceholder()
    }
    AnimatedVisibility(
        visible = adState is Ready,
        enter = fadeIn(tween(300))
    ) {
        NativeAdView(ad = (adState as Ready).ad)
    }
}
```

**Why it matters:** Skeleton placeholders are standard UX loading patterns. The policy risk only arises if the placeholder impersonates a real ad. Keep it skeletal and non-interactive.

---

## 3. Ad Layout — Avoid Accidental Clicks

**Rule:** Ad placement must not encourage accidental clicks or interfere with app functionality.

**Prohibited layouts:**
- Ad placed immediately next to buttons, navigation, or interactive elements
- Ad overlapping app content or partially off-screen
- Ad in a scrollable container where it can be unintentionally tapped
- Floating/sticky ads that follow user scroll without clear boundaries

**Required:**
- Minimum padding (8dp+) between ad and interactive elements
- Clear visual separation (border, background, spacing)
- Full ad view must be visible; no partial rendering

**KMP-specific:** When using `AndroidView` or `UIKitView` for native ads, ensure the modifier bounds match the actual platform view bounds exactly.

---

## 4. Native Ad Labeling Requirements

**Rule:** Native ads must clearly distinguish ad content from app content.

**Required elements:**
- AdChoices overlay must be visible and tappable (rendered by SDK automatically when using `NativeAdView`)
- If using custom native ad rendering, manually add "Ad" or "Sponsored" label near the ad content
- Do NOT remove or obscure the AdChoices icon

**KMP implementation:**
- Android: `NativeAdView` handles AdChoices automatically if ad is registered properly
- iOS: `GoogleMobileAds.NativeAdView` includes AdChoices; do not hide or reposition it

---

## 5. Interstitial Ad Timing

**Rule:** Interstitials must appear at natural transition points, never unexpectedly.

**Prohibited:**
- Showing interstitial on app launch (use App Open Ad instead)
- Showing interstitial during active gameplay or user input
- Showing multiple interstitials back-to-back without user action between
- Auto-dismiss or skip interstitials programmatically

**Required:**
- Show at level completion, screen transition, or natural pause
- Implement frequency capping (e.g., max 1 per 3 minutes)
- Load in advance; only show when user navigates to new screen

---

## 6. Rewarded Ad Integrity

**Rule:** Rewarded ads must grant the promised reward reliably and immediately upon completion.

**Prohibited:**
- Not delivering reward after user watches full ad
- Granting reward before ad completes (or without ad interaction)
- Fake/placeholder rewards
- More than one reward per completed ad view

**Required:**
- Grant reward ONLY in the `onRewarded` / `adDidDismiss` callback
- Reward must match what was promised in the call-to-action
- Handle edge case: user force-quits app mid-ad — persist reward state

---

## 7. Test Ad Unit Enforcement

**Rule:** Production ad unit IDs must NEVER be used during development or testing.

**Required:**
- Use official Google test ad unit IDs in debug builds
- Gate production IDs behind `BuildConfig.DEBUG` / `#if DEBUG` checks
- Never commit production ad unit IDs to version control

**Test IDs:** See `references/test-ad-unit-ids.md`

---

## 8. Content Restrictions Near Ads

**Rule:** Ads must not appear alongside prohibited or restricted content.

**Prohibited adjacency:**
- Adult content, gambling, or violence
- User-generated content without moderation
- Content that infringes copyright
- Misleading or deceptive content

**Policy implication for KMP apps:** If your app displays user-generated financial data or news, ensure it is moderated or clearly separated from ad placements.

---

## 9. SDK Policy Compliance

**Rule:** Only official Google Mobile Ads SDK methods may be used for ad loading, display, and tracking.

**Prohibited:**
- Simulating impressions or clicks programmatically
- Wrapping ad clicks to add custom tracking (interferes with SDK click handling)
- Using reflection to modify SDK behavior
- Implementing "click-for-reward" systems on non-rewarded ads

**Required:**
- Load ads via official `AdLoader`, `InterstitialAd.load()`, etc.
- Let SDK handle impression and click tracking natively
- Do not interfere with SDK lifecycle callbacks

---

## 10. Consent Requirements (GDPR / ATT)

**Rule:** Consent must be obtained before personalized ads are shown.

**Required:**
- Initialize UMP SDK before Mobile Ads SDK
- Honor `canRequestAds` flag — do NOT load ads if false
- On iOS, request ATT permission before loading ads (required for personalized ads)
- Provide privacy options entry point if user is in EEA

**See:** `references/consent-and-ump.md` for full implementation.

---

## 11. Native Ad Lifecycle — Impression Validity

**Rule:** Impressions only count when the ad is fully loaded, visible, and properly measured.

**KMP-specific risk:**
- Animating ad visibility before ad load completes can cause invalid impressions
- Platform view interop (`AndroidView` / `UIKitView`) requires the view to be measured before SDK registers impression
- If ad container is 0x0 during initial frame, impression may be lost

**Mitigation:**
- Load ad completely before making container visible
- Ensure modifier provides concrete dimensions (no `wrap_content` equivalent ambiguity)
- On iOS, `UIKitView` must have `isInteractive = true` for proper click tracking

---

## Quick Compliance Checklist

| Checkpoint | Status |
|-----------|--------|
| No scale/bounce/slide animations on ads | [ ] |
| Fade-in only, and only after fully loaded | [ ] |
| Shimmer placeholder uses gray blocks only (no fake ad content) | [ ] |
| Placeholder non-interactive and stops when ad loads | [ ] |
| AdChoices visible on all native ads | [ ] |
| 8dp+ padding from interactive elements | [ ] |
| Interstitials at natural transitions only | [ ] |
| Rewarded ads grant reward in callback only | [ ] |
| Test ad units in debug builds | [ ] |
| UMP consent before ad load | [ ] |
| Production IDs not in source code | [ ] |
| `isInteractive = true` on iOS UIKitView | [ ] |

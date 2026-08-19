# LULT — Local Ugandan Languages Translator

Built against your two reference documents: the original 7,195-line
architecture doc, and this latest 2,364-line formal PRD (26 numbered
sections — Executive Summary through Compatibility Strategy). This README
maps what's actually implemented against the PRD's own section numbers.

---

## What's new in this round (PRD-driven)

### Per-language API keys (your explicit request)
`TranslationKeyStore.kt` stores a key per language — Runyankole, Luganda,
Rutooro each get their own field in Settings, plus a "Default" fallback
used for any language without its own key. **Worth knowing:** Sunbird's
real API uses one key across every language it supports, so entering the
same key in all four fields is normal and correct. The per-language
structure matches PRD Section 12.2's "Secondary/future provider if
configured" — it's ready for the day a language routes through a
different provider than Sunbird, which is a live possibility for Rutooro
specifically (see the in-app warning — it's still not on Sunbird's
documented endpoint list).

### Provider abstraction (Section 12.1)
`TranslationProvider` interface + `SunbirdTranslationProvider`
implementation. PRD Section 7.2 is explicit that this build is "tightly
coupled to a single translation provider: Sunbird" — the interface exists
so a second provider is a new class, not a rewrite of the service.

### Content classification & privacy firewall (Section 11)
`ContentClassifier.kt` — every piece of on-screen text is classified as
UI / System / User Content / Sensitive / Authentication / Unknown using
Android field metadata (password, editable) plus text-pattern heuristics
(OTP-shaped digit sequences near a keyword, card-number-shaped sequences,
authentication keywords). Enforcement:
- **AUTHENTICATION** content is never processed at all — not translated, not shown in the overlay.
- **SENSITIVE** content is never sent to the cloud, in any privacy mode — only phrasebook/cache, which are local. If neither has it, the original text is shown, untranslated, rather than risking a cloud call.
- Everything else respects the user's **Privacy Mode** (Maximum Privacy / Balanced / Online Translation — Section 11.3), chosen in Settings.

**Honest caveat:** this is a first-pass heuristic classifier, not a
guarantee. It will have both false negatives (something sensitive that
slips through as "Unknown") and false positives (ordinary text that gets
over-cautiously withheld). PRD Section 11.2 calls this out too —
"detection is advisory unless a policy rule explicitly blocks
processing." Tune the patterns against real screens on your A53 before
trusting this as a hard privacy boundary.

### Service state machine (Section 8)
`ServiceState.kt` implements the PRD's exact 9 states (UNINITIALIZED →
PERMISSIONS_REQUIRED → READY → ACTIVE/PAUSED/OFFLINE/DEGRADED/RECOVERING
→ STOPPED). `ServiceStateHolder` is a simple in-memory holder the
service updates as it runs; not yet wired into a visible dashboard
indicator beyond the existing status text.

### Safe overlay rendering, simplified (Section 16)
`OverlayManager.showBatchOverlay()` now implements 3 of the PRD's 5
rendering modes: **Replace** (fits in original bounds), **Compact**
(smaller font, same bounds), and **Bubble** (a small "ⓘ" indicator when
even a compact font won't fit — better than letting translated text
overflow into neighboring UI). It also skips placing a label that would
collide with one already placed this pass (Section 16.2: "avoid
overlaying one translated label on another").

**Not implemented:** *Adjacent* and *Expand* modes (shifting into empty
space beside/below the source). Doing those correctly needs measuring
real empty space on an actual screen — reasonable to add once you're
looking at real devices rather than guessing at margins that might not
hold up.

---

## Full section-by-section status against the new PRD

| PRD Section | Status |
|---|---|
| 6. POC Gate | ⬜ Not run — worth doing before more feature work, see "Suggested next step" |
| 7. System Architecture / 7.3 pipeline / 7.4 UI diffing | ✅ Implemented |
| 8. Service State Machine | ✅ Implemented |
| 9. Permissions / Permission UX | 🟡 Core permissions implemented; revocation detection and guided recovery path not built |
| 10. Accessibility Service & UI Capture | ✅ Implemented (timeout-based, no node cap) |
| 11. Content Classification & Privacy Firewall | ✅ Implemented (heuristic — see caveat above) |
| 12. Translation Architecture (provider abstraction, batching, retry) | ✅ Implemented |
| 13. Terminology Engine | ⬜ Deferred — needs real governed terminology data, not fabricated |
| 15. Offline Translation Architecture | 🟡 Level 1 (phrasebook) mechanism built, needs native-speaker-verified content; Levels 2/3 deferred, no model file exists |
| 16. Safe Overlay Rendering Engine | 🟡 3 of 5 modes implemented (Replace/Compact/Bubble) |
| 17. Translation Memory and Cache | 🟡 Session cache implemented; user-approved TM and structured cache keys (17.3) not built |
| 18. App Exclusion and App Policy | 🟡 Implemented, simplified UI (text list, not a picker) |
| 19. Accessibility of LULT Itself | ⬜ Deferred |
| 20. UX Spec (7-step onboarding, dashboard, notifications) | ⬜ Deferred — only a minimal onboarding + settings screen exist |
| 21. Future Innovations (confidence, community correction, prediction) | ⬜ Explicitly future-scoped by the PRD itself |
| 22. Backend and Translation Gateway | ⬜ Deferred — see security note below, this matters |
| 23. Data Architecture | 🟡 Prefs/EncryptedSharedPreferences cover "Low" privacy-class entities; no local database (Room) yet |
| 24. Security Architecture | 🟡 Most rules followed; **Section 24.2's "do not expose provider credentials in client code" is NOT satisfied** — see below |
| 25. Performance/Battery Budgets | 🟡 Auto-pause implemented; no real device measurement done yet |
| 26. Device Compatibility Strategy | ⬜ Not tested against device tiers yet |

---

## The one security tension worth reading carefully

PRD Section 24.2 says plainly: **"do not expose provider credentials in
client code."** Section 22 recommends the fix — a project-controlled
backend gateway that holds the real Sunbird key server-side, so the app
never has it at all.

This build does not have that gateway. `TranslationKeyStore` encrypts the
key at rest on the device, which is a reasonable *local-testing*
posture, but it is explicitly **not** what the PRD's own security section
calls for. A determined attacker with your unlocked phone, or a rooted
one, can still get the key out. This is fine for you testing on your own
A53. It stops being fine the moment this goes to other people's phones —
building the Section 22 gateway (even a minimal one: one endpoint that
takes text + language, holds the real Sunbird key itself, returns a
translation) should happen before any real distribution.

---

## Deferred, and why — being specific rather than vague

- **Terminology Engine (13) / user-approved Translation Memory (17.2-17.3):** both need real curated data (approved translations, reviewed by someone) — building the mechanism without real content would just be empty scaffolding.
- **Language-pack security (15.3 — signing, integrity, rollback):** irrelevant until Level 2/3 offline models exist to secure in the first place.
- **Backend/Translation Gateway (22):** a real, separate service — see security section above for why this matters more than it might seem.
- **Full UX spec (20):** 7-step onboarding, dashboard, notification integration — real scope, not built past a minimal 2-screen flow (MainActivity + SettingsActivity).
- **POC Gate (6):** the PRD describes this as a formal go/no-go checkpoint before deeper investment — worth actually running (test the core read→translate→overlay loop on your A53 against 3-5 real apps) before sinking more time into sections like the Terminology Engine or Backend Gateway.

---

## Building it via GitHub Actions (works today — Firebase Studio no longer accepts new signups)

A ready-to-go workflow is already included: `.github/workflows/build.yml`.

1. Create a new repo: **https://github.com/new**
2. Push this project to it (or use GitHub's "upload files" web UI to drag in the `PhoneTranslator` folder if you're not comfortable with git commands yet).
3. Go to the repo's **Actions** tab — the "Build Debug APK" workflow runs automatically on every push to `main`, or click **"Run workflow"** to trigger it manually without pushing anything.
4. When the run finishes (a few minutes), open it and scroll to **Artifacts** at the bottom — `PhoneTranslator-debug-apk` is a zip containing the built `.apk`. Download it, unzip, and you have a real installable APK.
5. Transfer that `.apk` to your Samsung A53 (email it to yourself, use a USB cable, or Google Drive) and install it — you'll need to allow "install from unknown sources" for whichever app you use to open it, since it's not from the Play Store.
6. Open the app → Settings → paste your Sunbird key into the Default field → go back → turn the translator on.

This gives you a real APK without installing Android Studio locally or needing Firebase Studio. The tradeoff: no live device connection or in-browser IDE like Firebase Studio offered — you're building headless and sideloading the result. If you want the interactive editing experience back, Google's suggested Firebase Studio replacements are Google AI Studio or Google Antigravity; worth a look if GitHub Actions feels too code-first for how you want to keep iterating.

## Building it via Android Studio (alternative)

## File map (current)

```
PhoneTranslator/
├── app/src/main/java/com/accessibility/phonetranslator/
│   ├── MainActivity.kt                     — onboarding + language picker
│   ├── SettingsActivity.kt                 — per-language keys, privacy mode, exclusions
│   ├── TranslationAccessibilityService.kt  — reads screen, classification + diff-based pipeline
│   ├── OverlayManager.kt                   — bubble + safe-layout overlay (Replace/Compact/Bubble)
│   ├── TranslationProvider.kt              — provider interface (spec 12.1)
│   ├── SunbirdTranslationProvider.kt       — Sunbird implementation of the above
│   ├── ContentClassifier.kt                — content classes + privacy-mode enforcement (spec 11)
│   ├── RateLimiter.kt                      — sliding-window limiter + exponential backoff
│   ├── OfflinePhrasebook.kt                — Level 1 offline phrasebook loader
│   ├── UISnapshot.kt                       — screen capture data model + diffing
│   ├── TranslationCache.kt                 — in-memory LRU session cache
│   ├── TranslationKeyStore.kt              — encrypted per-language API keys
│   ├── ServiceState.kt                     — service state machine (spec Section 8)
│   ├── Prefs.kt                            — language, on/off, exclusions, privacy mode, idle tracking
│   └── SupportedLanguage.kt                — language list + API-confirmation flags
├── app/src/main/assets/phrasebook.json     — Level 1 phrasebook (needs native-speaker fill-in)
├── app/src/main/res/
│   ├── layout/ (activity_main, activity_settings, overlay_bubble)
│   ├── values/ (strings.xml, styles.xml)
│   └── xml/accessibility_service_config.xml
└── AndroidManifest.xml
```

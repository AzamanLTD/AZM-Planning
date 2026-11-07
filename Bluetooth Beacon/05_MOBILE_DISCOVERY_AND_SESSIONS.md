# 05 — Mobile discovery, preferences and sessions

## Platform design

Flutter owns shared state/UI. Background BLE detection must use native platform mechanisms and be tested on physical devices; a Dart stream alone is insufficient.

### Android
Evaluate filtered BLE scanning. Android documentation describes BluetoothLeScanner.startScan with PendingIntent for matching results when the process isn't running. Background limits and Bluetooth permissions apply; periodic scans are not a recommended substitute.

- Background BLE: https://developer.android.com/develop/connectivity/bluetooth/ble/background
- Bluetooth permissions: https://developer.android.com/develop/connectivity/bluetooth/bt-permissions

Review target SDK and location-derived permission requirements; test OEM battery restrictions.

### iOS
For iBeacon, evaluate Core Location region monitoring/ranging. Background delivery depends on authorisation, app config, Bluetooth/location services and OS. Do not promise delivery after force-quit or disabled permission/services.

- Apple: https://developer.apple.com/documentation/corelocation/determining-the-proximity-to-an-ibeacon-device

### Flutter/native boundary
Native iOS uses Core Location; native Android uses filtered scan/PendingIntent; Flutter handles preferences, authenticated API, dedupe, approved routes, in-app card and notification tap. Evaluate plugin maintenance and background support. Keep permissions/background modes explicit/testable.

## App-state behaviour

| State | Behaviour |
|---|---|
| Foreground | In-app card/sheet after backend resolution/preferences |
| Background with OS event | Optional system notification if permission/policy allow |
| Notification denied | No fake notification; foreground still available |
| Bluetooth/location unavailable | Explain; manual marketplace discovery |
| Network unavailable | Honest retry/search; no stale transaction screen |
| OS suppresses/force-stop | Event may not arrive; no guarantee or false telemetry |

Do not overlay modal above another app or lock screen. Notification is the background prompt; user opens Azaman.

## Presentation preferences

Effective policy combines merchant config, user's vertical preferences, notification setting, personalisation consent, app state, OS permission, cooldown, dismissal and ambiguity.

Suggested modes:
- OFF: no prompt.
- IN_APP_ONLY: foreground card only.
- IN_APP_AND_NOTIFICATION: foreground card when active; notification when OS delivers background event.

Avoid duplicate presentation. User setting can suppress merchant-requested notification, never override it.

## Onboarding/settings

Ask which verticals matter: retail, restaurants, hotels, fuel and transit. Make optional and explain. Settings: discovery per vertical, system notifications, personalised ranking, marketing consent/withdrawal, pause, permission state/OS settings, privacy and data controls. Category interest is not blanket tracking/marketing consent.

## Repeat visits and personalisation

Do not increment visits per range callback. A qualified session requires active device/assignment, stable observations, dedupe, bounded open criteria, absence/hysteresis close, and meaningful separation/cooldown before new visit.

Evaluate “two distinct visits” with pilot data. Commuter may pass daily with no interest; first-time user who searched menu may be relevant. Use search interest mainly to rank in-app suggestions, not trigger intrusive marketing.

## Search/recommendation policy

Use matching queries only with personalisation enabled and within retention. Prefer explicit interest/current context; category/tag similarity and recency; minimise raw query retention; suppress after dismissals; no marketing notification solely from a search match; keep marketing consent separate; make ranking explainable.

## Cooldowns

Define same-event dedupe, per-branch notification cooldown, in-app dismissal cooldown, global frequency budget, re-entry criteria and distinction from transactional notifications. Server suppression authoritative across devices/reinstall; client dedupe is UX aid only.

## Offline

No offline checkout. If backend resolution fails, do not load stale assignment/capability; do not later treat delayed observation as current visit; no queued payment/pump selection/success; offer marketplace search and retry. A diagnostic queue requires explicit purpose, expiry, integrity and privacy design.

## Test matrix

Android/iOS versions and OEMs; foreground/background/locked/terminated/force-quit; install and permission denied/revoked/restored; Bluetooth toggle; notification allowed/denied; battery saver; online/offline/flapping; single/adjacent/multiple beacon; RSSI fluctuation and disappearance; dedupe/cooldown/dismissal; notification deep link; stale/untrusted route refusal.

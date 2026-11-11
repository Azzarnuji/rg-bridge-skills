---
name: rg-bridge
description: Use when integrating or debugging the rg_bridge Flutter SDK (RGBridge) — device info, 16 hardware tests (touchscreen, sensors, camera, mic, GPS, WiFi, Bluetooth, biometric, buttons), the WebView JS bridge, form submission, desk inbox, QR take-photo file transfer, and scan-diagnostic transaction submission/accept. Covers initialization, Indonesian/English localization (RgBridgeLocale), one-screen-per-test structure, the mandatory camera/touchscreen library widgets, controller lifecycle, result streams, REQUIRES_USER_ACTION, permissions, and troubleshooting.
---

# RGBridge Flutter SDK

`rg_bridge` is a Flutter package that exposes public surfaces on one
`RGBridge` instance:

| Surface | Access | Needs `baseUrl` | Purpose |
|---|---|---|---|
| Device | `sdk.device` | No | Device info + 16 hardware tests |
| WebView | `sdk.webview` | Yes | Session, controller, JS ↔ Dart events |
| Form | `sdk.formSubmission` | Yes | Init session + submit form |
| Desk | `sdk.desk` | Yes | Customer inbox URL |
| QR | `sdk.qr` | Yes | Take-photo QR + file transfer |
| Transaction | `sdk.transaction` | Yes | Submit scan result + accept request |

## When to use this skill

Use for any task that:

- adds `rg_bridge` to a Flutter app, or initializes `RGBridge(...)`;
- runs a hardware capability test and needs result handling;
- localizes SDK text to Indonesian/English or switches language at runtime;
- builds a WebView flow backed by an rg_bridge session;
- submits an rg_bridge form, creates a desk inbox, or starts a take-photo QR;
- submits a scan-diagnostic result and accepts the inspection request;
- debugs `NOT_SUPPORTED`, timeouts, missing events, or dispose issues.

Do not use for unrelated Flutter plugin work.

## Non-negotiable rules

1. Import only the public barrel: `package:rg_bridge/rg_bridge.dart`.
   Never import `package:rg_bridge/src/...` from app code. Camera and
   touchscreen widgets are intentionally obtained from
   `sdk.device.buildCameraWidget(...)` and
   `sdk.device.buildTouchscreenWidget(...)`.
2. Create **one** `RGBridge` instance for the app. `RGBridge` is a singleton;
   the first construction fixes `baseUrl`, `clientId`, and locale.
3. The library automatically tracks every result emitted by controllers created
   through `sdk.device` and stores it in the device session. Host apps may still
   subscribe to `controller.getResultStream` for UI, but must not manually save
   results.
4. `REQUIRES_USER_ACTION` is intermediate. Only act on
   `result.status.isTerminal`.
5. Always `checkDispose()` device controllers in `dispose()`. Call
   `sdk.webview.offAllEvents()` when leaving a WebView flow.
6. Only load trusted URLs/HTML in the WebView. Never log session tokens, form
   values, or PII.
7. Never hardcode real `clientId` values in committed code or docs; use
   placeholders / `String.fromEnvironment`.
8. The legacy README shows outdated WebView signatures. Trust this skill and
   the source (`lib/src/addons/...`) over old docs.
9. Every hardware test runs in its **own host-built screen**
   (`Check<Test>Screen`). The screen owns the UI and controller lifecycle. The
   library owns result persistence. The only exception to host-built screens is
   `camera` and `touchscreen`: use `sdk.device.buildCameraWidget(...)` /
   `sdk.device.buildTouchscreenWidget(...)`, which automatically register their
   final result in the library session.
10. Never construct `CheckCameraWidget` or `CheckTouchscreenWidget` directly.
   Their constructors are not part of the public barrel because direct
   construction bypasses automatic device-session result tracking.
11. All device controller types are exported by the barrel, so typed controller
    variables may be used directly after importing
    `package:rg_bridge/rg_bridge.dart`. Never import controller files from
    `package:rg_bridge/src/...`.
12. iOS setup is host-only: put **all seven** usage descriptions in the host
    app `ios/Runner/Info.plist` — `NSCameraUsageDescription`,
    `NSMicrophoneUsageDescription`, `NSSpeechRecognitionUsageDescription`,
    `NSFaceIDUsageDescription`, `NSLocationWhenInUseUsageDescription`,
    `NSMotionUsageDescription`, `NSBluetoothAlwaysUsageDescription`. iOS does
    not merge package plists; a missing motion/Bluetooth key crashes the app.
    On Android, `MainActivity` must extend `FlutterFragmentActivity`.
13. QR take-photo device labels must be read from the library:
   `await sdk.device.getDeviceDisplayInfo()` then pass `deviceName` /
   `deviceDetail` to `sdk.qr.photo.show(...)`. Never hardcode a device name or
   detail (no `'Apple iPhone 13'`, no `'128 GB • Purple'`) and never invent the
   photo count — `sessionCode`, `photoSteps`, and `maxFiles` come from the
   server response.
14. Treat `result.status` / `status.wire` as the machine contract. Localization
   changes only user-facing descriptive text. Status tokens (`PASS`, `FAILED`,
   `SKIPPED`, `NOT_SUPPORTED`, `REQUIRES_USER_ACTION`) and any status prefix in
   `message` are **never translated**; `testId` and `action.type` stay
   untranslated too. Never parse the localized `message` for app logic — use
   `status`.
15. Use only `sdk.transaction.*` for scan-diagnostic transactions. The
    transaction `httpClient` is private by design; never expose or try to access
    `sdk.transaction.deviceSubmission.httpClient`, and never import
    `package:rg_bridge/src/addons/transaction/...`.

## Install and initialize

`rg_bridge` is distributed from the private GitLab pub registry, not pub.dev.
Get the registry URL and secret token from the maintainer.

First, store the token for the registry (paste the secret token when prompted):

```bash
dart pub token add <PRIVATE_PUB_HOSTED_URL>
# Paste the secret token when prompted
```

Then add the dependency:

```yaml
dependencies:
  rg_bridge:
    hosted: <PRIVATE_PUB_HOSTED_URL>
    version: ^0.1.5
```

```bash
flutter pub get
```

```dart
final sdk = RGBridge(
  baseUrl: 'https://api.example.com',
  clientId: 'rg_your_client_id',
);
```

Device-only code may omit `baseUrl`. WebView, Form, Desk, QR, and Transaction
require a valid `baseUrl` + `clientId`.

## Localization (Indonesian / English)

SDK text is Indonesian by default. English is opt-in at construction:

```dart
final sdk = RGBridge(
  baseUrl: 'https://api.example.com',
  clientId: 'rg_your_client_id',
  locale: RgBridgeLocale.en, // default: RgBridgeLocale.id
);
```

Switch at runtime with the exported i18n API:

```dart
RgBridgeI18n.configure(locale: RgBridgeLocale.id);
```

Coverage: device result/action messages, camera/touchscreen/QR widget labels,
the biometric prompt, and the STT recognition locale (`id_ID` / `en_US`).
Locale is **global** — `RGBridge(locale:)` calls `RgBridgeI18n.configure(...)`,
so the last constructed `RGBridge` wins. Do not create multiple SDK instances
with different locales.

Localization does not change the wire contract:

- `result.status` stays `passed` / `failed` / `skipped` / `notSupported` /
  `requiresUserAction`; `status.wire` stays `PASS` / `FAILED` / `SKIPPED` /
  `NOT_SUPPORTED` / `REQUIRES_USER_ACTION`.
- When a `message` carries a status prefix (`PASS - ...`), that prefix remains
  the same English token in both locales.
- `testId`, `action.type`, and JSON keys are never translated.

Never derive state by parsing `message`; read `status`, `testId`, and
`action.type`.

## Device-test screens (required)

The host may subscribe for presentation, but result persistence is automatic:

```dart
final camera = sdk.device.checkCamera();
camera.getResultStream.listen((result) {
  // Optional UI update only. Do not save the result manually.
});
camera.checkStart();
```

After the test flow, build the backend-ready payload without performing a
network request:

```dart
final payload = await sdk.device.buildSummary();
// payload contains qrValue: null, deviceInfo, tests, installedApps: [].
await dio.post(endpoint, data: payload);
```

`buildSummary()` defaults to `resetSession: true`, so a successful summary
build clears the tracked test state for the next inspection. Use
`buildSummary(resetSession: false)` when the payload is only being previewed.

The library also exposes `getTestResult(testId)` and
`getAllTestResults()` for UI/debug inspection.

Every test gets its own screen. The host builds the screen (route, layout,
buttons); the screen owns the controller lifecycle.

```dart
// import 'dart:async';
// import 'package:rg_bridge/rg_bridge.dart';
class CheckWifiScreen extends StatefulWidget {
  const CheckWifiScreen({super.key, required this.sdk});
  final RGBridge sdk;

  @override
  State<CheckWifiScreen> createState() => _CheckWifiScreenState();
}

class _CheckWifiScreenState extends State<CheckWifiScreen> {
  late final _controller = widget.sdk.device.checkWifi();
  StreamSubscription<DeviceTestResult>? _sub;
  DeviceTestResult? _result;

  @override
  void initState() {
    super.initState();
    // Optional UI subscription; the SDK tracks the result automatically.
    _sub = _controller.getResultStream.listen((result) {
      if (!mounted) return;
      if (!result.status.isTerminal) {
        // REQUIRES_USER_ACTION — show instructions, keep waiting.
        setState(() => _result = result);
        return;
      }
      setState(() => _result = result);
    });
    _controller.checkStart(); // fire-and-forget; never await a result.
  }

  @override
  void dispose() {
    _sub?.cancel();
    _controller.checkDispose();
    super.dispose();
  }
  // build(): host UI + notifiers, e.g. checkRequestEnable/checkNotifyScan.
}
```

### Library widgets (mandatory for camera & touchscreen)

| Widget | Test | Rule |
|---|---|---|
| `sdk.device.buildCameraWidget()` | `camera` | **Mandatory.** Host pushes it as a route. |
| `sdk.device.buildTouchscreenWidget()` | `touchscreen` | **Mandatory.** Host pushes it as a route. |
| `TakePhotoQrWidget` | QR photo | QR surface, not a hardware test. |

All other tests are **host-built screens**: the host writes the UI and calls the
controller notifiers (`checkVerifyGuess`, `checkNotify*`, ...).
`navigation_button` is host-built as well (`PopScope` +
`WidgetsBindingObserver`); the library ships no navigation screen.

```dart
// camera — factory required; do not construct the widget directly
final result = await Navigator.of(context).push<DeviceTestResult>(
  MaterialPageRoute(
    builder: (_) => sdk.device.buildCameraWidget(
      checkIsFront: false,
      onCheckFinished: (r) => Navigator.of(context).pop(r),
    ),
  ),
);

// touchscreen — factory required; do not construct the widget directly
final result = await Navigator.of(context).push<DeviceTestResult>(
  MaterialPageRoute(
    builder: (_) => sdk.device.buildTouchscreenWidget(
      columns: 15,
      rows: 26,
      timeout: const Duration(seconds: 60),
      onCheckFinished: (r) => Navigator.of(context).pop(r),
    ),
  ),
);
```

The per-test screen table lives in `references/device-tests.md`.

Status wire values: `PASS`, `FAILED`, `SKIPPED`, `NOT_SUPPORTED`,
`REQUIRES_USER_ACTION`. `DeviceTestStatus.isTerminal` excludes the last one.
The wire values and any status prefix in `message` are not localized; only the
descriptive text is. Use `status`, not `message`, for branching.

Test ids: `touchscreen`, `accelerometer`, `gyroscope`, `biometric`,
`bluetooth`, `wifi`, `gps`, `camera`, `speaker`, `vibration`, `speech`,
`microphone`, `flashlight`, `navigation_button`, `power`, `volume_button`.

## Canonical WebView recipe

```dart
final session = await sdk.webview.initSessionWebView(
  type: 'pengecekan-laptop',
  firstName: 'John',
  lastName: 'Doe',
  memberCode: '123456',
  meta: {'conversation_id': 'conv-123'},
);

for (final type in WebviewEventType.values) {
  sdk.webview.onEvent(type, (event) {
    print('${event.type.name}: ${event.data}');
  });
}

final controller = sdk.webview.createController(
  onConsoleMessage: (m) => print('[JS] ${m.message}'),
);
await sdk.webview.openWebView(
  controller: controller,
  url: session.webViewUrl,
);
// host renders WebViewWidget(controller: controller)
```

Flutter → JS:

```dart
await sdk.webview.sendToWebView(
  controller: controller,
  event: const WebViewEvent(
    type: WebviewEventType.formSuccess,
    data: {'id': 'abc'},
  ),
);
```

`createController` takes no session. `loadHtmlString` and
`loadHtmlStringWithInlineJs` require a `controller`. Events:
`formSubmitted`, `formCancelled`, `formError`, `formSuccess`, `formClosed`,
`imageCaptured`.

## Form, Desk, QR in one glance

```dart
// Form
final session = await sdk.formSubmission.initFormSubmission();
final result = await sdk.formSubmission.submitForm(
  sessionToken: session.sessionToken,
  form: SubmitFormModel(/* ... */),
);

// Desk
final inbox = await sdk.desk.createCustomerInbox(
  name: 'Ahmad', phone: '08...', email: 'a@b.c',
  memberCode: '0001', ttlHours: 24,
);

// QR take-photo — device label from the library, never hardcoded
final display = await sdk.device.getDeviceDisplayInfo();
final info = await sdk.qr.photo.show(
  deviceName: display.deviceName,
  deviceDetail: display.deviceDetail,
);
sdk.qr.photo.onFileReceived.listen((file) { /* ReceivedFile */ });
// info.sessionCode, info.photoSteps, info.maxFiles (from server)
// widget: sdk.qr.photo.buildQr(size: 220)
// or: TakePhotoQrWidget(url: info.url, size: 220)
```

## Transaction in one glance

```dart
// Submit the scan result (payload from device.buildSummary())
final submit = await sdk.transaction.deviceSubmission.execute(
  await sdk.device.buildSummary(),
  args: DeviceSubmissionArgs(includePricing: true),
);

// Keep submit.requestCode, then mark the request completed
final accept = await sdk.transaction.deviceSubmissionAccept.execute(
  args: DeviceSubmissionAcceptArgs(requestCode: submit.requestCode),
);
// accept.status -> COMPLETED
```

`DeviceSubmissionModel` exposes `result`, `message`, `requestId`,
`requestCode`, `pricing` (`priceMin`, `priceMax`, `message`), and `hdc`
(`value`, `reason`, `chipsetApi`, `chipsetDevice`). `hdc.value == true` means
the device **is** an HDC device; `false` means it is **not**. Parsing accepts
snake_case/camelCase, numeric strings, and missing pricing; `hdc` is
defensive too (missing object/fields fall back to
`DeviceSubmissionHdcModel.empty` / defaults). Response models also expose
snake_case `toJson()`
so `fromJson(model.toJson())` round-trips. Catch `HttpValidationException`
(HTTP 422) separately from network/non-2xx errors.

## Reference routing

Read the matching file before writing non-trivial code:

| Topic | File |
|---|---|
| Requirements, install, init, lifecycle, platform setup | `references/quickstart.md` |
| All `get*` info APIs, the 16 `check*` tests, the required screen-per-test table, and the mandatory camera/touchscreen widgets | `references/device-tests.md` |
| WebView API, events, channels, HTML loading, permissions | `references/webview.md` |
| Form model contract, desk inbox, QR streams and teardown | `references/form-desk-qr.md` |
| Transaction submit + accept, models, errors, security | `references/transaction.md` |
| Errors, emulator/iOS limitations, debug checklist | `references/troubleshooting.md` |

## Verification

After editing a Flutter app:

```bash
flutter pub get
flutter analyze
flutter test        # if tests exist
```

Hardware behavior must be verified on a physical Android or iOS device.
iOS volume/power use public APIs but still need a real iPhone; the Simulator
cannot change volume through `MPVolumeView`. Emulators and desktop targets
need the simulation notifiers documented in `references/device-tests.md`.

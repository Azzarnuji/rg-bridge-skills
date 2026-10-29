# Quickstart

## Requirements

| Item | Value |
|---|---|
| Dart SDK | `^3.12.2` |
| Flutter | `>=1.17.0` (use current stable) |
| Android | Physical device strongly recommended |
| iOS | Some native channels are stubs → `NOT_SUPPORTED` |

## Dependency

```yaml
dependencies:
  rg_bridge: ^0.1.1
```

Internal Git source:

```yaml
dependencies:
  rg_bridge:
    git:
      url: https://git.stechindonesia.co.id/raja-gadai/rg_bridge.git
      ref: main
```

```bash
flutter pub get
```

## Import

```dart
import 'package:rg_bridge/rg_bridge.dart';
```

The barrel exports `RGBridge`, enums, models, and `CheckTouchscreenWidget`.
Controller types are used through `sdk.device.*`; they do not need separate
imports. Never import `src/`.

## Initialize

```dart
final sdk = RGBridge(
  baseUrl: 'https://api.example.com',
  clientId: 'rg_your_client_id',
);
```

`RGBridge({String baseUrl = '', required String clientId})`.

Recommended service wrapper:

```dart
class RGBridgeService {
  late final RGBridge sdk;

  RGBridgeService() {
    sdk = RGBridge(
      baseUrl: const String.fromEnvironment('RG_BASE_URL'),
      clientId: const String.fromEnvironment('RG_CLIENT_ID'),
    );
  }
}
```

One instance per app. Do not construct `RGBridge` inside every screen.

## Lifecycle

```text
Create RGBridge
  → use device / webview / form / desk / qr
  → handle result / event
  → cleanup listener + controller
  → screen/flow done
```

| Object | Cleanup |
|---|---|
| Device controller | `checkDispose()` |
| Device subscription | `await sub.cancel()` |
| WebView event | `offEvent(type)` / `offAllEvents()` |
| WebView controller | Follows host `WebViewWidget` lifecycle |
| QR photo | `stop()` ends session; `dispose()` closes streams |
| UI timer | Cancel on terminal result / dispose |

## Platform setup

Android permissions are merged from the package manifest: Bluetooth,
Bluetooth scan/connect, location, WiFi state, camera, record audio, vibrate.
Do not copy them manually.

iOS `ios/Runner/Info.plist`:

```xml
<key>NSCameraUsageDescription</key>
<string>Camera is used for the camera test and photo capture.</string>
<key>NSMicrophoneUsageDescription</key>
<string>Microphone is used for microphone and speech tests.</string>
<key>NSSpeechRecognitionUsageDescription</key>
<string>Speech recognition is used for the microphone test.</string>
<key>NSFaceIDUsageDescription</key>
<string>Face ID is used for the biometric test.</string>
```

## Suggested structure

```text
lib/
├── sdk/rgbridge_service.dart
├── features/device_check/
├── features/webview/
├── features/form/
└── main.dart
```

## First-run checklist

1. `flutter pub get`
2. Run on a physical Android device.
3. `getDeviceInfo()` returns brand/model.
4. One test stream emits a terminal result.
5. Only then wire WebView/Form with a valid `baseUrl` + `clientId`.
